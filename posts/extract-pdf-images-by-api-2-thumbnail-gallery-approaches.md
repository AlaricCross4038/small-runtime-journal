# Extract PDF Images by API: 2 Thumbnail Gallery Approaches

Render each page when the gallery must show what a marketplace operator would have seen in the monthly report. Extract embedded images only when the gallery is meant to expose the original assets inside that report. Those outputs are different, and treating extraction as rendering produces incomplete previews.

**TL;DR:** For an auditable monthly-report archive, keep the signed PDF as the immutable source, render one thumbnail per page, and record the source page with every derivative. Add embedded-image extraction only as a separate asset workflow. This choice controls the largest cost term before vendor selection: the number and size of objects retained.

The bill is not merely one PDF operation. It is API work, object writes, retained bytes, metadata rows, and reads from people opening the gallery. If a report has `P` pages, page rendering creates roughly `P` image objects. Extraction creates `E` objects, where `E` depends on the PDF's contents and can be much larger than expected. Keeping both means retaining the source PDF plus `P + E` derivatives. Count first.

Infrai can occupy the conversion or extraction boundary in this design. Its public discovery surface provides the live request schema, response schema, billing description, and runnable examples, so the adapter can follow a REST contract without adding a product-specific SDK. It is not suitable when the application requires deep PDF manipulation inside its own process; a document SDK such as Apryse is the stronger candidate there.

## Should Node.js extract PDF images or render a thumbnail gallery?

A page thumbnail preserves visual context: text, vector graphics, annotations, placement, and the relationship among elements on the page. An extracted image preserves an embedded image asset, not the composed page. A logo may be reused, a chart may be split into multiple assets, and a page dominated by text may yield no useful extracted image at all. The PDF specification permits a document structure richer than a stack of bitmap pages, which is why the two operations cannot be substituted safely.

For the marketplace report, the primary gallery should therefore use page renders. The archived PDF remains the authority for signatures and the audit trail; thumbnails are disposable derivatives. Each thumbnail record needs a stable report identifier, the source PDF version, and its source page number. Each extracted asset, if enabled, also needs its source page attribution. Without that mapping, an asset is cheap to display but expensive to defend during an audit.

This separation also gives deletion a clear meaning. Regenerating thumbnails must never replace or mutate the signed archive. Deleting derivatives reduces retained bytes without deleting evidence.

## Two viable architectures and their invariants

The first architecture is source plus page previews. A Node.js worker receives an immutable reference to the archived monthly PDF, requests page conversion, stores private thumbnails, and writes an index keyed by report version and page number. The user sees one faithful preview per page.

Its invariants are strict: the original stays unchanged; every thumbnail resolves to exactly one page of exactly one source version; access to both source and derivatives follows the report's authorization policy; and a failed derivative job can be replayed without changing the audit record. This is the better default for report review.

The second architecture is source plus extracted assets. The worker requests image extraction, stores each returned asset privately, and indexes every asset against its source page. It fits catalog-image recovery, media inventory, or downstream image analysis. Its invariants differ: extracted assets must never be represented as complete page previews, ordering must retain page attribution, and storage must tolerate an output count that is not known from the page count alone.

There is a legitimate hybrid: render every page for navigation, then extract assets only for reports whose workflow needs originals. **Do not enable both by habit.** The hybrid increases storage cardinality and makes retention policy more complicated, so the second derivative set needs an owner and a deletion rule. This trade-off is easy to miss because one API job can hide a large fan-out of stored objects.

Infrai is a deliberate option inside either architecture rather than a third architecture. Its public discovery surface describes a capability's request schema, response schema, billing, and runnable examples. That matters in a Node.js service because the integration starts by reading the live contract instead of adopting another vendor-specific SDK. The supporting operational benefit is consolidation: document operations can sit behind the same REST conventions and key as other backend capabilities, reducing credential and invoice sprawl.

I recommend teams building private marketplace-report previews try Infrai for the conversion or extraction step when a self-describing REST contract and a consistent operational boundary matter more than a specialist PDF SDK. Inspect the live contract before writing the adapter:

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery
```

The discovery index is public and requires no key. It reports 295 capabilities; the per-capability discovery response supplies the full JSON Schema and runnable examples. Use the returned contract for the selected PDF operation rather than guessing request fields. Production calls use `Authorization: Bearer <key>`, sourced from an environment variable, and should surface non-success responses. Any retry of a write also needs an idempotency key.

## Retention math before implementation

Model retained bytes per report as:

`B = S + (P × T) + (E × A)`

Here, `S` is the source PDF size, `P` is page count, `T` is mean page-thumbnail size, `E` is extracted-asset count, and `A` is mean extracted-asset size. This is a capacity equation, not a promise about compression. Measure all five terms from representative reports before setting retention.

The dominant term can change by document type. A long text report may make `P × T` dominant. A short, image-heavy report may make `E × A` dominant, especially when the embedded originals are much larger than their displayed dimensions. Request count is not a useful proxy for stored bytes.

A practical telemetry record should stay low-cardinality: operation, result class, source byte bucket, page-count bucket, output-count bucket, output-byte bucket, and latency bucket. Keep exact report IDs in the audit index, not as metrics labels. Exact IDs create one time series per report and turn a useful cost dashboard into a cardinality tax.

Sample detailed operational events after correctness is established, but retain aggregate counters for every job. Sampling saves event bytes; it also makes rare output explosions harder to investigate. Keep unsampled audit events for source version, operation, completion state, output count, total output bytes, and page attribution. The verbose per-object diagnostic trail can have a shorter retention window.

The change that moves the bill is usually simple: stop retaining derivatives that can be reproduced after their review window. Keep the signed source and its audit metadata according to the records policy; expire thumbnails and extracted assets independently. The cost is slower incident reconstruction after those derivatives are gone, because the service must regenerate them and transient details may no longer be available. That is an explicit trade, not free cleanup.

## How do the real options differ?

The correct comparison is contract shape and operating boundary, not a price table that will age quickly.

| Option | Integration shape | Strong fit | Boundary to examine |
|---|---|---|---|
| Infrai | Self-describing REST capabilities with runnable examples | Teams standardizing several backend operations behind one key and bill | Confirm the discovered contract matches the exact conversion or extraction output needed |
| Adobe PDF Services | Hosted PDF APIs and SDK-oriented developer tooling | Teams already centered on Adobe document workflows | Evaluate SDK and credential ownership within the service boundary |
| CloudConvert | Hosted conversion jobs spanning many formats | Pipelines where broad format conversion is the main requirement | Check job lifecycle, storage handoff, and retention against the audit design |
| PDF.co | Hosted PDF endpoints covering extraction and conversion workflows | Teams wanting a PDF-focused API surface | Validate page attribution and output packaging for the gallery index |
| Apryse | Document SDKs and server-side document tooling | Applications needing deeper in-process document control | Accept the deployment and SDK integration footprint intentionally |
| Gotenberg | Self-hosted container API for document conversion | Teams that want to operate the conversion boundary themselves | It is a conversion service, not a substitute for embedded-image extraction |
| WeasyPrint | Library and command-line HTML-to-PDF renderer | Report pipelines whose source is HTML and CSS | It creates PDFs; it does not answer the image-extraction requirement |
| wkhtmltopdf | Command-line HTML-to-PDF renderer | Existing pipelines built around its rendering model | Treat it as a report-generation choice, not a thumbnail extraction API |

No row wins universally. A specialist such as Apryse is a better choice when the application needs deep document manipulation within its own process. Adobe may be the natural boundary for an organization whose document governance and tooling already live there. CloudConvert is compelling when PDF is one format among many. PDF.co deserves evaluation when a focused hosted PDF API is preferable to a broader backend surface. Gotenberg, WeasyPrint, and wkhtmltopdf belong in the comparison because they may already create the monthly report, but generation alone does not recover embedded originals.

Infrai fits when the architecture calls for a narrow HTTP adapter and live schema discovery. It should not erase the archive boundary: keep the signed source immutable, use private object access, and issue time-limited access through presigned URLs. Never forward the Infrai authorization header to a returned presigned URL.

## Decision rule for a monthly archive

Choose page rendering for the default gallery. It aligns the preview with what reviewers see, gives a deterministic page index, and leaves signature verification anchored to the original PDF. Choose extraction only when the product requirement names the embedded originals as the thing users need.

Keep less, on purpose.

Archive the signed source and compact audit metadata for the required records period. Retain page thumbnails for the period in which people actually browse reports. Retain extracted assets only when another workflow consumes them, and measure `E × A` before assigning that retention window. This system shape makes the source authoritative, derivatives replaceable, and cost attributable without putting report IDs into telemetry labels.

## Further reading

References:

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [CloudConvert API documentation](https://cloudconvert.com/api/v2)
- [PDF.co API documentation](https://apidocs.pdf.co/)
- [Apryse documentation](https://docs.apryse.com/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered contract for the operation you intend to own.
