# One missing error listener killed every pod of two services
*Why the same dropped socket crashed the kysely services and left the knex ones standing.*

## Two mornings told the same story
Two mornings, one week apart, the platform had the same network event. In the second morning's four minute window, 79 services lost their RabbitMQ connection and four lost Postgres connections. That part was not ours to fix.

Exactly two services died: an offers service with 13 pods and a user service with 6. Every pod alive in that window exited with code 1 within three minutes of each other.

The thread chased memory first, then a query. Both guesses died on the numbers.

## The memory story collapsed under the first chart
The first hypothesis was out of memory. It sounded tidy until I looked at the limits and the shape of the crash.

The pod memory limit was 700 MiB. Peak per-pod usage across the whole morning was 389 MB, or 53 percent, and it stayed flat through the crash. Node heap used peaked at 98 MB against a 384 MB `max-old-space-size`. That is not an exhausted process.

The exit code killed the idea too. It was 1, not 137 for `OOMKilled`, and not 134 for a V8 heap abort. The loud chart showing 752 of 935 MiB was the wrong chart. That was an aggregate across pods, not one process.

## The slow query story failed the control group
The second hypothesis was the highest-traffic query overfetching. It also failed fast.

Every pod died together. Other services saw the same Postgres error in the same window. A slow query in one method cannot do that.

That method only showed up on the log line because it is the highest-traffic DB method, so it owns most pooled connections. It looked guilty because it was busy, not because it was wrong.

## The crash path was five lines of library code
The failure is boring once followed.

1. A connection sits idle in the pool. `pg-pool` attaches its idle listener as the client error handler.
2. The socket dies. `pg`'s `Client` raises `Connection terminated unexpectedly` and emits `error`.
3. The idle listener fires and calls `pool.emit('error', err, client)`.
4. `Pool` extends `EventEmitter`, and nobody listens on `error`, so Node throws it as an uncaught exception.
5. Our shared logger catches uncaught exceptions, logs `uncaught exception`, and calls `process.exit(1)`.

The shared database client library builds the kysely client on a raw `pg.Pool` with no error listener: `new Pool({...options})` then `new Kysely({ dialect: new PostgresDialect({ pool }) })`. Nothing ever registered `pool.on('error')`.

That was the whole crash.

## The control group stayed up for one reason
Four services saw the same Postgres disconnect. Two of them crashed on the kysely path. Two of them stayed alive on the knex path: a restaurant service and an availability service.

That split matters more than the outage itself.

`knex` never creates a `pg.Pool`. It creates `pg.Client` instances under its own tarn pool and attaches an error handler to each client. No shared `Pool` `EventEmitter`, no unhandled error event, no process exit.

A fifth service also logged `Connection terminated unexpectedly`, but its message body was another service's query text, and it has no database dependency at all. That was a propagated RPC error, not its own pool. Same words, different failure.

## Why keepalive became the leading explanation
The missing handler is a year old. The offers service had been on the kysely path since July of the previous year, so the pool had been unguarded for about twelve months.

The bug was old. The failure was new. Two things explain the timing.

First, Postgres socket drops are rare here. Across the two weeks of retained logs, the offers service logged `Connection terminated unexpectedly` on exactly two days, the two crash days, and those are the only two days a RabbitMQ disconnect hit more than 90 services.

| | AMQP disconnects, event window | AMQP disconnects, whole day | offers service exits, window | offers service exits, whole day |
| --- | --- | --- | --- | --- |
| first morning | 64 | 93 | 24 (17 pods) | 30 |
| second morning | 79 | 91 | 14 (13 pods) | 14 |
| an ordinary day | | 11 to 13 | | 0 |

The window columns cover the three to four minutes of the event; the day columns cover the whole day. Exits are container exits, not distinct pods: some pods came back and died again. Localised blips of 4 to 13 services happen most days and never crash anything.

Second, and this is the inferred part: a library release five weeks before the first crash turned on TCP keepalive on the pool defaults: `keepAlive: true`, `keepAliveInitialDelayMillis: 30000`, and `query_timeout: 300000`, and it removed `reapIntervalMillis: 200`. Both crash dates are after that upgrade.

The reason keepalive matters is simple. Without TCP keepalive, a socket whose peer vanished sits there silently, and the failure shows up on the next query. It lands in that query's promise chain and behaves like a normal request error. With keepalive on, the kernel notices the dead peer and closes the socket while it is idle in the pool. That is the one path that reaches `pool.emit('error')`.

`query_timeout` is not a candidate. It rejects the query and leaves the socket alone.

I cannot prove this from logs: retention stops before the upgrade, so there is no pre-upgrade fleet-wide event to compare against. What supports it is the commit message that added keepalive: start probes sooner, to reduce the idle-drop window behind the cloud NAT gateway. The team knew idle connections were being silently dropped and turned keepalive on to detect it. Detecting it is exactly what makes an idle pooled client emit `error`.

## Rolling back would hide the crash
There is no version with the handler to roll back to. The kysely client factory has five commits in its history and none of them registered `pool.on('error')`.

Rolling back keepalive would probably hide the crash, and it would bring back the silently dead connections keepalive was written to catch. The same commit turned keepalive on for the knex path, and the knex services did not crash. Keepalive alone is harmless. Keepalive plus an unguarded `pg.Pool` kills the process.

## The SBOM made the blast radius look bigger than it was
The software bill of materials says 152 services list kysely. That number is real, and it is also misleading.

It is transitive. The shared library declares kysely as a direct dependency next to knex, so every service on the library inherits it whether it uses it or not. A pure-knex service showed kysely 0.29.4 in its SBOM with no kysely in its own `package.json`.

The real discriminator is the `useKysely: true` opt-in in each service's own `createService` call. An organisation-wide code search returns 10 production services. All 10 were exposed. Nobody calls the kysely factory directly outside the library.

Why did the other 8 survive? The two that died had by far the most query volume in the window, 125,043 and 41,100 `pg.query` spans in three hours against 21,373 for the next one, and the other 8 logged no Postgres drop at all. That is consistent with them having fewer idle sockets for the kernel to reap. It is a plausible explanation, not a demonstrated one.

An SBOM tells you what is installed, not what is used.

## One listener in the shared library stops the process exit
The fix is one line in the shared library.

```ts
pool.on('error', onPoolError)
```

`onPoolError` is an option so the platform can override the default. The default is `console.error` because the factory has no logger in scope. The platform registration overrides it with a structured log tagged with the database name and the target, `master` or `replica`, under the new log name `DATABASE_POOL_ERROR`.

`pg-pool` has already removed and destroyed the client by the time it emits `error`, so the listener only needs to exist. There is nothing to clean up.

The knex path stays untouched. It already attaches its own handler per client and was never affected.

This fixes all 10 services at once, on their next library bump.

What cut this investigation down to one line of code was not the stack trace. It was the control group: the two services that saw the same dropped socket, on the same morning, and stayed up. Find the population that saw the same input and did not fail, and the difference between the two is usually the bug.
