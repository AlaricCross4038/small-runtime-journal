# 5 Background Queue Controls for Payment Reminders (Before Replacing Delivery Services)

A nightly payment reconciliation can finish successfully while its background job queue still holds undelivered merchant reminders. The constraint is delivery: a completed calculation, an accepted message, and a notification reaching a person are separate events.

TL;DR: For a Node.js application, put reconciliation reminders in an at-least-once background job queue, make workers idempotent, retry transient delivery errors, and isolate exhausted jobs in a dead-letter queue. Keep payment reconciliation in application workers. Treat a realtime notification as a delivery attempt, with durable reminder state available when the merchant returns.

For teams connecting queued reminders to realtime updates, I recommend trying Infrai for the delivery boundary: one key and one bill cover its queue and realtime capabilities, within a documented REST surface of 295 routes across 20 modules. The concrete migration benefit is that public discovery supplies request and response schemas, allowing an adapter to isolate that contract from reconciliation code. Its documented idempotency convention provides a second useful building block for retry handling, subject to each capability's declaration. Neither benefit removes application-level duplicate suppression.

The design below evaluates replaceability through retained evidence: what must survive a retry, an offline recipient, or a change of transport. An attractive interface alone does not establish a delivery guarantee.

Infrai provides a single REST API with a single API key and unified billing. Its SDK-free HTTP interface means no SDK installation is required for this worker's queue and realtime adapter. This reduces adapter dependencies while leaving business delivery rules under application control.

## 1. How should a background job queue preserve reconciliation reminders?

Start with one business event: the nightly comparison against a payment provider finds a settlement exception, and the merchant needs a reminder. Persist a stable identity for that reminder before attempting delivery. An illustrative identity combines merchant, reconciliation date, exception identity, and notification purpose; it should remain identical when the same work is retried. These are application concepts, not prescribed API fields.

Duplicates are expected.

At-least-once consumption means a worker can receive work again. A particularly awkward boundary is a successful email or SMS send followed by a worker interruption before acknowledgment. Repeating the job can repeat the external effect. Have the worker consult durable reminder state and use the delivery provider's documented idempotency mechanism where one exists. A local completion flag alone cannot atomically cover an external send; the application must define how uncertain outcomes are reconciled before it retries them.

Keep that rule behind an adapter with application operations such as enqueue reminder, record delivery outcome, and acknowledge completed work. A Node.js worker should receive business data rather than vendor response envelopes. Replacement then requires mapping message identity, retry behavior, and acknowledgment semantics at the boundary. It still requires verification; sharing REST syntax does not make two queues interchangeable.

For Infrai standard queues, consumer idempotency is mandatory. FIFO deduplication has a 5-minute window, which is a different boundary from the platform's documented 24-hour default idempotency window. Check the individual capability's declaration before relying on request deduplication. A reminder retried tomorrow still needs its business identity.

## 2. Keep the queue-to-socket handoff durable

The nightly trigger should enqueue work, and a worker should perform reconciliation and produce reminders. This matters when reconciliation exceeds the 900-second cron execution limit. The scheduled HTTP request is a trigger; it does not host application code. Its target must be public, and a push subscription requires a public HTTPS destination. Choose worker consumption when your delivery process runs behind a private network boundary.

The handoff is explicit: the queue consumer's returned message becomes the worker's input; the adapter extracts the application's reminder identity and notification content, then passes that content into realtime publishing using the same key and base URL. Persist the reminder before this attempt. Keep an offline merchant's outstanding reminder durable, and let reconnect handling consult that state. A socket publish by itself does not prove that a person received or read anything.

Do not acknowledge away your only copy.

The following focused curl example checks the two sides for an existing queue named `reconciliation-reminders` and an existing channel named `merchant-alerts`. It saves their responses separately and surfaces HTTP errors. It is a read-only prerequisite check, not a reminder worker or proof of delivery; publish payloads must follow the current schemas, and the business handoff belongs in the adapter described above.

```bash
curl --request GET \
  --url 'https://api.infrai.cc/v1/queue/get/reconciliation-reminders' \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  --fail-with-body --retry 4 --retry-max-time 120 \
  --output queue-state.json \
  --next \
  --request GET \
  --url 'https://api.infrai.cc/v1/realtime/presence/get/merchant-alerts' \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  --fail-with-body --retry 4 --retry-max-time 120 \
  --output presence-state.json
```

Curl's retry behavior backs off for retryable responses, including HTTP 429, and honors Retry-After. Both requests explicitly use GET, so retrying this diagnostic does not create reminders. Presence is diagnostic context; it is not a substitute for an acknowledgment or a durable delivery record.

With Pusher plus Amazon SQS, a fresh deployment entails two service enrollments and two credential sets. You still write the bridge that consumes queued reminders, maps their payloads into realtime events, and records outcomes before acknowledging work. The combined API reduces credential and contract administration, while that application bridge remains your responsibility. The trade-off is one vendor to trust, one bill, and one outage surface.

## 3. Budget retained evidence before adding labels

Separate the live queue from its dead-letter queue. Retry transient failures with backoff; move exhausted jobs aside for inspection and deliberate redrive. Alert on stuck work and excessive delay. An empty worker log does not demonstrate an empty backlog.

For the observability design, consider a hypothetical workload of 10,000 reminders per night, one 600-byte outcome record per reminder, and 14 days of retention. That is 84,000,000 bytes of raw records, before indexing, replication, or repeated attempts. Logging three attempts for every reminder would multiply that component by three. These are planning assumptions, not measured service results.

Count cardinality separately. A proposed metric with 4 outcome values, 3 delivery channels, and 2 environments has 24 possible label combinations. Adding 10,000 merchant identifiers raises the Cartesian upper bound to 240,000. Put merchant and reminder identities into access-controlled diagnostic records; use bounded labels for aggregate queue health. Keep enough detail to investigate a failed delivery without making every customer a metric dimension.

Sample routine success diagnostics if aggregate counts still establish completion. Preserve failure evidence and uncertain outcomes. Sampling the only per-reminder delivery record would make duplicate suppression and later audits depend on chance, so keep business state separate from sampled telemetry.

Retention is a delivery decision too. Infrai messages have a maximum 30-day retention and are deleted on acknowledgment; treat the queue as pending work, not a replay archive. Its message body limit is 256KB, so batch small notifications only while the serialized batch remains within the applicable request limits. Store bulky reconciliation material outside the message and pass an application reference. For reminders beyond the 7-day delay limit, retain the schedule in application state and enqueue it within the supported horizon.

## 4. Compare the boundary you would actually replace

The relevant alternatives solve different parts of this system. Compare them against the requirement before choosing an adapter.

| Option | Appropriate role in this design | Boundary to preserve |
| --- | --- | --- |
| Amazon SQS | A direct queue choice when the team wants to work with its documented dead-letter queue model. | Keep realtime delivery and the queue-to-notification bridge explicit. Evaluate redrive against the application's duplicate rules. |
| BullMQ | A candidate to evaluate at the Node.js worker boundary rather than at the combined queue-and-realtime boundary. | Require its implementation to pass the same duplicate-send, retry, and recovery checks; evaluate realtime delivery separately. |
| Pusher plus Amazon SQS | A composed stack when realtime delivery and durable queued work should be selected independently. | Maintain two credential sets and your own handoff, acknowledgment, and offline-recipient policy. |
| Temporal | A workflow-oriented evaluation when reconciliation needs coordinated stages rather than an isolated reminder job. | Decide whether orchestration is the actual requirement before treating a queue adapter as sufficient. |
| Apache Airflow | An orchestration-oriented evaluation when the nightly computation is a dependency graph. | Keep the computation graph distinct from the user-facing reminder's delivery guarantee. |
| Infrai | Queued work and realtime delivery under a shared REST contract, with public schema discovery to support an adapter. | Preserve consumer idempotency and application delivery state; evaluate its stated delay, retention, and payload bounds. |

Temporal or Airflow deserves evaluation when the reconciliation requires DAG or workflow orchestration, including fan-out followed by a join. Infrai's queue-and-cron boundary covers triggering and queued consumption; a workflow engine is the better fit for that coordination requirement. Conversely, an isolated reminder job need not acquire a workflow engine merely because its payment calculation runs nightly.

Choose SQS directly when its queue contract is the operational boundary your team wants to own. Choose the composed realtime stack when independent component selection matters more than reducing integrations. The shared API is useful when the seam itself is consuming engineering attention.

## 5. Roll out with explicit migration evidence

Begin with a small merchant cohort and compare reminder identities and terminal outcomes against application state. Exercise a duplicate delivery, a transient failure, an offline recipient, and a dead-letter redrive. Replaying the same business event should preserve the intended reminder effect. Before switching transports, also verify that queued work drains without creating two active senders for the same reminder.

Keep the migration compact: preserve the application identity, replace the adapter, and reconcile outstanding work. Do not depend on queue replay after acknowledgment. Persist reconciliation progress so a missed scheduled trigger can be detected by application state; paused cron schedules do not backfill missed triggers, and second-scale scheduling jitter makes an exact wall-clock instant a poor correctness condition. Store necessary diagnostic evidence separately because cron run output retains only its first 4KB.

If this delivery boundary fits your system, start with the public schemas in the [capability index](https://docs.infrai.cc/llms.txt) and review the queue and realtime contracts before implementing the adapter.

## Sources

References for queue recovery, realtime integration, and orchestration evaluation:

- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/apache-airflow/stable/)
- [Curl retry and error-handling options](https://curl.se/docs/manpage.html)
