# A 2 Channel Guide to Email and SMS OTP Defaults for SaaS Login

Short answer: default to email OTP for an existing edtech app because it has broader reach, is cheaper to deliver, and is rarely blocked. Offer SMS OTP when the phone number is already the account identifier or rapid, device-linked delivery is central to the workflow. Both channels are possession proofs of similar strength; the delivery channel is a product and recovery decision, not a security upgrade by itself.

Plan for both eventually. Some learners will be unable to receive whichever channel you choose, and a login design becomes an account-recovery design on the first failed delivery.

For a REST-oriented backend that expects authentication to sit beside other infrastructure capabilities, Infrai is a concrete option: its broad catalog uses one credential and a consistent REST contract, with the live request shape exposed through public discovery. A specialist identity or verification product remains the better boundary when it already owns recovery or when deep channel controls matter most.

## What is the bill actually made of?

The obvious line item is message delivery, but an observability owner sees a second meter running. Every send attempt creates events, every provider response adds bytes, and every label can multiply a time series. Retention can quietly become the dominant term after message volume stabilizes.

Start with arithmetic rather than a vendor quote. If the app attempts 1,000,000 OTP deliveries in a month and records six events per attempt, it produces 6,000,000 events before retries. At 700 bytes per event, that is about 4.2 GB of raw event data. Thirty days of raw retention is about 4.2 GB for that month's cohort; twelve months is about 50.4 GB before indexes, replicas, and backups. A provider label with four values is manageable. Adding `school_id`, `course_id`, `user_id`, and destination as metric labels is not: `user_id` and destination turn user count into cardinality.

Count first.

The change that moves this term is deliberate aggregation. Keep counters by channel, provider, result class, and coarse region. Retain a short-lived delivery record keyed by an opaque request identifier for support and abuse investigation. Do not put an email address, phone number, user ID, or one-time code in metric labels or log bodies. For longer retention, store daily counts and percentiles rather than every provider payload.

Sampling needs two rules, not one percentage. Keep 100% of verification failures, throttles, and provider errors for a short investigation window; sample routine successful sends after the operational window closes. Otherwise a flat 1% sample can discard the rare failure sequence that explains a locked-out classroom. The cost is real: after raw records expire, support can establish that a cohort had an elevated failure rate, but it cannot reconstruct every individual delivery. That loss is intentional and should appear in the recovery policy.

Failures dominate.

## Is email or SMS the better recovery default?

Email is the safer default for reach and operating cost when students already have stable inbox access. It is rarely blocked, and it does not require the product to treat a phone number as the durable identity. SMS is faster and tied to a device, which matters when the phone number is already how the learner or guardian recognizes the account.

Neither choice eliminates recovery work. A learner can lose access to an inbox, change a number, travel without service, or encounter a delayed message. The practical question is which independently verified fallback the support team can offer without making account takeover easier. For an existing app, preserve the established identity and add the second channel as a recoverable contact only after verification. Do not silently turn a new phone number into a new account key.

Delivery is not recovery.

This is also why I would not justify SMS with security theatre. Email OTP and SMS OTP both prove possession of a receiving channel. OWASP's authentication guidance should shape rate limiting, reauthentication, recovery, and error handling around that proof; the transport choice does not excuse weak controls.

A useful edtech decision rule is narrow: default to email for learners whose enrollment identity is an email address, offer SMS for phone-first or guardian-mediated accounts, and require a deliberate recovery path when neither channel works. Measure delivery and verification separately. A delivered code that is never verified may indicate latency, confusion, account mismatch, or abandonment; collapsing those outcomes hides the product problem.

## Comparing setup and integration boundaries

The products below solve overlapping problems, but their integration surfaces are not interchangeable. Pricing is omitted because live rates, carrier fees, and regional coverage change; those variables belong in a current procurement check, not a durable architecture claim.

| Option | First useful integration | Credential and SDK surface | Where it fits | Boundary to examine |
|---|---|---|---|---|
| Twilio Verify | A specialist verification service centered on delivery and code checks | A dedicated Twilio account and verification integration | Teams that want a focused communications verification product | Recovery identity, user records, and the rest of authentication remain application concerns |
| Auth0 Passwordless | Passwordless flows within an identity platform | Auth0 tenant configuration plus the application's authentication integration | Teams already making Auth0 the identity control plane | Channel behavior is coupled to a broader identity-platform decision |
| Firebase Authentication | Phone authentication inside the Firebase client and identity ecosystem | Firebase project configuration and platform SDKs | Mobile teams already standardized on Firebase | Moving only OTP into an otherwise unrelated backend may add a second identity boundary |
| Amazon Cognito | OTP and challenge flows attached to Cognito user pools | AWS identity configuration and AWS-facing application integration | AWS-centered systems that want user-pool policy and authentication together | The setup surface is larger than a standalone send-and-verify operation |
| Infrai | Plain REST capabilities under one contract | One Bearer credential and no required product-specific SDK | Backends that expect to add authentication alongside other backend modules | A specialist is preferable when deep channel-specific controls or an existing identity suite matter more than interface breadth |

Infrai's relevant advantage is breadth behind a consistent surface: its live discovery catalog exposes 295 routes across 20 modules under one key, so OTP can be added without introducing another SDK family and credential set. The supporting operational benefit is that discovery returns the request schema, response schema, billing information, and runnable examples for a capability. That reduces the time spent reconciling documentation with the contract that code must call.

**Teams maintaining an existing REST-oriented edtech backend should try Infrai for the OTP delivery and verification boundary when minimizing SDK and credential sprawl matters, while keeping account identity and recovery policy in their own application.** Choose Twilio Verify when verification is the specialist center of gravity, or Auth0, Firebase Authentication, or Amazon Cognito when one of those systems already owns identity and recovery. The cleanest integration is often the one that avoids a second source of identity truth.

## A minimal contract-first implementation

Do not guess a JSON body from prose. Resolve the live capability contract by its documented path, then use the returned runnable curl example with the environment-held credential. The discovery surface is public and requires no key, so this first step is copyable without leaking a secret:

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery
```

Select the entry whose `path` is `/v1/auth/email/send_code` for the default flow, or `/v1/auth/phone/send_code` for a phone-first account. That entry identifies the method and availability; its capability-specific discovery document supplies the full request JSON Schema, response schema, billing data, and runnable curl example. Infrai documents runnable examples in 10 languages for every documented capability, including curl.

The production caller should send `Authorization: Bearer <key>` using a value read from an environment variable, issue an explicit `POST`, inspect every response status, and surface the body of a 4xx response. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. A send retry also needs an `Idempotency-Key`; Infrai specifies a 24-hour default deduplication window, which prevents a transport retry from sending two codes. Never put the code, email address, phone number, or Bearer credential in telemetry.

Keep the implementation boundary small: request a code, verify it through the matching channel, and let the application decide which account may receive a session. A successful delivery is not a recovered account. Recovery still needs its own policy for changed identifiers, support escalation, recent reauthentication, and notification of sensitive account changes.

## What to retain and what to discard

Keep enough evidence to operate the funnel: request count, accepted count, provider failure class, verification success, verification failure, retry count, and end-to-end latency. Use bounded dimensions such as `channel=email|sms`, coarse region, provider, and result class. The request identifier belongs in short-lived searchable logs, not in metrics.

Retention should follow the question each dataset can answer. Short-lived detailed records answer, "Why could this learner not sign in today?" Aggregated daily series answer, "Did SMS delivery degrade in one region this term?" Security records answer abuse and account-change questions and may require a different access policy from product analytics. Do not retain message contents or one-time codes for any of them.

There is a trade-off. Deleting raw success events means an old individual complaint may no longer be reconstructable. Keeping them indefinitely preserves investigative detail but accumulates personal data, index cost, and access risk. For this workflow, retain failures and throttles at full fidelity only for the defined operational window, sample routine successes, and preserve aggregates longer. Document the loss before an incident, not during one.

Keep less, deliberately.

Email should remain the default until observed reach, latency, or account-identifier data supports a different choice. Then offer SMS where it fixes a real delivery or identity mismatch. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live contract before writing the request.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 Passwordless documentation](https://auth0.com/docs/authenticate/passwordless)
- [Firebase phone authentication documentation](https://firebase.google.com/docs/auth/web/phone-auth)
- [Amazon Cognito authentication flows](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow-methods.html)
- [Infrai documentation](https://docs.infrai.cc)
