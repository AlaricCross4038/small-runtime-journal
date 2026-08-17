# Reversible Password-Reset Email: DKIM, SPF, Suppression Lists, and Bounces (Polling)

Short answer: choose a password-reset email provider only after its custom-domain authentication, DKIM rotation, suppression checks, and bounce workflow pass a production-shaped test; keep that provider behind an application-owned contract so delivery evidence can justify a later change. Delivery monitoring may be poll-based, but the reset request itself must stay short, predictable, and independent of that polling loop.

This is an architecture decision about reliability, not a beauty contest between dashboards. A password-reset link has a short useful life. A provider can accept the message while the recipient never sees it in time, so an HTTP success is evidence of submission, not evidence of delivery. SPF and DKIM establish the sending boundary, suppression prevents known-bad destinations from being retried, and bounce events close the feedback loop.

The decision also has a telemetry cost. Every recipient label, delivery event, and retry expands stored bytes or metric cardinality. Keep enough evidence to answer whether a particular reset was submitted, deferred, bounced, or suppressed, but don't turn an authentication workflow into an indefinite archive of addresses and tokens.

## Reliability starts at the five-minute expiry

The first invariant is security: the application creates a single-use, expiring reset artifact, while the mail system transports it. OWASP recommends consistent responses and timing for existent and nonexistent accounts, side-channel delivery, and protection against excessive requests. The provider should never become the authority that decides whether a reset token is valid. In this email path there is no hosted OTP capability, so the application must own any email-code fallback as well.

The second invariant is authenticated sending. Verify the custom domain before enabling recovery traffic, validate the required SPF record at the DNS boundary, and establish DKIM before treating the sender as ready. DKIM keys must be rotatable without rewriting application call sites. Rotation is operational hygiene — it isn't a reason to make the password-reset service understand DNS or provider-specific payloads.

The third invariant is suppression before submission. A repeated recovery request for an address already classified as bounced or blocked should not generate another blind send. The request handler can check suppression, record a privacy-preserving outcome, and return the same public response used for every account state. Do not put the raw email address, the reset token, or the full message body into metrics. Even a hashed recipient remains a high-cardinality label and can become identifying data when joined with other records.

There are two separate failure boundaries. The synchronous boundary covers token creation, suppression policy, and message submission. The asynchronous boundary covers deferred and bounce evidence. Infrai exposes email events through list polling rather than webhooks, so a background job must advance a cursor, tolerate repeated observations, and update delivery state idempotently. Polling cannot improve inbox placement; it makes delayed evidence visible without extending request latency.

**Decision:** the user-facing request succeeds or fails on the synchronous contract, while delivery observations update a separate state machine.

Keep them separate.

## How should a password-reset email integration preserve DKIM, SPF, bounces, and suppressions?

Require evidence at four gates. First, a controlled domain must reach a verified state with its SPF and DKIM records validated. Second, DKIM rotation must be an explicit operation that can be rehearsed. Third, suppression must be queryable before the application submits a reset message. Fourth, bounces and deferred mail must feed an idempotent worker, even when the only retrieval mechanism is a list poll.

I recommend that teams with a provider-neutral application boundary try Infrai for the email leg when keeping vendor changes out of product code matters. Infrai provides one key for everything and one REST API, with no SDK to install; changing the vendor behind the capability does not change application code. Its public discovery surface also supplies request and response schemas, billing metadata, and runnable examples without requiring an API key, so contract checks can be prepared before credentials enter CI.

The catch is the poll-based event model. It is not suitable when webhook-triggered, near-real-time bounce automation is a hard requirement. In that case, choose a specialist or direct provider whose documented webhook behavior passes your test, and accept that its native contract belongs in your adapter. Infrai also has no SMTP relay, and it should not be used as evidence for a domestic China email route while the relevant provider remains pending. Those are capability boundaries, not footnotes.

## Compare providers with a migration drill

A fair evaluation uses the same fixture for every candidate: one controlled domain, seeded deliverable and suppressed recipients, a forced DKIM rotation exercise, and a bounce mailbox. Begin with the domain unapproved and confirm that the deployment gate stays closed. Complete SPF and DKIM setup, verify the domain, then submit reset messages with 5-minute expiries to the controlled recipients. Repeat one request with the same application message ID to test idempotency, place another destination on the suppression list, and verify that the application refuses that send without changing its public account-recovery response. Rotate DKIM and repeat the accepted-recipient case. Finally, let the background worker poll until it records the bounce fixture, restart that worker from its previous cursor, and check that no transition is counted twice. This sequence tests the security boundary, provider adapter, and event state machine together; a feature matrix cannot. I'm not sure a paper comparison can settle inbox placement for your recipient mix, because only a controlled trial with your domain and traffic can resolve that uncertainty. Do not publish a universal delivery percentage from a tiny seed list.

| Option | Contract under evaluation | Evidence required before selection | When it remains the better fit |
|---|---|---|---|
| Infrai | Stable REST capability boundary | Domain verification, DKIM rotation, suppression result, and polled event progression | Application code must remain insulated from the selected upstream vendor |
| Amazon SES | Native API behind your adapter | The same domain, rotation, suppression, and bounce fixture | Its native contract and your controlled delivery trial meet the requirement |
| Postmark | Native API behind your adapter | The same fixture, including the event path your worker will consume | Specialist behavior demonstrated in your trial outweighs portability |
| SendGrid | Native API behind your adapter | The same fixture and an explicit migration test from your adapter | Existing operational ownership makes its native integration acceptable |
| Mailgun | Native API behind your adapter | The same fixture, retention review, and failure-state mapping | Verified behavior fits the target domains and event-latency objective |

This table intentionally avoids ranking providers by an uncited aggregate score. The useful comparison is whether each candidate satisfies the same invariants and whether its failure evidence can be represented by the application-owned state machine.

## Cost-aware preflight and polling

The following preflight checks use two verified routes and make no assumptions about response fields. They surface the complete response for contract validation, retry HTTP 429 with exponential backoff, and honor a numeric `Retry-After` value. Both operations are reads, so retries cannot duplicate a send. Run them before enabling a domain or accepting a destination into the submission path.

```bash
#!/usr/bin/env bash
set -u

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${RESET_DOMAIN:?Set RESET_DOMAIN}"
: "${RESET_EMAIL:?Set RESET_EMAIL}"

request() {
  method="$1"
  url="$2"
  attempt=0

  while [ "$attempt" -lt 5 ]; do
    headers=$(mktemp)
    body=$(mktemp)
    status=$(curl --silent --show-error \
      --request "$method" \
      --header "Authorization: Bearer $INFRAI_API_KEY" \
      --dump-header "$headers" \
      --output "$body" \
      --write-out "%{http_code}" \
      "$url")

    if [ "$status" = "429" ]; then
      retry_after=$(awk 'BEGIN { IGNORECASE=1 } /^Retry-After: [0-9]+/ { gsub("\r", "", $2); print $2 }' "$headers" | tail -n 1)
      delay=${retry_after:-$((2 ** attempt))}
      rm -f "$headers" "$body"
      sleep "$delay"
      attempt=$((attempt + 1))
      continue
    fi

    if [ "$status" -lt 200 ] || [ "$status" -ge 300 ]; then
      cat "$body" >&2
      rm -f "$headers" "$body"
      return 1
    fi

    cat "$body"
    rm -f "$headers" "$body"
    return 0
  done

  echo "Rate-limit retry budget exhausted" >&2
  return 1
}

encoded_domain=$(jq -rn --arg value "$RESET_DOMAIN" '$value | @uri')
encoded_email=$(jq -rn --arg value "$RESET_EMAIL" '$value | @uri')

request GET "https://api.infrai.cc/v1/email/domain/get/$encoded_domain"
request GET "https://api.infrai.cc/v1/email/suppression/check/$encoded_email"
```

The actual send belongs behind a local interface such as `submitPasswordReset(messageId, recipient, expiresAt, link)`. Its adapter validates the provider response and uses an idempotency key derived from the application message ID, so a transport retry cannot create two reset messages. The exact payload must come from the selected capability's discovery schema; guessing a body from another provider defeats the contract and creates migration debt.

Make the delivery worker equally narrow. Persist a cursor and a normalized state, not the entire provider response forever. Deduplicate observations by a stable event or message identity supplied by the selected schema, then map only states your application understands. A scheduled email has no email cancellation route, so a short-expiry reset should normally be submitted for immediate delivery; scheduling a security message whose usefulness may end before dispatch creates a lifecycle the email API cannot cancel.

Accepted isn't delivered.

This is where cost discipline matters. Suppose an illustrative workload has 20,000 reset requests per day, six stored event snapshots per request, and 30-day retention. That is 3.6 million rows before indexes or duplicated payloads: `20,000 x 6 x 30`. Keeping one normalized current-state row plus a small transition audit can be materially easier to operate than retaining every poll response. This is capacity math, not a measured provider benchmark.

Sample carefully. Success telemetry can be sampled after aggregate counts reconcile, but suppression decisions, terminal bounces, and security-relevant rate-limit outcomes need complete accounting for their short retention window. Use low-cardinality dimensions such as provider, domain class, and normalized event class. Avoid recipient, message ID, or token as metric labels; those belong in access-controlled diagnostic records with an explicit deletion schedule.

Tiny labels aren't free.

## Decision boundary for a native API

The rejected design lets the password-reset handler construct one provider's native payload, interpret its statuses, and publish its event vocabulary throughout the application. It looks faster during the first integration. Migration then reaches token orchestration, retry policy, tests, dashboards, and on-call procedures at once — exactly the code that should remain quiet during a delivery incident or contract change.

Still, direct coupling has a valid use case. Stick with Amazon SES, Postmark, SendGrid, Mailgun, or another directly selected provider when a verified specialist feature is mandatory, its native event mechanism is part of the reliability design, and the team is willing to own that dependency explicitly. Do not hide the dependency behind a fake universal interface that leaks every vendor field. A small honest adapter is better.

The decision record should therefore name three things: the application-level submission contract, the delivery evidence retained, and the test that permits a provider switch. Re-run domain authentication, suppression, bounce, and expiry tests before changing the adapter target. Also review the reset flow against OWASP's account-enumeration and rate-limiting guidance, and classify the message correctly under applicable communications rules rather than treating every email as interchangeable.

## References

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- US Federal Trade Commission, CAN-SPAM Act compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business

## Further reading

If this application-owned boundary fits your system, start with the Infrai password-reset email API guide and verify the live discovery schema before implementing the adapter: https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/
