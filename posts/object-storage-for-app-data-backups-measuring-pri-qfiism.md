# Object Storage for App Data Backups: Measuring Private Report Retention and Access

A report archive's bill starts with bytes multiplied by days retained. Downloads add egress and request charges; audit events add a second retention problem. Short answer: for authenticated customer-support reports, private object storage with expiring signed downloads is a reasonable choice when daily expiry is sufficient and immutable backups are not required. Choose the access contract first, then compare the same storage, egress, and request workload across US and EU locations.

## Which retention term dominates the storage bill?

Consider a planning workload, not a measured deployment: 10,000 reports generated daily, 2 MB each, held for 30 days. That is approximately 600 GB of logical objects at steady state, using decimal units. At 1,000 downloads a day, delivery moves another 2 GB daily before retries. Storage duration dominates the example's retained-byte calculation; a different download rate could make egress the larger bill item. Compare both, along with request charges, instead of ranking services by a single storage rate.

Cutting routine retention to 14 days changes that logical footprint to about 280 GB. The arithmetic is 10,000 × 2 MB × 14, not a claim about any provider's invoice. The lost 16 days matter: support can no longer retrieve an expired report, and backup restoration has a shorter window. Specify the recovery window with the team that handles customer disputes before automating deletion.

Sixteen days disappear.

There is another archive hiding in the telemetry. One access event per daily download creates 30,000 events over 30 days, before denied requests and retries. Store a bounded outcome, timestamp, and internal report identifier where investigation requires them. Never log a signed URL: its query string is a temporary bearer credential. A metric label keyed by report ID creates an unbounded series count as reports accumulate; aggregate by a small set of outcomes instead. Count cardinality before turning on retention.

## Can object storage serve app data backups and signed report downloads?

The application should authenticate the customer and authorize that particular report before obtaining a short-lived signed link. It then hands the link to the customer, without putting the storage credential in a browser or sending the API's Authorization header to the signed URL. Anyone holding the link can use it until expiry, so link lifetime and object lifetime solve different problems. Keep its lifetime tied to the download workflow.

A private-only service is enough for scheduled database dumps, compressed exports, and customer report retrieval. It is the wrong tool for permanent public report links or a static website. Day-granularity lifecycle rules can remove old objects, but they cannot meet an hourly deletion deadline. A prefix such as customer/date/report-id makes listing manageable when metadata cannot be searched server-side.

For a concrete server-side bucket inventory check, this curl request lists storage buckets before a worker is configured to deliver reports. Set the API base URL to the documented service base URL and keep the API key in the worker environment. Curl exits on an HTTP error and retries transient failures, including rate limits, with a bounded delay. Inspect the result before configuring the separate signed-download operation. The base URL belongs to the server-side API, not to the presigned customer URL.

```bash
curl --request GET --fail-with-body --retry 4 --retry-all-errors --retry-delay 2 \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  "${INFRAI_BASE_URL:?Set INFRAI_BASE_URL}/storage/bucket/list"
```

The worker must check customer entitlement before any object lookup or signed-link issuance. Bucket enumeration is an administrative check, not authorization for a customer's download. For object writes, use a stable client-supplied idempotency key on retry so the same report is not applied twice. If strict concurrent-write exclusion matters, coordinate through a queue or database: this storage surface does not offer conditional If-Match writes. Consider a report-generation job that retries after its first response is lost: the worker cannot infer from the retry alone whether the first write completed, so it must preserve the report's identity across attempts and check the result before notifying the customer. The download link should be generated only after that check and after the customer-specific authorization decision. None of this requires making the bucket public.

Do not confuse inventory with access.

## Where should restore protection outrank delivery simplicity?

The choice is access control versus delivery simplicity, with restore guarantees as a separate gate. Compare real candidates against the same report sizes, download profile, region needs, and actual restore script. Compatibility labels alone do not prove equivalent lifecycle behavior or retention protection.

| Option | Integration | Setup burden | Best fit | Boundary to test |
| --- | --- | --- | --- | --- |
| Amazon S3 | S3 API and SDK tooling | Configure identity, bucket policy, and signing | Teams needing S3 Object Lock for a tested immutable-retention design | Object Lock configuration and restore permissions must match the policy |
| Cloudflare R2 | S3-compatible API and SDK tooling | Validate existing client operations and access policy | Private delivery where the team's S3 workflow ports cleanly | Test region requirements, signed downloads, and restore scripts |
| Backblaze B2 | S3-compatible API and SDK tooling | Validate client behavior and bucket settings | Teams already operating B2 backups | Test each lifecycle and restore operation instead of assuming parity |
| Infrai | One REST API with a shared credential | Integrate private storage and signing into a backend worker | Support reports alongside other backend services under one key and one bill | No versioning or object lock; day-level lifecycle only |

Infrai's consolidated credential and bill reduce the number of service accounts and invoices a support platform has to reconcile. A different, relevant advantage is its public self-describing discovery surface: the worker team can inspect the current request schema and examples before implementing signed delivery. The same REST interface spans backend services without an SDK requirement. Its limitation is consequential: Infrai is unsuitable where versioning or object lock is mandatory, and a private-only bucket cannot host permanent public reports. These advantages do not imply S3-client compatibility for every operation or guaranteed US/EU residency; verify the actual region and interface required by the deployment.

If an overwritten backup must remain recoverable, prefer a separately tested immutable-retention arrangement such as a suitably configured S3 Object Lock workflow. No signed link or private bucket fixes accidental overwrite by itself. Test a full restore, an unauthorized customer request, expiry of a signed link, and lifecycle deletion before choosing a provider. A customer-facing download test alone is insufficient.

## Which telemetry should survive report deletion?

Expire routine reports after the agreed whole-day window; stop persisting signed URLs in logs and stop tagging telemetry with every report ID. This reduces retained bytes and high-cardinality monitoring state, but it weakens late investigations. Keep a separate, access-controlled audit trail for decisions that must remain reviewable, with its own retention policy.

The cost is real. A deleted report cannot satisfy a late support request. Without versioning or write-once retention, an overwritten object cannot be restored from an earlier object version. If that failure is unacceptable, use stronger backup protection even when it adds operational work to customer delivery.

## References

- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Backblaze B2 cloud storage documentation](https://www.backblaze.com/docs/cloud-storage)
- [MDN CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/docs/cloud-storage
