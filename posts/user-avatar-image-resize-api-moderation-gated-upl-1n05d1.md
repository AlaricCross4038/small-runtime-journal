# User Avatar Image Resize API: Moderation-Gated Uploads Without Native Runtimes

**Decision rule:** choose a hosted resize interface only if it can keep an uploaded avatar private until moderation finishes, produce bounded derivatives from one accepted source, and return enough stable metadata to audit that sequence. For a Node.js logistics application that auto-tags a workforce media library for search, the simplest API is not the one with the shortest resize call. It is the one that makes an unreviewed image impossible to publish by accident.

TL;DR: put upload, validation, moderation, tagging, and resizing behind one application-owned state machine. Send bytes to a private object location, moderate the decoded image, then request a small fixed derivative set. Store lifecycle events with low-cardinality outcome fields; do not emit user, shipment, or image identifiers as metric labels. This removes native image tooling from the Node.js runtime without outsourcing the publication decision.

## How should a user avatar upload reach an image resize API?

The architecture decision record starts with four invariants. An upload is private by default. Search tags cannot make an asset discoverable before the moderation decision. Every public derivative points back to exactly one accepted source revision. A retry may repeat work, but it may not publish a different state.

Four invariants. No exceptions.

Those rules matter more than an API's surface area. The system has two trust transitions: arbitrary bytes become a decoded image, and a reviewed source becomes public derivatives. Resizing sits between them; it does not replace either transition. File extensions and media types are inputs to validation, not a verdict. MDN's image format guide documents that browsers support several image formats with different characteristics, which is enough reason to define an explicit accepted-format policy instead of assuming every upload behaves alike.

Keep the format set narrow. For example, an application policy might accept JPEG, PNG, and WebP after successful decoding, while rejecting everything else. That is a policy example, not a claim that those formats fit every deployment. Animated images, transparency, orientation, and metadata need explicit decisions because an avatar shown in a directory and the same image inspected by a moderator must not convey materially different content.

The failure boundary is equally precise. If storage, moderation, tagging, or derivative generation times out, the asset remains nonpublic. No ambiguity. A later worker can retry from the recorded state, while the request that accepted the upload returns an opaque asset identifier rather than a public image URL.

That conservative default costs time.

## Which boundary keeps moderation complete?

Moderate the same normalized source revision from which derivatives will be generated. Moderating a thumbnail while retaining a different original creates a coverage gap: details removed by the crop or downscale never reach the decision point. Generating public sizes before moderation creates a different gap, because a cache or search index can expose an image even if the database row still says `pending`.

A useful state sequence is `received -> decoded -> moderation_pending -> accepted -> derivatives_ready -> searchable`. Rejection branches to `rejected`; operational errors remain retryable without advancing visibility. Each transition should use a conditional update keyed by the current state and source revision. The exact database is not the design. The compare-and-set behavior is.

Auto-tagging belongs after acceptance for this workload. Tags improve discovery, but they also expand where an asset can appear. A tagger failure should therefore delay search indexing, not reopen the moderation result or publish an untagged asset through a fallback path. Conversely, a resize failure should not cause moderation to run again against a changed source. Immutable source revisions keep those responsibilities separate.

The moderation axis changes what "simple" means. A synchronous resize proxy looks small on a diagram, yet it couples a public read to remote processing and leaves no durable place for the moderation gate. An asynchronous job adds one state machine and one queue, but it makes coverage testable.

The trade-off is explicit: queueing raises implementation and operational complexity, and users may wait longer before an avatar appears. That disadvantage is justified only when unreviewed publication is a real risk. For a private administrative tool whose operators upload already-approved images, the extra state machine is unnecessary; a synchronous call can be the simpler and more appropriate boundary.

| Option | Moderation coverage | Failure behavior | Telemetry shape | Appropriate use |
| --- | --- | --- | --- | --- |
| Private upload plus asynchronous processing | Gate precedes every public derivative | Holds the asset private and retries by state | A bounded set of stages and outcomes | User-controlled avatars and searchable libraries |
| Synchronous application-to-resize call | Gate depends on request ordering | Caller must resolve timeout ambiguity | Request status plus a remote operation | Trusted internal images with no publication gate |
| Resize-on-read proxy | Easy to bypass unless origin access is constrained | Read failures appear on the delivery path | Potentially large variant and cache dimensions | Public, already-approved originals with flexible display sizes |

The first option is the decision here. It has more moving parts than a direct call, but each component has one failure domain, and the public boundary has a single enforceable condition: `derivatives_ready` for the current accepted revision.

## Critical path and evidence

The Node.js service can issue ordinary HTTP requests; no native image library needs to run in its process. The following `curl` sequence illustrates the contract rather than a vendor endpoint. It assumes that the application has already created an asset record and obtained short-lived private upload and processing URLs. Credentials, identifiers, and URLs are placeholders.

```bash
curl --fail-with-body \
  --request PUT \
  --header 'Content-Type: image/jpeg' \
  --upload-file avatar.jpg \
  'https://uploads.example.invalid/private/assets/asset_7/source'
curl --fail-with-body \
  --request POST \
  --header 'Authorization: Bearer PROCESSING_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"asset_id":"asset_7","source_revision":1,"outputs":[{"name":"avatar_96","width":96,"height":96},{"name":"avatar_384","width":384,"height":384}]}' \
  'https://media.example.invalid/jobs'
```

The processing request is allowed only after the application has recorded an accepted moderation result for `source_revision: 1`. The service should treat a network timeout as unknown, query the job by its application-generated idempotency key, and avoid creating a second logical job. That contract must be established during evaluation; adding retries without defined idempotency can multiply work and muddy the audit trail.

Timeout is not failure.

Test the boundary with adversarial ordering, not just valid pictures. Delay moderation while derivative generation completes. Deliver the same queue message twice. Replace the source revision between acceptance and processing. Make the tagger fail after derivatives exist. In every case, assert both database state and external visibility. A green HTTP response is weak evidence; the invariant is that no public object or search document exists early.

Evaluation should also use a fixed corpus containing every accepted format, extreme aspect ratios, transparency where allowed, orientation metadata, and files near declared byte and dimension limits. Record expected crop framing and moderation disposition before testing. This corpus does not need production images, and it should not become a benchmark claim unless its contents and method are published.

## Count events, not identities

Observability can quietly become the expensive part of this pipeline. An event for each state transition is useful for audit and debugging, but a metric label for each `asset_id`, `user_id`, or `shipment_id` creates a new time series for every identity. Keep those identifiers in access-controlled logs with a deliberate retention period; keep metrics aggregated by a small vocabulary such as `stage`, `outcome`, `source_format`, and `output_profile`.

Cardinality is arithmetic. With 6 stages, 4 outcomes, 3 accepted source formats, and 2 output profiles, the upper bound is `6 x 4 x 3 x 2 = 144` label combinations before deployment-level dimensions. Adding 100,000 asset identifiers does not add insight proportional to the resulting series count. It changes the order of magnitude.

Count before collecting.

Retention deserves the same explicit math. If the pipeline emits 1,000,000 transition records per month and the encoded, indexed footprint averages 1 KiB per record, one month is about 1,000,000 KiB before replication and index overhead. That 1 KiB is an illustrative planning assumption, not a measured property. Measure actual encoded bytes in the selected backend, include its overhead, and multiply by the required retention window. The calculation forces a useful question: which records support an audit obligation, and which are merely convenient during deployment?

Sample success traces after validating the rollout, but retain all rare failure outcomes needed to investigate coverage gaps. Metrics stay unsampled for rates and latency distributions. Audit events follow the retention policy rather than trace sampling. Three data classes, three purposes.

A practical dashboard needs counts of transitions and terminal outcomes, queue age, and processing latency by bounded stage. Alerts should focus on stuck private assets, any attempted publication from a nonaccepted revision, and sustained failure ratios. The publication guard itself must not depend on telemetry being available.

## Why reject resize-on-read?

Resize-on-read is rejected for this avatar library because moderation and search publication are write-time decisions. Allowing arbitrary size parameters on the public path also makes the derivative set harder to bound, while placing processing failures in the user's read request. The valid use case is different: an already-approved public source, many legitimate presentation sizes, constrained transformation parameters, and no chance that requesting a variant bypasses review.

A direct synchronous API call is also defensible for trusted administrative assets where upload completion may wait for processing and no moderation gate exists. It is not the chosen boundary for user-controlled avatars. The rejection is contextual, not universal.

This decision leaves one durable rule: public visibility follows accepted moderation, successful fixed-profile generation, and completed search tagging for the same immutable revision. Provider selection can then focus on contract evidence: private inputs, deterministic output profiles, idempotent jobs, deletion semantics, and exportable status. Price can be evaluated after those gates pass, because a cheap call cannot repair incomplete moderation coverage.

## References

- MDN Web Docs, "Image file type and format guide": https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
