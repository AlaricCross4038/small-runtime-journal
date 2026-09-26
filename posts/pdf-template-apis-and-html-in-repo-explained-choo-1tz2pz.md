# PDF Template APIs and HTML-in-Repo Explained (Choosing Invoice Layout Ownership)

**Short answer:** put an invoice layout in the repository when engineers own its release, tests, and rollback; use a template-management API when operations or design staff must publish layout changes without an application deployment. For a marketplace that watermarks invoices before sharing them with sellers or buyers, keep the watermark policy outside either template. The rendering boundary should receive an already authorized, immutable document request.

That decision rule is more durable than a feature comparison. HTML versus a managed PDF template is mostly an ownership question disguised as a rendering question. Both can produce a PDF, and PDF itself has a standardized model under ISO 32000-2. The material difference is who may change the presentation, how that change becomes a release, and which system can prove what happened later.

My architecture decision record therefore starts with one invariant: the same accepted invoice revision, template revision, locale, and watermark policy revision must identify the same intended output. Byte-for-byte equality is a separate goal and may depend on the renderer. The business requirement is traceable intent.

Ownership comes first.

## Should invoice PDF templates use an API or HTML in a repo?

An invoice is not just a page. It is a financial record crossing a trust boundary. Before external sharing, the marketplace must decide which party may receive it, which fields may be visible, and whether the copy needs a watermark such as a recipient identifier or distribution status. A template must not make those authorization decisions. If it does, a designer editing presentation can accidentally edit policy.

The critical failure boundaries are easier to reason about when stated explicitly:

1. The application validates invoice data and sharing authorization.
2. A policy component resolves the watermark instruction from stable inputs.
3. The rendering service combines data, layout revision, fonts, locale, and watermark instruction.
4. Object storage accepts the completed artifact only after rendering succeeds.
5. The audit event records identifiers and outcomes, not invoice contents.

No partial artifact should become shareable. A timeout is not proof that rendering failed, so retries need an idempotency key tied to the invoice revision and render intent. The response should distinguish an accepted render from a completed artifact; HTTP semantics provide the vocabulary, while the application contract defines the state transition.

This separation also contains a subtle ownership problem. If a watermark is embedded in every layout, changing the sharing rule requires editing every template. If it is a post-processing step with an explicit policy revision, invoice designers cannot omit it, and the same rule can cover a newly added locale without copying presentation logic.

Policy stays outside.

## Decision record: compare ownership before syntax

The two choices should be evaluated with the same release controls. “HTML in Git” does not automatically mean tested, and “managed” does not automatically mean uncontrolled. The useful comparison is where the authoritative revision lives and who can advance it.

| Decision dimension | Repository markup | Managed template API |
| --- | --- | --- |
| Authoritative owner | Engineering team and code reviewers | Design or operations team under an explicit publish role |
| Release unit | Application or template bundle commit | Immutable published template revision |
| Review evidence | Pull request, tests, and commit history | Draft-to-publish audit record and approval policy |
| Rollback handle | Prior commit or artifact digest | Prior immutable template revision |
| Local reproducibility | Strong when renderer, fonts, and assets are pinned | Depends on exportability and a faithful preview environment |
| Best fit | Layout changes track application behavior | Layout changes have an independent business cadence |

Choose repository markup when invoice changes usually accompany schema changes, tax logic, or application releases. A single review can then cover the data contract and its presentation. CSS Paged Media defines page-oriented concepts such as page boxes and margin boxes, but renderer support still needs fixture testing; the specification is a contract reference, not evidence that every engine behaves identically.

Choose a managed template boundary when non-engineers genuinely own copy, spacing, brand assets, or locale variants and need a controlled publish workflow. “Genuinely” matters. If every publish still needs an engineer to verify fields and deploy code, the organization has added another control plane without transferring ownership.

The decision can be reversed later only if revisions are portable. Store the input schema, a template revision identifier, required asset digests, and representative fixtures independently of the editing surface. Preserve generated PDFs according to the record-retention policy, but do not pretend that archived output replaces source and release history.

Revisions are the unit of control.

## The critical path should expose revisions, not editor state

The application contract needs fewer degrees of freedom than the template editor. This illustrative request uses a reserved example domain and shows the minimum facts the renderer needs. The layout reference points to an immutable published revision; it does not mean “latest.”

```bash
curl --fail-with-body \
  --request POST \
  --url https://renderer.example/v1/pdf/generate \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: inv_8417-r3-layout_27-policy_6' \
  --data '{
    "document_type": "marketplace_invoice",
    "invoice_revision": "inv_8417-r3",
    "layout_revision": "layout_27",
    "locale": "en-US",
    "watermark_policy_revision": "policy_6",
    "recipient_class": "seller",
    "output": "pdf"
  }'
```

The renderer should reject missing revisions before work begins. It should also fetch invoice values from an authenticated source or accept them through a protected channel; putting customer data in logs for convenience creates a second, poorly governed copy of the invoice.

Test at three layers. Schema tests catch a renamed amount or address field. Structural PDF checks verify page count, required text, and metadata without depending on pixel identity. A small set of visual fixtures catches clipping, font substitution, page-break movement, and watermark collisions. The fixture set should include the awkward cases: a long legal name, the largest supported line-item count, a multi-page address, and every writing direction the product promises to support.

Deployment then becomes promotion of a tuple: renderer build, layout revision, font bundle digest, and watermark policy revision. Roll back the tuple, not one element in isolation. Otherwise a template rollback can meet a newer schema and fail in a way neither revision saw during review.

## How much telemetry does this decision really need?

Observability should answer release and failure questions without recreating the invoice database. Start with bounded dimensions: document type, outcome, renderer build, layout revision, watermark policy revision, locale from a controlled set, and a coarse failure class. Do not use invoice ID, buyer ID, seller ID, file name, or idempotency key as metric labels. Those belong in access-controlled traces or audit records when investigation requires them.

Count cardinality before shipping a label. Suppose the controlled dimensions have 2 document types, 4 outcomes, 3 active renderer builds, 12 active layout revisions, 6 locales, and 5 failure classes. Their theoretical cross-product is `2 x 4 x 3 x 12 x 6 x 5 = 8,640` metric series before environment, region, histogram buckets, or replicas. This is planning arithmetic, not a measured workload. It shows why a seemingly harmless revision label can multiply storage and query work.

Retention deserves the same arithmetic. If those 8,640 series emit one sample per minute, that is 12,441,600 samples per day. Keeping 30 days means 373,248,000 samples before compression and metadata overhead. The exact byte cost depends on the telemetry backend, so claiming a universal storage figure would be false. The useful move is to decide which questions require minute-level data for 30 days and which can survive as hourly rollups.

Keep audit records longer when regulation or business policy requires them, but keep their payload narrow: who requested sharing, which authorized subject received access, the revision tuple, timestamps, and the outcome. Logs can retain a shorter diagnostic window. Metrics can be aggregated. These are different data products with different access and retention rules.

Sampling also has a specific trade-off. Successful traces may be sampled after enough time has passed to characterize ordinary latency, while failures and policy rejections deserve complete capture within privacy constraints. Head sampling can miss rare slow or failed renders because the decision occurs before the outcome is known. Tail sampling sees the outcome but requires buffering and operational capacity. Neither choice licenses recording invoice contents.

Do the multiplication first.

Less is deliberate. A dashboard that can isolate `layout_27` failures after a promotion is useful; a label per invoice is an expensive index for a question better answered by a secured audit lookup.

## The rejected option still has a valid use case

For the marketplace described here, I would reject independently editable managed templates if engineers remain accountable for every invoice schema change and every release. Repository-owned markup keeps review, fixtures, and rollback in the same change unit, while the watermark stays in a policy-controlled rendering stage. This is not a universal preference. It follows the ownership boundary.

The rejected option becomes valid when a brand or localization team has real publishing authority, layout changes outnumber application changes, and the system can enforce immutable revisions, approvals, previews, export, and rollback. At that point, forcing each wording or spacing change through an application deployment makes engineering the accidental owner of editorial work.

The reverse is also true. A repository is a poor answer when it merely hides unreviewed templates inside application code. Whichever boundary wins should have one named owner, immutable release identifiers, representative fixtures, and an audit trail. The rendering technology is secondary to those controls.

The final rule is compact: place the template where its accountable editor can safely release it, and keep authorization plus watermark policy outside the layout. Measure revisions with bounded labels, retain detail only as long as a defined question needs it, and make every externally shared invoice traceable to an immutable decision tuple.

## References

- ISO, “ISO 32000-2 — Portable Document Format”: https://www.iso.org/standard/75839.html
- W3C, “CSS Paged Media Module Level 3”: https://www.w3.org/TR/css-page-3/
- IETF, “RFC 9110: HTTP Semantics”: https://www.rfc-editor.org/rfc/rfc9110
- OpenTelemetry, “Metrics Data Model”: https://opentelemetry.io/docs/specs/otel/metrics/data-model/
- OpenTelemetry, “Trace SDK: Sampling”: https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling
- OWASP, “Logging Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
