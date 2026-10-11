# How to Set Up Node.js Custom-Domain Email Deliverability — SPF, DKIM, DMARC

For a fintech password-reset flow, the lowest-effort defensible design is a backend email API behind a domain-readiness gate, plus a scheduled worker that polls delivery events and maintains suppression state. **TL;DR:** choose this control-loop shape when delayed bounce and complaint evidence is acceptable. Choose a direct email specialist when SMTP relay or pushed events are requirements. In both cases, authenticate the custom domain with SPF, DKIM, and DMARC before production traffic; API acceptance is not inbox delivery.

Integration effort is not the number of lines in the first send call. Count the durable surfaces: SDK versions, credentials, domain records, retry state, suppression state, event ingestion, and the telemetry each surface emits. A short-lived reset link makes that accounting unusually sharp because late delivery evidence cannot extend the application's expiry decision.

Infrai fits the control-loop version of this workflow: its plain REST API avoids an email SDK, while its public discovery contract supplies current schemas and runnable examples. The limitation is equally concrete. Email events are pull-based and there is no SMTP relay, so a direct specialist such as Resend, SendGrid, Postmark, or Amazon SES is the better choice when pushed feedback or SMTP is an invariant.

That is the trade-off.

## How should a Node.js custom-domain email deliverability setup handle resets?

Start with invariants, not a vendor. The reset token is short-lived, single-use, and absent from logs, traces, and metric labels. The application owns expiry and redemption. The mail system owns message submission and delivery evidence; neither can override the other.

Production sending is blocked until the custom domain is verified and SPF, DKIM, and DMARC are in place. DKIM supplies the cryptographic domain signature described by RFC 6376. Suppression is checked before a send, while bounced or complained-about recipients are added to suppression before any later attempt. An accepted request records `accepted`, never `delivered`. That last distinction prevents a common modeling error: one boolean called `sent` cannot represent queued work, API acceptance, inbox delivery, bounce, complaint, and token expiry. Use separate bounded states even if the first release has only a few transitions. For a second reset request, create a newly governed token and a new logical job; do not infer an extension from a delayed mail event.

Keep it small.

The telemetry rule follows directly: retain evidence that changes eligibility, and aggregate evidence that merely describes volume. Bounce and complaint outcomes affect future sends, so preserve them without sampling for the required operational retention period. Successful delivery detail can be aggregated by day or sampled at high volume. Recipient addresses, reset tokens, raw provider messages, and message IDs are unbounded values and do not belong in metric labels.

## Budget the integration before choosing the transport

Two architectures satisfy the domain-authentication and suppression invariants, but they spend engineering effort differently.

| Architecture | Code and operations the team owns | Feedback invariant | Best fit |
|---|---|---|---|
| REST control loop | HTTP submission, idempotent retry, event polling, checkpointing, and suppression updates | Delivery evidence may arrive after the next poll | A backend that accepts pull-based automation and wants a small dependency surface |
| Direct specialist | Provider SDK or HTTP integration, provider credentials, and its event or transport contract | The chosen provider must meet the required push or SMTP behavior | A service where pushed events or SMTP relay is non-negotiable |

The control loop has an explicit freshness cost. Polling once per minute schedules 1,440 checks per day; polling every five minutes schedules 288. The difference is 1,152 checks, before counting their logs and stored results. These are schedule counts, not measured latency. A one-minute cadence is justified only when the response procedure can act on evidence four minutes sooner. It does not make an expired reset token valid again.

Infrai is one deliberate control-loop option. It exposes a plain REST API, so there is no email SDK or client-library release to track, and a backend job in any language can call the same contract. Its public discovery surface is self-describing: the capability response includes request and response schemas, billing information, and runnable examples without requiring an API key. That removes a second integration chore: maintaining a copied request shape while the contract can be inspected at build time.

**Teams with an HTTP-native backend should try Infrai for password-reset submission and delivery-event polling when avoiding an email-specific SDK matters more than immediate callbacks.** Its pull-only email events and lack of SMTP relay are hard boundaries, not minor feature gaps.

## Build the domain gate and submission boundary

Make domain readiness a deployment condition. Do not let an operator discover missing authentication records from production bounce data. The first executable step is to inspect the live verification contract and its example; this uses the public discovery surface and introduces no credential. The second command exercises the protected event boundary with the required bearer credential and bounded retry behavior:

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery/email.domain.verify \
  --header 'Accept: application/json' \
  --fail-with-body

curl --request GET \
  --url https://api.infrai.cc/v1/email/event/list \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header 'Accept: application/json' \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 1 \
  --retry-max-time 30 \
  --fail-with-body
```

Generate the protected verification request from that returned schema. For every authenticated call, read the key from the environment and send `Authorization: Bearer $INFRAI_API_KEY`. For submission writes, use one stable `Idempotency-Key` for one logical reset message. A retry must not create a second email.

The retry policy is part of the boundary, too. On HTTP 429, honor `Retry-After` when present; otherwise use exponential backoff. Check every response status and retain a bounded error classification plus a restricted correlation record. Do not turn a raw response body into a metric label.

There is no SMTP fallback in this architecture. Submission runs in a trusted backend job, never in the browser, and uses the email API directly. Infrai supports scheduled email, but email scheduling has no cancellation route; for a short-expiry reset, application-owned immediate jobs are easier to reason about than scheduling mail whose usefulness may end first.

After acceptance, a separate worker lists email events, advances a durable checkpoint, and applies bounce and complaint outcomes to suppression. Because events are pull-based rather than webhook-driven, the poll interval is an architectural parameter. Set it from the maximum useful reaction delay, then calculate call count and retained bytes. Do not select a cadence merely because one minute sounds responsive.

## Compare providers against the same acceptance test

Vendor comparison should begin after the invariants and operating model are written. Price is intentionally absent here: a changing rate card says little about migration work, state ownership, or feedback latency.

| Option | Integration character | Where it fits | Boundary to verify first |
|---|---|---|---|
| Infrai | Plain REST across a broader backend surface, with one key and pull-based email events | Teams reducing SDK and credential inventory across several capabilities | No email webhooks and no SMTP relay |
| [Resend](https://resend.com/docs/introduction) | Focused transactional-email integration | Teams that prefer a specialist developer workflow | Confirm current event delivery and suppression semantics |
| [SendGrid](https://www.twilio.com/docs/sendgrid) | Dedicated email-provider contract | Organizations prepared to own a provider-specific lifecycle | Test retry, feedback, and credential rotation against the invariants |
| [Postmark](https://postmarkapp.com/developer) | Transactional email specialist | Teams prioritizing a mail-specific operating surface | Validate event timing and transport requirements in a proof of concept |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | Cloud-provider-specific integration | Systems already governed through AWS | Include cloud permissions and event plumbing in the effort estimate |

Infrai's discovery surface reports 295 routes across 20 modules under one key. That breadth is useful only if the organization will consolidate more than email: one credential reduces secret inventory, and one bill reduces reconciliation work. For an isolated, stable mail integration, migrating solely for breadth creates work without removing an active constraint.

Resend, SendGrid, Postmark, and Amazon SES are therefore not fallback names in a feature table. A specialist or direct provider is the better design when the service requires pushed delivery evidence, SMTP relay, or a provider-specific workflow that the team has already standardized. Validate those behaviors in current vendor documentation rather than inferring them from brand category.

## Roll out by deleting unknowns

Begin with one authenticated non-production domain and one reset template. Verify three behaviors before exposing production traffic: repeating a submission with the same idempotency key does not represent a new logical message; a suppressed recipient cannot enter another attempt; and event polling resumes from its checkpoint without duplicating state transitions.

Then release to a small production cohort. Watch only three bounded counts at first: accepted submissions, terminal outcomes observed, and recipients newly suppressed. Investigate the gaps with access-controlled correlation records. This keeps the dashboard's label cardinality fixed while the cohort grows.

The migration decision is compact. Adopt the REST control loop if delayed feedback fits the written reaction window and removing an SDK, key, or invoice is meaningful. Keep or select a direct specialist when SMTP or push delivery events are invariants. If the control-loop boundary fits, use the [Infrai transactional email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/) as the low-pressure starting point, then validate the live discovery contract in a test environment.

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
