# Vector Search vs Web Search for Node.js Help Centers: Ranking Signals at Scale

Short answer: use vector search for tenant-scoped meaning, web search for exact terms and fresh pages, and combine their scores only after enforcing authorization and freshness. In a multi-tenant SaaS help center, the ranking signal that matters most is the one you can explain and measure without allowing one tenant's corpus to leak into another's.

## Architecture decision record

The invariant is simple: every candidate carries a tenant identifier, an authorization scope, a source revision, and a timestamp. Retrieval may be approximate; access control may not be. A query should be normalized once, sent to both retrieval paths when useful, and merged in a service that can log the reason each result appeared.

| Option | Strong signal | Failure boundary | Index-cost shape |
| --- | --- | --- | --- |
| Vector search | Semantic similarity across paraphrases | Similar-looking articles can outrank the exact policy; embeddings lag edits | Embedding storage and refresh work grow with content revisions |
| Web search | Terms, fields, links, and recency | Synonyms and short queries can miss the intended concept | Inverted indexes are efficient, but analyzers and replicas still consume storage |
| Hybrid ranker | Reciprocal rank or learned blend of both lists | Score calibration and tenant filters become operational work | Two indexes plus merge telemetry; cost is visible and controllable |

I keep the merge deterministic. A practical baseline is reciprocal rank fusion: for each result, add `1 / (k + rank)` from each list, then apply boosts for an exact title match, the user's locale, and a recent revision. The constants belong in configuration, not in a prompt. They should be evaluated with a held-out set of real questions, stratified by tenant size.

## How should vector search and web search rank multi-tenant SaaS help center answers?

Start with hard filters. The query for tenant `acme` must never retrieve a document whose `tenant_id` is `beta`, even if the text is identical. Apply the filter inside each index request and repeat it at merge time. Defense in depth is boring; a cross-tenant answer is worse than a slow answer.

Then separate relevance from freshness. A web-search score often rewards term overlap, while a vector score rewards semantic proximity. Neither score is a probability, so adding them directly is misleading. Rank-based fusion avoids pretending that two unrelated scales are comparable, although it can hide a weak candidate near the cutoff. I am not sure one fixed `k` works for every tenant; a small evaluation table will tell you more than intuition.

For a help-center request, return evidence with each passage: document ID, section heading, revision, and canonical URL. The generation step should quote only those passages. This is the retrieval-augmented generation pattern described by Lewis et al.; the retriever supplies external memory while the generator conditions on it.

The critical path can remain an ordinary HTTP contract. Here is a deliberately vendor-neutral sketch using Node.js-facing endpoints:

```bash
curl -sS -X POST https://search.example.test/v1/vector/query \
  -H 'content-type: application/json' \
  -H 'x-tenant-id: acme' \
  -d '{"q":"How do I rotate an API key?","top_k":20,"tenant_id":"acme"}'
```

The response should include per-signal ranks and the policy decision that admitted each result. Do not log raw customer questions by default. Hashing a query can support deduplication, but it cannot support debugging intent; sample a small, consented slice instead and retain it for a defined period.

## Cost, operations, and the rejected shortcut

Index cost is a lifecycle problem, not a one-time provisioning number. Store one canonical document, derive chunks for retrieval, and delete superseded chunks after a grace period. If an article is edited ten times before publication, queue one embedding job for the final revision. That reduces write amplification and keeps stale vectors out of the candidate set. In a fintech help center, imagine a payment-limit article revised for three API versions in one afternoon: the index needs all active revisions for supported clients, but it should not retain every draft chunk indefinitely. A revision-aware tombstone, a publish timestamp, and a bounded grace window make that policy explicit. I count those bytes because the observability bill follows them; a dashboard that stores one high-cardinality label per chunk can cost more than the query path it describes.

Three words: measure the boundary.

Measure bytes per tenant, documents per tenant, refresh lag, and the 95th-percentile merge latency. Cardinality matters: a label for every document ID can make observability more expensive than retrieval. I would rather keep a bounded tenant label and sample document-level traces than retain every identifier forever.

Failure handling should preserve a useful, honest response. If vector retrieval is unavailable, a filtered lexical query can still answer exact error-code questions; if lexical retrieval returns no candidates, semantic retrieval can try paraphrases. Mark the fallback in telemetry and expose a confidence reason to the caller. Never silently widen the tenant filter to improve recall.

I would reject a vector-only design for policy-heavy help centers. It can retrieve a conceptually similar article while missing a required phrase such as an error code, version, or legal qualifier. I would also reject a web-only design when users ask in natural language and the documentation uses different terminology.

A single path is still valid for a narrow corpus: web search alone fits a small, terminology-stable site with frequent exact lookups; vector search alone fits an internal, low-risk knowledge base where paraphrase recall is the main goal. The catch is that neither choice removes the need for tenant isolation, revision handling, evaluation, and retention limits. Stick with the simpler path when its measured miss rate is acceptable.

Adopt hybrid retrieval as the default architecture, with explicit tenant filters and rank-based fusion. Review it with two queues: judged queries for relevance, and adversarial queries for isolation. Track nDCG or recall at a fixed cutoff, but pair those metrics with stale-result rate and index bytes per active tenant. A result that is relevant but unauthorized is a failed request, not a low score.

Re-run the evaluation after analyzer changes, embedding-model changes, or a large documentation migration. Keep the old index long enough to compare outputs, then retire it deliberately. This is where the architecture earns its keep: ranking signals can evolve without changing the authorization boundary.

## References

- https://arxiv.org/abs/2005.11401
- https://www.rfc-editor.org/rfc/rfc9110
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag
