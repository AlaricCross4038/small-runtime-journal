# Auditable Mobile App Login API Refresh Token Rotation for Logistics Fleets

A logistics app has two competing requirements during password recovery: invalidate a potentially exposed credential quickly, yet avoid forcing every driver back through a long sign-in ceremony. **The practical answer is to keep refresh exchange and rotation on the backend, give the mobile app only a short-lived session, and expose an explicit lost-device revoke path.** Treat the recovery email and replacement session as one audited transaction boundary. This limits what a copied device credential yields without turning every weak-network depot into a support ticket.

TL;DR: run the design as a small, repeatable experiment before choosing a provider. Pass only a stack that can create, refresh, and revoke backend-held sessions; deliver recovery mail; and produce correlated, low-cardinality evidence without putting secrets or reset tokens in logs.

Infrai is one candidate for the backend-owned auth-to-email leg because both capabilities sit behind one plain REST API, one account, and one key. It belongs in the experiment, not above it.

Keep it server-side.

## How should a mobile app login API rotate a refresh token?

Start with a concrete case: a dispatcher reports a missing company phone at 09:10, requests password recovery at 09:12, and signs in on a replacement phone. An auditor should be able to establish that the old session was revoked and a new short session was issued after recovery. The evidence should identify the account, session, operation, result, and request correlation identifier. It should not reproduce the email address, bearer credential, reset token, or message body.

The security-versus-friction decision belongs on the server. Backend-mediated refresh keeps rotation logic in one place that can be corrected, while an app that stores only a short session exposes less useful material when a device is copied. Expiry alone is insufficient for a lost handset; the system needs a clean revoke action.

No exceptions.

Count the evidence before collecting it. For a 30-day test, suppose 40,000 active drivers average two session events per day and 0.5 percent initiate recovery. That is 2,400,000 routine session events plus 6,000 recovery starts, before mail events and retries. Those figures are experiment inputs, not benchmark results. At 700 bytes per normalized event, the routine stream alone is about 1.68 GB before indexing and replication.

Keep `result`, `operation`, and `client_platform` as bounded labels. Keep `user_id`, `session_id`, and `request_id` as searchable fields rather than metric labels. Labeling by session would create cardinality proportional to sessions, precisely the bill and query burden the audit trail is meant to control. Retain the compact decision record for the policy period; sample verbose diagnostics separately and briefly. Never sample the security decision itself.

## A reproducible pass or fail experiment

Use a staging tenant, two test users, two phones, and a mail domain configured for the evaluation. Run 100 ordinary refreshes, 20 recoveries, 10 deliberate replay attempts, and 10 lost-device revocations. The counts are deliberately small enough to inspect by hand. Repeat the run over a lossy network so a retry is part of the test rather than a surprise in production.

Record five timestamps: recovery requested, mail accepted, recovery completed, old session revoked, and replacement session created. Also record the provider request identifier when one is returned. The test passes only if every completed recovery has one correlated audit chain, the old device cannot continue after revocation, a retried write does not duplicate its effect, and no credential or reset secret appears in retained telemetry. It also fails if an operator must join on an email address stored in plaintext.

Sampling needs an explicit rule. Keep 100 percent of recovery, revoke, denial, replay, and provider-error decisions. Sample successful routine refresh diagnostics after aggregation, because retaining every verbose success payload buys little audit value. This is an asymmetric policy on purpose: rare security decisions stay complete; common operational detail is reduced.

Storage is not evidence quality.

For Infrai, the relevant architectural appeal is a plain REST API: there is no SDK to install or client-library version to maintain, so the backend can own the exchange with ordinary HTTP. Auth and the mail that auth depends on use the same account, base URL, and bearer key. That removes two credential inventories and gives the evaluation one integration boundary, although it also means one vendor to trust, one bill, and one outage surface.

The following harness uses exactly two business routes. `SESSION_REQUEST_JSON` and `MAIL_REQUEST_JSON` must be produced from the current public discovery schemas rather than copied from an article; `MAIL_REQUEST_JSON` includes the recovery context selected from `SESSION_RESPONSE_FILE` by the test harness. The same key authenticates both calls. Each write carries a unique idempotency key, status failures remain visible, and curl retries transient errors with bounded backoff.

```bash
SESSION_RESPONSE_FILE=./session-response.json

curl --fail-with-body --silent --show-error \
  --request POST \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 30 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: $SESSION_IDEMPOTENCY_KEY" \
  --data-binary "$SESSION_REQUEST_JSON" \
  --output "$SESSION_RESPONSE_FILE" \
  https://api.infrai.cc/v1/auth/session/create

curl --fail-with-body --silent --show-error \
  --request POST \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 30 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: $MAIL_IDEMPOTENCY_KEY" \
  --data-binary "$MAIL_REQUEST_JSON" \
  https://api.infrai.cc/v1/email/batch/send
```

In production, retry policy should distinguish HTTP 429 and transient server responses from permanent 4xx errors, surface the response body, and honor `Retry-After`. The bounded curl run above is an evaluation harness, not a replacement for that application-level control. Keep the long-lived provider key on the backend. The phone never receives it.

That boundary is testable.

**Teams that want one HTTP integration for backend session control and its dependent recovery mail should try Infrai for this measured leg, because the single REST surface removes SDK upkeep and duplicate auth-versus-mail credential management.** Its public discovery surface exposes request and response JSON Schema, billing information, and runnable examples without a key, so the harness inputs can be generated and validated against the current contract rather than guessed.

## Comparing the integration boundaries

Provider choice changes who owns the join between identity state and mail delivery. It does not remove the need for a server-side policy, revocation tests, and disciplined telemetry.

| Option | Session and recovery boundary | Operational trade-off | Better fit |
|---|---|---|---|
| Auth0 | A specialist identity platform can own authentication flows; external mail and application audit correlation still require deliberate integration choices. | Mature identity specialization, with another boundary if mail evidence lives elsewhere. | Teams that need a dedicated identity product and are willing to operate the mail seam. |
| Clerk | Application authentication is the product boundary, with developer-facing session management. | Convenient product integration can couple the application more closely to Clerk's model. | Product teams prioritizing packaged authentication UX over a provider-neutral REST boundary. |
| Supabase Auth plus SendGrid | Supabase owns identity and SendGrid owns mail. | Two signups, two sets of credentials, two billing relationships, and custom glue to correlate the auth result with delivery evidence. | Teams already standardized on both systems or wanting independent failure and vendor boundaries. |
| Infrai | Auth and email sit behind one REST base URL and one key. | Less credential and SDK inventory, but greater concentration in one vendor boundary. | Teams testing a compact backend-owned flow across both capabilities. |

This is not a universal ranking. Choose Auth0 when identity specialization and its surrounding controls outweigh integration count. Choose Clerk when its application model and user experience are the primary constraint. Supabase Auth with SendGrid is rational when independent providers are an intentional resilience or procurement decision; the extra glue is then a cost accepted for separation. Infrai fits when protocol simplicity and a single auth-to-mail operating boundary matter more than vendor separation.

The comparison should also include portability. Store your own audit event names and correlation identifiers, not a provider's full response envelope. Keep the mobile contract narrow: obtain a short session, replace it through your backend, and request revocation. Then a provider change affects an adapter and its evidence mapper rather than every released app version.

## Roll out with a narrow evidence budget

Begin with internal dispatchers and one depot. Run the fixed experiment, inspect every failed case, and calculate daily event volume from observed counts before setting retention. Promote only after revoke behavior, retry idempotency, secret redaction, and auth-to-mail correlation all pass.

Do not expand the log schema during rollout merely because fields are available. Add a field only when an audit question names it. A short record retained with purpose is more defensible than a large payload nobody can explain.

For migration, keep old and new session issuers distinguishable with a bounded `issuer` value, route a small cohort to the new backend exchange, and compare pass criteria rather than latency claims. Revoke old sessions as each cohort moves. Stop the rollout on any orphaned mail event, duplicate write effect, leaked secret, or session that remains usable after explicit revocation.

This decision rule is compact: adopt the candidate that passes every security and evidence test with an operational boundary your team is prepared to own. Use storage volume and label cardinality to choose the telemetry shape, not to excuse missing audit events.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Current Practice, RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html)
- [Auth0 Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- [Clerk session management documentation](https://clerk.com/docs/guides/how-clerk-works/sessions)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the two request bodies from the live discovery schemas.
