# Green by absence: five ways a pipeline reports success while doing nothing
*A step that cannot fail cannot tell you it is broken.*

Five incidents, four repositories, one shape. A pipeline step looked green because it did not error, not because it proved the intended effect happened. The status came from the absence of failure, so the broken path kept passing.

## Coverage uploads can fail every time and still look green

In one open source Angular library, the Codecov upload step had `continue-on-error: true` and `fail_ci_if_error: false`. Codecov rejected every upload with `Token required because branch is protected`, and the step still reported success on every run.

What looked green was the upload step itself. What was happening was simple: no token, rejected upload, green exit.

It hid from pull request `361` to pull request `370`. That is nine pull requests merged before anyone noticed. The coverage badge just stopped moving.

The fix was to remove `continue-on-error`, set `fail_ci_if_error: true`, and authenticate with a token. The rule written afterwards: never give the upload step either flag.

```yaml
continue-on-error: true
fail_ci_if_error: false
```

```yaml
continue-on-error: false
fail_ci_if_error: true
```

## A skipped upstream job can make publish look like a deliberate no-op

In a TypeScript monorepo using Changesets and npm OIDC, version `0.1.0` was merged to `main` and never reached npm. The publish job showed `skipped`, not `failed`.

The changeset job ran only on `pull_request` events. On the push to `main` it was skipped, and GitHub carried that skipped upstream state into every downstream job with a bare condition, even through jobs that had already run.

The publish job's condition was `needs.x.outputs.should-publish == 'true'`, with nothing about the upstream job's status, so the skip flowed through it.

I compared three workflow runs. One was before the changeset job existed, and publish ran. One had everything skipped. One had the changeset job run and publish skip.

The fix was one line. In this dependency graph the condition needed `!cancelled()` and an explicit `needs.<job>.result == 'success'` check, because the job it depends on is legitimately skipped on push.

```yaml
if: needs.x.outputs.should-publish == 'true'
```

```yaml
if: !cancelled() && needs.x.result == 'success' && needs.x.outputs.should-publish == 'true'
```

## A planner can plan zero and still hide a missed release

In the same monorepo, a new package, `@scope/vue`, version `0.1.0`, published to npm. No GitHub release was created. The publish job reported success. Underneath it, the reconcile step logged `planned releases: 0`.

Three release scripts each carried their own copy of the published-package list. The new package had been added to two of the three. The planner looked the package up by name and silently dropped what it did not recognise.

The failure hid because an empty plan is also a valid result after a successful publish. A rerun after a real publish correctly plans nothing. So the same zero can mean success or blindness.

There was no error anywhere. The fix was one list in one module imported by all three scripts, and a test that the list matches the workspace.

## A cleanup job can exit 0 before it touches anything

In operations on a staging search cluster, a weekly Rundeck job was supposed to clean orphaned search documents for a list of restaurant groups. Every run was green. The problem it was meant to fix came back every Monday.

The job published one AMQP message per group and exited 0 as soon as publication succeeded. The delete happened later, in a worker. So `1 ok` meant messages published, not documents deleted.

Four things made a green run fail to complete the cleanup. The CSV held the wrong kind of uuid, format-valid but matching no document. The index list omitted the target index. The group that broke the failing test was not in the file. The service lacked a cluster permission, `scroll`, so every delete-by-query removed exactly one batch of 1000 documents and then threw.

A group with fewer than 1000 documents was emptied and looked like proof the job worked. A group with more was left partly deleted. One measured group had 3609 documents: 2483 after the scheduled run, 1483 after one manual call, and back to 3609 after re-indexing in about 90 seconds.

The deleted count existed in exactly one place, a worker log line `Deleted N documents`, and even that count does not prove the job finished: the partial deletions logged a count too. The check that works is the group's document count after the run against the expected state. The permission was granted, and the upstream cause, a stale weekly restore, was fixed separately.

## CI can be absent and still look like it is waiting

In the backend API, the CI workflow declared `pull_request: branches: [master]`. A pull request based on a feature branch reported `no checks reported`, which reads like CI pending rather than CI absent. Reviewers waited for checks that would never run.

There was a second layer. A Makefile target, `make test-unit`, was a stub that echoed `NOT_FOUND`. The real command was `npm run test:unit`. Two layers of "looks like it ran".

The fix was to run CI on all pull requests, or at least say in the PR body that it does not run and validate locally.

## Two one-liners carry the same shape

The first one is `npm view pkg@missing-version`. It exits 0 with empty stdout. Only a missing package exits non-zero. Any check that uses the exit code to ask whether something is published reports `already published` forever. The only signal that matters is the output.

The second one is a `private: true` flag. You cannot verify it with `npm publish --dry-run`. The command packs and exits 0 regardless. A unit test has to assert the flag directly.

Both are the same problem in miniature: the command ran, and nothing about that proves the thing I cared about.

## Verify the destination, and give zero two names

The repair is the same each time: check the expected final state where the effect lands. The report accepted by the coverage service, the version on the registry, the release on GitHub, the document count in the index, the check run on the pull request. Not the command's exit code, and not the job's colour.

The monorepo's publish job now asks the registry for the exact version of each package before deciding whether to publish, and fails open when the registry cannot answer, because a missing answer is not a positive answer.

The hardest case is the legitimate zero. A rerun that has nothing to publish and a planner that did not recognise a package both report zero. When "nothing to do" is a valid outcome, the pipeline needs a second signal for "did not recognise the input", or the two stay indistinguishable.

I call the pattern green by absence: a status derived from the non-occurrence of an error instead of the occurrence of the effect. The repair has its own failure mode. A verification step can become one more step that cannot fail, which moves the lie one layer deeper.
