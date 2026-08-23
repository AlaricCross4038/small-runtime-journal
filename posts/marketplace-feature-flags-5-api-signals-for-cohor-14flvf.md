# Marketplace Feature Flags: 5 API Signals for Cohort Rollout Decisions

For a marketplace experiment served by an Express API in Node.js, the least complex workable feature flag design uses deterministic tenant-and-user assignment, evaluated once per request, for a percentage rollout. Record five bounded telemetry fields: flag key, rule version, assigned variant, tenant cohort, and evaluation reason. Keep those fields on the request span or one decision event, then join them to business and browser outcomes through an existing correlation ID. This preserves the evidence needed to compare cohorts and reconstruct an incident without turning every flag evaluation into another billable log line.

Keep it bounded.

The percentage is the easy part. The hard part is proving which rule a request saw after configuration changed. A boolean such as `new_checkout=true` cannot answer that question.

## What the incident must be able to prove

A feature flag is operational control, not application configuration with a nicer interface. Martin Fowler's treatment of Feature Toggles distinguishes categories with different lifetimes and operational characteristics; a release toggle and an experiment toggle therefore should not inherit one undifferentiated retention policy. The observability design begins with the reconstruction query: for a failed checkout, identify the subject, the exact decision rule, the chosen variant, and the downstream result.

For a multi-tenant marketplace, use a stable subject key derived from tenant ID and user ID. Tenant ID alone keeps every user in a seller account together, which is useful for account-level changes but weak for measuring a buyer-facing experiment. User ID alone may scatter employees or test accounts across variants when the tenant is the real risk boundary. The composite key supports user targeting while retaining tenant context. It must not contain an email address or other raw personal identifier in telemetry; emit an internal opaque identifier or an approved one-way representation according to the system's privacy policy.

The five fields have distinct jobs. `flag_key` selects the decision being investigated. `rule_version` freezes the policy identity even if the percentage changes later. `variant` records the result, such as `control` or `treatment`. `tenant_cohort` carries a bounded analytical class such as `small_seller` or `enterprise_seller`, not a tenant ID. `evaluation_reason` separates percentage assignment from an explicit allowlist, denylist, prerequisite, or default. Without the reason, a 10% rollout can appear to violate its own allocation because manually targeted users are mixed into the denominator.

One more field is tempting: the subject key. Don't make it a metric label. High-cardinality identities belong in sampled traces or access-controlled decision events with deliberately short retention. Metrics should aggregate by bounded dimensions such as variant, cohort, route family, and outcome. That separation is the first cost control and the first guard against an unusable time-series index.

## How should an Express API record feature flag percentage rollout user targeting?

Evaluate the flag in middleware after authentication establishes the tenant and user, but before the experiment can change the response. Attach an immutable decision object to request-local context. Handlers may read it; they shouldn't evaluate the same flag again. Re-evaluation creates a subtle reconstruction gap when configuration refresh occurs between calls, and it spends telemetry twice for one decision.

No second lookup.

Assignment should be deterministic. Conceptually, hash a namespaced value containing the flag key, rule version, tenant key, and user key into a fixed bucket range, then compare the bucket with the rollout threshold. A threshold of 2,500 in a 10,000-bucket space represents a 25% example rollout. The exact hash function and canonical string format are implementation choices, but both must be specified and tested across runtimes. Changing either is a migration because it can reassign subjects. I'm not sure a single composite subject is correct for every marketplace; an account-level pricing experiment may need tenant-only stickiness. The experiment owner has to settle that unit before exposure begins.

Use the API request itself to verify stable behavior. The pseudonymous headers below stand in for identities established by authentication; production clients should not be trusted to assert them directly.

```bash
for attempt in 1 2 3; do
  curl --silent --show-error \
    --request POST \
    --header 'content-type: application/json' \
    --header 'x-test-tenant: tenant-042' \
    --header 'x-test-user: user-197' \
    --header "x-test-attempt: ${attempt}" \
    --data '{"listing_id":"listing-731","quantity":1}' \
    "${MARKETPLACE_ORIGIN}/checkout/quote"
done
```

Three calls by the same subject should resolve to the same variant while the rule version remains fixed. The integration test should also send subjects on both sides of known bucket boundaries, confirm that an explicit targeting rule reports a different evaluation reason, and confirm that the response correlation ID resolves to exactly one recorded decision. Do not test a percentage by expecting 25 of 100 arbitrary users to land in treatment; deterministic hashing is not obligated to produce that exact count in a small sample. Test boundary fixtures for correctness, then use a larger offline distribution test to catch skew.

Express error handling needs the decision context too. Record the outcome after the response finishes, or in the error middleware when the request terminates abnormally, using the same correlation ID. A decision event that exists only on successful responses biases the cohort comparison precisely when incident reconstruction matters. Keep evaluation pure and local; telemetry export must never decide whether the request proceeds.

## Retention math comes before instrumentation

Start with event volume, not enthusiasm. If the service handles 8 million eligible requests per day and emits one 350-byte decision payload for each, the raw payload alone is 2.8 GB per day before indexing, envelopes, replicas, or compression. Thirty days is 84 GB of raw payload. These are arithmetic examples, not storage forecasts: actual billed volume depends on serialization, metadata, compression, replication, and the destination's accounting rules. Measure the encoded event in the real pipeline before setting a budget.

One event per evaluation is usually the wrong default. Emit bounded counters for the whole population, attach decision attributes to traces already selected by the service's sampling policy, and retain a dedicated decision event only where audit or incident requirements justify it. A practical split might retain aggregate counters for trend detection, sampled traces for request-path analysis, and a narrowly scoped exposure table for experiment analysis. The catch is that head sampling can discard the rare failing request before its outcome is known, while tail sampling needs buffering and an explicit capacity plan. Your mileage may vary because failure frequency and reconstruction obligations determine which loss is acceptable.

Count cardinality before deployment. With 2 variants, 4 tenant cohorts, 6 route families, 5 outcomes, and 3 active rule versions, a bounded metric can occupy up to 720 logical series before infrastructure-added labels. Adding 100,000 user IDs multiplies the theoretical space to 72 million. That comparison is why identities do not belong on counters. It also shows why stale rule versions need an expiry policy: old labels cost money long after an experiment ends.

Retention should follow the decision window. Keep detailed evidence long enough to cover the experiment analysis period plus the incident reporting delay, then compact or delete it under a documented policy. Release toggles with a two-day rollout and long-running experiment toggles need different treatment. Longer retention isn't automatically better; it increases cost and privacy exposure while often preserving rule decisions whose code path no longer exists.

Measure once.

## Compare cohorts without lying to yourself

The cohort report needs assignment counts, exposure counts, successful outcomes, failures, and missing-telemetry counts by rule version. Assignment and exposure are different: middleware can assign a variant even when validation rejects the request before the experimental path runs. Treating assignment as exposure dilutes the apparent effect and can hide a variant-specific failure. Define the exposure point next to the code that first exercises the changed behavior.

For browser-facing changes, Core Web Vitals offer standardized outcome names: Largest Contentful Paint, Cumulative Layout Shift, and Interaction to Next Paint. web.dev describes assessment at the 75th percentile, segmented by mobile and desktop. A marketplace should preserve that segmentation and then compare tenant cohorts, rather than averaging a fast desktop population with a slow mobile population. Server latency, checkout error rate, and business conversion still require separately defined measures; the cited browser thresholds do not validate a server-side experiment.

Incident reconstruction has a stricter bar than a dashboard. Given a correlation ID, an operator should be able to recover one rule version and one variant, see why targeting selected it, and follow the request outcome. Given a rule version, the operator should be able to aggregate outcomes by bounded tenant cohort without scanning raw identities. If those two queries require different stores, document the join key and its retention. A beautiful rollout graph cannot compensate for a missing join after the affected configuration has been replaced.

There are limits. This design is not suitable when regulation requires a complete, immutable record of every policy decision; use an audited decision ledger with access controls and retention reviewed by legal and security teams. Tenant-only assignment is preferable when a mixed experience inside one account breaks workflow consistency. Stick with a static configuration change when no gradual exposure, targeted rollback, or experiment analysis is required—the flag lifecycle and telemetry would add control-plane work without buying a useful decision.

## Roll out the telemetry before the treatment

Deploy in three compact stages. First, compute the decision in shadow mode and compare the bounded counters with existing traffic totals; any gap must be explainable as ineligible traffic, sampling, or loss. Second, enable the treatment for internal test identities and verify that a correlation ID joins the decision to both successful and failed outcomes. Third, begin external percentage rollout only after the cohort dashboard and reconstruction query agree on the same rule version.

Set deletion conditions at creation time: an owner, an expiry date, the rule version naming scheme, and the telemetry retention class. During rollback, freeze the affected rule version in the incident record before changing allocation. After the observation window closes, remove the flag path, expire its label values, and preserve only the aggregate evidence required by policy. The result is modest by design—stable assignment, bounded dimensions, and enough detail to answer who saw what without paying to remember every evaluation forever.

## References

- Martin Fowler, "Feature Toggles (aka Feature Flags)": https://martinfowler.com/articles/feature-toggles.html
- web.dev, "Web Vitals": https://web.dev/articles/vitals

Further reading begins with the two primary sources above: one for toggle categories and lifecycle design, the other for browser outcome definitions and percentile assessment.
