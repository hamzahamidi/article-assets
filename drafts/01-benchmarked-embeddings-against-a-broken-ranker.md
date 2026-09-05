# I benchmarked embeddings against a broken ranker
*Before you add a vector channel to your retrieval, read your fusion loop.*

## The benchmark setup
I was building a local-first code retrieval tool for coding agents, exposed over MCP. It indexed 88 repositories, 108k files, mostly TypeScript, and 59k documentation chunks on a laptop with SQLite plus FTS5.

The system was hybrid by design. Several channels, code full text, path, trigram, exact identifier, endpoint, graph edges, and one prose channel, each returned a ranked list, then weighted reciprocal rank fusion merged them.

The metric I cared about was BCR, budget-constrained recall at 8192 bytes: how much of the ground truth an agent gets inside a fixed byte budget. Recall@10, nDCG@10 and MRR were reported too.

## Five gates, written down before any model ran
Embeddings had been declared in config, weighted per-intent, and never implemented. The vector channel existed in name only, and the local provider was a hashing stub.

Before running anything I wrote down five gates. All five had to pass for embeddings to ship enabled.

1. Documentation quality: dBCR >= +0.05, CI lower > +0.02.
2. No regression on identifier queries: dBCR >= -0.01.
3. p95 latency increase: <= 80 ms.
4. Index size increase: <= 40%.
5. Full corpus embed: <= 45 min.

I used `bge-small-en-v1.5`, 384 dimensions, on-device through ONNX. It embedded 58,609 chunks in 38.5 minutes, and the index grew 115 MB on 1,003 MB, which was 11.4%.

The evaluation sets came from the real corpus, because the synthetic fixture only had 8 document chunks. I built 62 documentation questions, each answered by one unique chunk and phrased without sharing a rare term with it, and 40 identifier queries, each naming a symbol with exactly one exported definition.

## The first run looked convincing and was wrong
Embeddings won every quality metric on documentation questions. Then gate 2 failed: a drop of 0.025 on identifier queries.

The traced example was neat enough to fool me. One query was a symbol name that is also an ordinary English word: Achievement. The intent classifier read it as "implement a task", the intent where the vector channel weighs 1.0, and three loyalty-domain documents at cosine similarity 0.71 to 0.75 pushed the real type definition from rank 2 to rank 6.

I wrote the ADR with that conclusion: the queries embeddings help and the queries embeddings hurt share an intent, so no intent-level switch can separate them. Every observation in it was accurate. The conclusion was wrong.

## Latency exposed the bug before quality did
The 62 documentation questions ran at p50 145 ms and p95 305 ms against a 120 ms ceiling. Short identifier queries were fine, with p50 16.7 ms.

The prose channel, `nl_fts`, cost 110.3 ms of a 154 ms total. It ran two searches: BM25 over symbol names (80.0 ms) and BM25 over documentation chunks (30.4 ms). Searching symbol names with prose terms was 52% of the entire query.

Pruning common terms did not help, and I recorded that. Disabling the symbol half did: documentation retrieval got faster and better at the same time, which a real trade-off does not do.

## The bug was a list, not a model
The channel built one array:

```ts
hits.push(...symbolMatches)
hits.push(...docMatches)
```

Fusion assigns rank by array position:

```ts
let rank = 0;
for (const hit of hits) {
  rank++;
  contribution = weight / (k + rank)
}
```

`k` is 60.

When the symbol search returned its full 100 rows, the best document match sat at rank 101 whatever its score: `w/161` against `w/61` for the symbol at rank 1. Documents were suppressed 2.6x for being second in a push. Nobody decided that ordering. It was a list built by two push calls.

## The fix gave documents their own place
I split the channel into `symbol_fts` and `doc_fts`. Each list receives its own ranks and its own weight, and `symbol_fts` is weighted 0 for the intent whose queries are sentences. One channel, one behaviour, one weight.

## What the fix did to the numbers
On the 62 documentation questions the lexical arm moved from BCR 0.131 to 0.204, nDCG@10 from 0.102 to 0.209, MRR from 0.087 to 0.186, R@10 from 0.173 to 0.272, and p95 from 315 ms to 122 ms. The richer arm, with graph and query expansion, moved from BCR 0.151 to 0.188 and p95 from 305 ms to 123 ms.

Three comparisons, kept apart. On the broken baseline, embeddings had reached BCR 0.185. Fixing the ranker alone reached 0.204 (+0.073), with no index cost and less latency. Adding embeddings on top of the fixed baseline reached 0.240 (+0.052): another positive point estimate, and the one the gate is about.

Embeddings were not bad. The ranker was suppressing the thing I wanted to measure.

## The second gate run
| gate | required | measured (after the fix) | result |
|---|---|---|---|
| 1 quality on documentation queries | dBCR >= +0.05, CI lower > +0.02 | +0.052, CI [-0.009, 0.123], p = 0.069 | FAIL |
| 2 no regression on identifier queries | dBCR >= -0.01 | +0.000 | pass |
| 3 p95 latency increase | <= 80 ms | +29.3 ms (126.3 -> 155.6) | pass |
| 4 index size increase | <= 40% | +115 MB on 1,003 MB = +11.4% | pass |
| 5 full corpus embed | <= 45 min | 38.5 min | pass |

The experiment did not clear the predefined quality gate: the point estimate clears +0.05, but the interval includes zero at n = 62. Embeddings stay off until the dataset is larger, not because they showed no benefit.

The Achievement failure no longer exists. The vector channel had been filling a vacuum the ranker created. Once documents ranked properly, its document hits merged with hits the lexical channels already found and displaced nothing.

## The trade was real and I took it anyway
Documents now compete fairly, so where the right answer is code it costs. Corpus identifier queries moved from 0.978 to 0.950 (-0.028). The synthetic fixture's lexical arm moved from 0.589 to 0.524 (-0.065). I swept weights over the plausible range and the regressions did not move, so they are structural, not a tuning miss.

I took the trade for three reasons. The prose gain is larger than the identifier loss and lands on the primary entry point. The old behaviour was a bug, and keeping a bug because a fixture benefits from it is backwards. And prose p95 dropped from 305 ms to 123 ms: 3 ms over the 120 ms ceiling, back within reach.

## I would keep the wrong version of the decision record
I revised the ADR that recorded the wrong conclusion and kept the first version inside it: the measured mechanism, the reproduced query, the cosine scores, and the mistake about where the failure lived. The sentence that matters: a measured mechanism, reproduced and traced to a specific query, was still an artifact of a bug somewhere else. The measurement was right and the explanation was not.

A related trap from the same day. A first latency profile read the prose channel at 53.8 ms, half the truth, because it called the channels with a hand-built context instead of the one the engine builds. The real one enables query expansion, which grows the term list.

## Inspect the fused list before adding to it
Before adding a channel to rank-fused retrieval, I now print each channel's ranked list and check that it is ranked on its own merits. Any channel that concatenates two result sets has this latent bug, and a new channel that helps may be compensating for one you are suppressing.

The gates did their job too. A +0.052 point estimate on 62 items would have been reported as a result without them.
