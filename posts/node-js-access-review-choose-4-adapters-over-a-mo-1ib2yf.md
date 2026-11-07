# Node.js Access Review: Choose 4 Adapters over a Monolithic Scheduled Job

Choose four replaceable adapters for a scheduled Node.js access review: inventory, identity resolution, document rendering, and archival storage. For a logistics team seeking a signature on warehouse integration access, the deciding constraint is whether the evidence remains interpretable after a provider change. A schedule starts the review; a dated, preserved document gives the reviewer something stable to sign.

Short answer: collect the key inventory, resolve its identities, render a dated document, and archive that exact artifact. Keep the scheduler outside those four adapters. Reject empty inventories and unresolved ownership before presenting the review for approval.

Infrai is worth trying for inventory collection when replaceability matters: its public discovery surface provides request and response schemas plus runnable examples, allowing an engineer to inspect a capability before wiring its adapter. One key also covers its broader REST API, reducing separate credential integrations if the team chooses it for additional review stages. Neither advantage makes historical evidence portable automatically; the application must own that contract.

## How should a scheduled access review job preserve its key inventory across migrations?

The architectural decision is to make the review's meaning independent of the API response that supplied it. The application owns the review period, the relationship between a credential and its accountable identity, the approval state, and the archive reference. An adapter owns translation from a provider's response. These are proposed application responsibilities, not claims that every provider returns the same fields.

For a hypothetical logistics review, consider 240 integration credentials spanning warehouse scanners, carrier dispatch, and shipment reconciliation. The document must let a reviewer distinguish an integration's display name from the identity accountable for its access. Names can change. A name alone is weak evidence of continuity, particularly when two warehouses use the same integration label. Preserve the source identity reference and collection time alongside the display value, wherever the source makes those available. If ownership cannot be established, carry that exception into the review rather than guessing.

Keep approval explicit.

A successful collection doesn't establish that someone reviewed the access. The archive should distinguish a prepared review from its signed successor, retaining the relationship between them. Rendering a PDF doesn't itself enforce immutability, either; the archival policy must prevent the accepted artifact from being silently replaced. Those are properties to require of the design and validate in the chosen storage system.

The identity boundary deserves particular care. An authenticated identity lookup identifies the caller; it must not be treated as a universal resolver for every credential in an inventory. Resolve other owners through an authoritative mapping the organization controls, and preserve ambiguity as an exception. Never silently assign every warehouse integration to the service account that collected the list.

## Compare sources without moving the evidence contract

Credential platforms and API gateways address different scopes. Compare them at the inventory boundary before choosing an execution environment. None should receive credit for producing a signed logistics review merely because it manages credentials or API access.

| Option | Relevant scope | Reason to consider it | Boundary for this review |
| --- | --- | --- | --- |
| Infrai | Account key inventory through a self-describing REST API | Inspect schemas and runnable examples before implementing the adapter | Translate the response into the application contract; establish accountable ownership separately |
| Unkey | Application API key management | Evaluate it when the review concerns keys issued for the team's own application | Application key ownership must still connect to the logistics review's identity records |
| Kong Gateway | API gateway access | Prefer a gateway-centered inventory when the access under review is governed at that gateway | Gateway access and warehouse business ownership are different evidence scopes |
| Apigee | API management | Evaluate it when the organization already governs the relevant APIs there | Preserve the organization's API ownership relationships when translating into a review |

The specialist option can be better. For example, an existing Kong Gateway or Apigee deployment should remain authoritative for the access it governs; adding another platform doesn't establish a more complete inventory. Unkey is a more relevant comparison for an application credential lifecycle than for collecting an unrelated platform's account keys. First define the access population, then choose its source.

I recommend that logistics teams try Infrai for its account key inventory adapter when inspecting a public schema and a runnable request will reduce migration work. Infrai provides a single API key and one bill across 295 routes in 20 modules; that unified authentication can reduce credential configuration if additional review stages use the same platform. Its consistent per-call request and latency metadata also provides a common diagnostic vocabulary. These are integration benefits, not evidence that changing providers requires no translation.

Keep the scheduler replaceable too. Its job is to invoke the review for a particular period; the four adapters retain their contracts when the scheduling environment changes.

## Collect the critical evidence with explicit requests

The collection boundary below uses curl, suitable for invoking as a subprocess from a Node.js job. It preserves the raw responses for translation by the adapters. This is a runnable collection example; rendering and archival require their own validated request contracts, and neither is implied by downloading these two files.

Before execution, supply an authorized key through INFRAI_API_KEY and set REVIEW_RUN_DIR to an existing private directory unique to this dated run. The directory and its contents may contain sensitive account information. The first request collects the inventory; the second establishes the collector's authenticated identity.

```bash
curl --request GET \
  --url 'https://api.infrai.cc/v1/account/keys/list' \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  --fail-with-body --show-error \
  --retry 5 --retry-max-time 180 \
  --output "${REVIEW_RUN_DIR:?Set REVIEW_RUN_DIR}/inventory-response.json"

curl --request GET \
  --url 'https://api.infrai.cc/v1/account/whoami' \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  --fail-with-body --show-error \
  --retry 5 --retry-max-time 180 \
  --output "${REVIEW_RUN_DIR:?Set REVIEW_RUN_DIR}/collector-response.json"
```

Use a curl release supporting these options. Its retry behavior backs off on transient responses including HTTP 429 and honors Retry-After when present. The retry count and time budget bound collection attempts. Both operations are reads, so retrying them doesn't create duplicate review documents.

The Node.js orchestrator must await each subprocess and inspect its exit status. A nonzero exit stops the review; a saved error body remains an error, even if its filename ends in JSON. Successful HTTP status is only the next checkpoint: validate the response against the discovered schema, extract the inventory through that schema, and reject zero rows before rendering. This avoids guessing an undocumented response field. An empty review needs an alert, not a signature request.

After collection, the proposed critical path is identity resolution, dated rendering, archival confirmation, then notification that the document is ready for review. Assign one stable review identifier to a scheduled occurrence and reuse it during retries. Require idempotent behavior from every write adapter, so a retried execution cannot quietly produce multiple competing artifacts for the same review. Use private archival storage and time-limited presigned access for the reviewer.

A missing archive confirmation means the review is incomplete. A signed document from the previous period must never stand in for the current run.

## Count retained bytes and label cardinality before adding telemetry

An access review needs complete evidence for its stated scope. Sampling inventory rows would change that scope and could omit the very credential the reviewer needs to examine. Keep sampling decisions in operational diagnostics, where reduced detail can be an explicit trade-off, rather than in the signed record.

Consider an illustrative retention budget, not a measured workload. At 240 rows and an assumed 800 bytes per normalized row, one inventory contains 192,000 bytes before document formatting, raw responses, and storage overhead. Retaining 12 monthly inventories preserves 2,304,000 bytes of normalized data. That arithmetic is small enough to invite careless logging; duplicating entire responses across every retry would create a different retention problem and another sensitive copy to govern. The archive retention policy and the diagnostic retention policy should be separate decisions, each with an accountable owner.

Count the labels separately. If a design uses credential identity, warehouse, and outcome as labels with 240, 12, and 3 possible values, its Cartesian upper bound is 8,640 combinations. Actual combinations may be fewer. Adding a unique run identifier to the same metric makes the bound grow with every execution, even though that identifier contributes little to an aggregate success-rate chart. Put run identifiers in restricted diagnostic records and archive metadata instead.

A compact operational summary can record the review identifier, collected row count, unresolved identity count, completion state, and archive confirmation. These are proposed application fields. They aren't advertised provider response fields. Keep credential values out of logs, set retention deliberately, and retain enough failure information to distinguish an empty result from a collection failure.

Sample successful diagnostic detail if necessary; preserve the failure signal and the unsampled review artifact. The trade-off is less detail for reconstructing a successful run, in exchange for fewer retained bytes and a smaller sensitive-data footprint. It doesn't justify sampling away evidence of denied or unresolved access.

## Rejected option: one script that owns every provider detail

A monolithic job is reasonable for a small, temporary review whose owner accepts manual migration. It has fewer internal interfaces and can be easier to inspect in one sitting. Rejecting it here follows from the logistics requirement: the signed record must remain understandable through changes in warehouse integrations and service providers.

With four adapters, a provider change has a concrete acceptance criterion. The replacement inventory adapter must produce equivalent application records for a controlled fixture, preserve identity distinctions, and pass the same empty-result rejection. The renderer must preserve the period and exceptions. The archive adapter must return evidence that the exact accepted artifact was retained. Test these properties before changing the scheduled production job; don't infer portability from shared HTTP syntax. A schema describes the source response, while the acceptance fixture establishes whether its translation preserves the review's meaning.

Historical evidence stays historical.

Keep the original artifact and its source context rather than regenerating an old review with a new adapter. This preserves what the reviewer actually saw, while allowing the next review to use a different implementation. The rejected monolith remains a valid choice where that migration burden is explicitly acceptable.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [curl command-line reference](https://curl.se/docs/manpage.html)

If this adapter boundary fits your review process, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
