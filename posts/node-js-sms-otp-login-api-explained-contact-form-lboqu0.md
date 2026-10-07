# Node.js SMS OTP Login API Explained (Contact-Form Evidence and Cooldowns)

A developer-tools contact form has an unusual constraint: the system must route a request to the correct support queue while preserving enough evidence to show that a sensitive submission came from a verified session. The right design is a small server-side state machine. It issues a short-lived SMS code, enforces resend and verification limits independently, consumes a successful challenge once, and records decisions rather than secrets.

**TL;DR:** bind one challenge to a normalized phone identifier, purpose, and opaque session; make resend a transition on that challenge rather than a fresh unlimited action; hash the code; and keep a compact audit event with an explicit retention deadline. A `429` should not reveal whether an account exists. Queue routing happens only after verification, and the routing record refers to the verification event without copying the phone number or code into every log.

## How should a Node.js SMS OTP login API enforce cooldowns?

Start with the claim, not the message provider. For this contact form, the useful claim is narrow: at a stated time, the service accepted a valid code for a particular challenge and then permitted a submission to enter, for example, the security-support queue. SMS does not prove who legally owns a phone number, and the archived NIST SP 800-63B guidance treats use of the public switched telephone network as a restricted authenticator. The evidence model should not inflate that signal into identity proofing.

The minimum linkage is `submission_id -> verification_event_id -> challenge_id`. The verification event can hold a pseudonymous subject key, purpose (`support_contact`), outcome, policy version, timestamps, and coarse delivery channel. It should not hold the plaintext code. Nor does a delivery receipt establish successful authentication; delivery and verification are separate events with separate meanings.

That boundary matters.

This distinction changes support routing. The form payload determines the queue from controlled fields such as product area and issue class. The verified event gates submission of sensitive categories. A free-text phrase never selects a privileged queue by itself.

NIST's cited guidance supplies several useful boundaries: a verifier-generated out-of-band secret must contain at least six decimal digits or equivalent entropy, is accepted only once, must be completed within ten minutes, and requires rate limiting for failed attempts. Ten minutes is a ceiling in that guidance, not a default recommendation. A service may choose a shorter validity period after testing delivery latency and recovery behavior.

This design has limitations. SMS depends on telephone-network availability and is a restricted authenticator under the cited NIST guidance, so it is not suitable when the required assurance level excludes restricted authenticators. A phishing-resistant authenticator is the better design for privileged administration. For a support contact form, SMS can provide a bounded possession signal, but it cannot establish legal identity, guarantee that one person exclusively controls a number, or make the form payload trustworthy. Those trade-offs must appear in the threat model and the compliance claim.

## A bounded challenge state machine

Model the lifecycle explicitly: `pending`, `verified`, `expired`, or `blocked`. Store `code_digest`, `expires_at`, `resend_available_at`, `send_count`, `failed_attempt_count`, and a monotonically increasing version. An atomic conditional update decides every transition. Without that condition, two parallel requests can both observe an old counter and both send.

Resend cooldown and failed-code limiting solve different problems. The cooldown suppresses repeated outbound work and user hammering; the attempt limit bounds online guessing against an already issued secret. Add a broader limiter keyed by a privacy-preserving subject key and another by network source. Neither key is sufficient alone: subject-only controls invite distributed abuse against one person, while source-only controls punish shared office or carrier networks.

For a concrete starting policy, consider a 60-second resend cooldown, three sends per challenge, five verification failures, and a five-minute code lifetime. Those are example configuration values, not values mandated by NIST. Record the policy version so reviewers can reconstruct which thresholds applied. Measure completion and abuse signals before changing them.

Keep responses deliberately plain. A request for a registered and an unregistered number should have the same public shape and similar processing path. Internally, the service can record `send_suppressed`, `challenge_expired`, or `attempt_limit_reached`; publicly, it can return an opaque challenge handle and a generic status. This limits account discovery without erasing operational detail.

One trap is easy to miss: if resend creates a new live challenge but leaves the old one valid, the effective guessing budget multiplies. Invalidate the earlier code atomically or retain one challenge and rotate its digest and version. Short rule. One subject, one purpose, one active challenge.

## The smallest useful API exchange

The following calls demonstrate the boundary rather than a framework. The client supplies an idempotency key for the initial request, retains only the opaque challenge identifier, and never decides whether an attempt is valid. Authentication state stays on the server.

```bash
curl --request POST https://auth.example.test \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: 018f-contact-security-01' \
  --data '{"action":"create_challenge","phone":"+12025550123","purpose":"support_contact"}'

curl --request POST https://auth.example.test \
  --header 'Content-Type: application/json' \
  --data '{"action":"resend","challenge_id":"ch_7Vf3","purpose":"support_contact"}'

curl --request POST https://auth.example.test \
  --header 'Content-Type: application/json' \
  --data '{"action":"verify","challenge_id":"ch_7Vf3","code":"482731","submission_id":"sub_c81a"}'
```

The create operation should return a stable generic response when repeated with the same idempotency key. The resend operation performs an atomic check of cooldown, expiry, send budget, and challenge version before dispatch. Verification compares a digest in constant time, increments the failure count on a mismatch, and changes `pending` to `verified` on success. A second success cannot occur because the state transition is conditional.

The API should use `429 Too Many Requests` for an enforced request limit and may include `Retry-After`; clients must still treat the server as authoritative. Do not turn the countdown shown in a browser into the control. Two tabs, a stale clock, or a direct request bypasses it immediately.

## Count cardinality before collecting telemetry

Compliance evidence and diagnostic telemetry have different retention needs. Put them in different datasets. An audit event needs stable fields for reconstruction; a delivery trace needs enough detail to diagnose latency and failures; an application log needs neither a phone number nor the submitted message body.

| Dataset | Useful fields | Avoid as labels | Retention decision |
| --- | --- | --- | --- |
| Verification audit | event ID, challenge ID, pseudonymous subject, purpose, outcome, policy version, timestamp | phone, code, free text | Set from the evidence obligation and deletion policy |
| Delivery operations | channel, coarse region, provider status class, latency bucket | message ID, phone, exact error text | Short window for incident analysis |
| Metrics | outcome, route class, policy version | challenge ID, submission ID, subject key | Aggregate; keep only the window used for trends |

Cardinality is a budget. Suppose a metrics label uses `challenge_id`: 2 million monthly challenges can create roughly 2 million label values before combinations with outcome, route, and region. That number is illustrative arithmetic, not a measured workload. Event identifiers belong in an indexed audit record or sampled trace, not in metric dimensions.

Retention math should be equally explicit. At 2 million challenges per month and two 600-byte audit events per challenge, raw event payloads are about 2.4 GB per month before indexes, replicas, and storage overhead. Writing the multiplication down prevents a vague compliance requirement from quietly becoming indefinite retention. Sampling can reduce diagnostic traces, but it must not randomly discard records that the evidence policy says are mandatory. Sample debug detail; preserve required decisions.

Redaction needs a test, not a promise. Assert that codes, full phone numbers, authorization headers, and form free text do not appear in logs. Also test the awkward paths: malformed JSON, cooldown denial, expired challenges, duplicate verification, downstream timeout, and queue rejection. Error serialization is where secrets often escape because generic middleware captures the request body.

No exceptions.

## Roll out without weakening the gate

Begin in shadow mode by computing the proposed decision while the existing gate remains authoritative. Compare aggregate outcomes, not raw secrets. Then enable issuance for an internal cohort, exercise duplicate requests and concurrent resends, and verify that one atomic transition wins. Expand by route class only after dashboards separate delivery, verification, throttling, and queueing failures.

Before general release, rehearse key rotation for code hashing, deletion at the retention deadline, and recovery when SMS is unavailable. Recovery must use a separately assessed authenticator or support process; raising attempt limits is not recovery. The final operational check is compact: one active challenge, one consumed success, bounded retries, no secret-bearing logs, and a submission record that points to durable evidence.

That is enough evidence to explain a routing decision without retaining the conversation forever.

## Sources

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management (archived): https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 6585, Additional HTTP Status Codes (`429 Too Many Requests`): https://www.rfc-editor.org/rfc/rfc6585.html
- RFC 7231, Hypertext Transfer Protocol Semantics and Content (`Retry-After`): https://www.rfc-editor.org/rfc/rfc7231.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
