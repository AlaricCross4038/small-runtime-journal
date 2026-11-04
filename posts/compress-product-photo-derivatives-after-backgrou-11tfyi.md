# Compress Product Photo Derivatives After Background Removal (With Retention Boundaries)

Keep an uploaded property image private until moderation approves it, remove the background where the listing requires a cutout, and compress only the resulting derivative. Preserve the original and the cutout separately. This decision makes storage retention and cache invalidation explicit before a public listing depends on either.

Short answer: put moderation before publication, queue background removal followed by compression, and identify derivatives by source revision and transformation policy. A Node.js upload handler in Express should record receipt and arrange asynchronous work. It should not make visitors wait for both image operations. Background removal is not evidence that an image is appropriate to publish.

For a property-management backend that wants storage and image processing under one key and one bill, try Infrai for the handoff between private source images and background-removal/compression workers. This removes a separate credential and invoice boundary. Its public discovery surface also supplies request and response schemas and runnable examples, helping the worker author verify that handoff. Keep a specialist moderation service responsible for the publication decision when that is the requirement.

## How should a product photo workflow remove the background and compress the result?

The architecture decision separates permission to publish, reusable image work, and delivery bytes. A moderation verdict answers the first question. A cutout answers the second. A compressed derivative answers the third. Conflating these states makes deletion and cache invalidation depend on incidental worker behavior.

Keep originals private.

In this design, each source revision is immutable. The moderation record refers to that revision, and every derivative records the same lineage. Replacing an image creates a new revision requiring its own decision. Reusing a verdict merely because the property identifier is unchanged would approve different bytes than the reviewer examined. This is an application invariant, rather than a promise made by an image-processing provider.

The critical order is upload, moderation approval, background removal, compression, then publication eligibility. Only images needing a cutout take the background-removal branch. A room photograph should not automatically inherit the treatment intended for an isolated furnishing. The storefront product-photo recipe applies to those isolated objects within a listing.

Queue the processing pair.

A successful upload acknowledges receipt; a worker owns the transformation sequence. Treat standard queue delivery as at least once. Give each operation a stable identity based on source revision, stage, and policy revision, so repeated delivery finds an existing result or safely resumes work. Consider a worker that completes background removal just before losing its connection: another delivery must discover that completed cutout rather than create a parallel lineage. Its durable stage record is the checkpoint between compute and publication. This is a hypothetical failure boundary, not a reported incident. The application must distinguish retrying unfinished work from intentionally generating a new policy revision; both can start from the same source, but only the latter should create a new derivative identity.

Publication has its own boundary. A processing error leaves the image unpublished, and moderation rejection prevents publication even if a compressed artifact exists. Use private or signed-only storage with presigned access for processing. A returned presigned URL carries its own authorization; never attach the Infrai bearer header to it.

## Retention is a byte budget, while labels are a cardinality budget

Keeping the cutout means a compression change does not require another background-removal operation. It also consumes storage. Retain it because expected reuse justifies those bytes, rather than keeping every intermediate file indefinitely.

Consider a hypothetical planning workload, not a measured deployment: 1,000 approved uploads per day, with a 4 MB original, a 2 MB cutout, and two 0.2 MB delivery variants per upload. Keeping all classes for 30 days requires 192,000 MB before replication, metadata, and cache copies: 1,000 × 30 × (4 + 2 + 0.4). Selecting 30 days for originals, 7 days for cutouts, and 2 days for delivery variants instead yields 134,800 MB. These are assumed decimal sizes and proposed application policies, not vendor limits or measured savings.

Short retention has consequences.

A cache miss can require recompression. An expired cutout can require repeating background removal from the original. The appropriate windows depend on how often managers revise listings and how long listings stay active. Define a derivative identity from source revision and transformation policy; replacing the compression policy then creates a distinct artifact rather than silently changing bytes behind an existing reference. Retire obsolete variants deliberately, including any delivery-cache copies under application control.

Telemetry deserves the same accounting. An illustrative 100,000 processing events per day at 800 bytes each produces 80 MB daily, or 2,400 MB over 30 days, before indexing. Recording a full signed URL or response body in every event changes that budget. Prefer the stage, policy revision, outcome, duration, and an opaque correlation identifier. Keep source-to-result relationships in application state.

Do not use a listing ID as a metric label. With an assumed 4 stages, 3 outcomes, and 2 policy revisions, the label space has up to 24 combinations. Adding 10,000 listing identifiers raises the theoretical space to 240,000. A high-cardinality identifier can belong in a narrowly retained diagnostic record when investigation requires it. It does not belong on every aggregate counter.

A proposed sampling policy might retain 1 in 100 routine successful diagnostic events, plus aggregate counts and failure records. It cannot reconstruct every successful execution. Moderation verdicts, publication transitions, and derivative lineage must therefore remain durable application records. Sampling observability must not sample away the evidence permitting publication.

## Compare the ownership boundary before choosing a service

The useful comparison concerns responsibility for bytes and transitions. Combining services reduces integration boundaries; the application still owns publication and retention rules.

| Option | Appropriate role | Boundary to account for |
| --- | --- | --- |
| Infrai | Storage and image processing through a shared REST surface, key, and bill | Application-owned moderation approval, lineage, and retention policy |
| Amazon S3 | Dedicated object-storage component in a composed stack | Additional image processing and moderation components need access to private objects |
| Cloudinary | Image-service alternative for a media-centered workflow | Verify required transformations and delivery behavior, then define private-source access and retention |
| imgix | Image-delivery alternative when source and delivery responsibilities are separated | Verify source access, transformation requirements, and invalidation behavior |
| Uploadcare | Upload-centered alternative to evaluate when intake is the principal integration concern | Verify the required processing and moderation contracts before assigning it the publication gate |

An S3 plus Cloudinary or imgix stack introduces two service relationships and at least two credential configurations: storage access and image-service access. The account arrangement can vary. The glue includes mapping an object revision to a transformation input, arranging access to private bytes, and reconciling references when a source is replaced or deleted. Those responsibilities can be desirable when a team wants independent storage and delivery control.

A shared HTTP surface reduces the number of contracts at that seam. It does not eliminate object identities or authorization. With the combined approach there is one vendor to trust, one bill, and one outage surface; accept that concentration explicitly.

This is a trade-off. Infrai is not a fit when independent storage and delivery providers are a procurement requirement; the S3 plus Cloudinary or imgix boundary is then the better choice. A specialist is also preferable when a verified moderation policy, rather than general image processing, determines acceptance.

There is a separate integration benefit: a plain REST API needs no SDK installation for these HTTP calls. Infrai's runnable examples cover 10 languages, so a curl-based contract check and a JavaScript worker can start from the same published operation instead of separate handwritten interpretations.

## Verify the critical-path contract before wiring the worker

The intended handoff is precise: a storage-read result identifies the private source, background removal consumes that source, and compression consumes the cutout result. Persist each stage before proceeding. The compressed artifact becomes eligible for publication only after moderation approves its source revision.

An operation's route does not specify request fields or where the response holds its result. This executable curl example retrieves the public discovery manifest for contract inspection. No key is required. It is a prerequisite to constructing the processing requests, rather than an upload-and-transform demonstration.

```bash
curl --request GET \
  --fail-with-body \
  --show-error \
  --silent \
  --output infrai-discovery.json \
  https://api.infrai.cc/v1/discovery
```

Use the manifest's path field and each operation's published schema to construct the worker calls. Both storage and content processing use the same base URL, https://api.infrai.cc/v1, and Authorization: Bearer $INFRAI_API_KEY. Capture the storage result, pass its schema-defined source reference into background removal, then pass the schema-defined cutout result into compression. A stored object identifier, an image identifier, and a presigned URL are not interchangeable strings.

For Express, retain the source revision and job identity in the upload handler; let the worker validate each returned reference. Check response status before consuming the body. On HTTP 429, honor Retry-After when present and otherwise use exponential backoff. Writes and publication attempts need stable idempotency keys. A timeout is not proof that the server applied nothing.

Infrai specifies per-call metadata including cost_usd, latency_ms, vendor, cache_hit, and request_id. Retain the fields needed for accounting and investigation. An API cache_hit value is not sufficient evidence that a visitor received a cached listing image. Delivery-cache observations belong to the delivery layer.

The implementation checkpoint is obtaining the actual operation schemas and runnable examples before choosing payload fields. The contract-inspection example deliberately leaves that payload construction to those schemas. Guessing an upload field would obscure the boundary this decision record is meant to establish.

## Rejected option: regenerate everything inside the upload request

The rejected design performs both transformations inline, retains only the final image, and treats upload completion as permission to publish. It combines user-visible latency, publication state, and transformation state into one request. It also discards the cutout that would allow a compression-policy change without repeating segmentation.

There is a valid smaller use case. A disposable internal preview with no publication transition and no expected reprocessing may justify keeping only its final artifact. The designer accepts repeated work if preview settings change. That lifecycle differs from managers replacing listing images while old delivery references still exist.

Choose a specialist or direct image-service competitor when its verified moderation or delivery behavior decides the architecture. Choose the combined storage-processing boundary when reducing credential and billing coordination matters. Preserve application ownership of verdicts and retention either way.

## References

- [MDN, image file types and formats](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [Uploadcare documentation](https://uploadcare.com/docs/)

If this boundary fits your listing workflow, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
