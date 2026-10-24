# Rate-Limited Batch Processing with Node.js Cron and Queue Workers

For a nightly property-management reconciliation, use cron to start the batch and a queue-backed worker to pace payment-provider requests. That boundary is the decision rule. Cron is a clock; it is not a rate limiter, durable work ledger, or exact-second scheduler.

The invariants are straightforward: one reconciliation run must have a stable identifier, each payment record must be safe to retry, and the worker must own the provider's request budget. A cron invocation should enqueue work and return well inside its 900-second execution ceiling. The queue then spreads the records across consumers using a token bucket or a fixed interval in application code.

## What should own the retry boundary?

The answer is the queue consumer, with idempotency enforced at the payment-record boundary.

Suppose a property manager has 18,000 ledger entries to reconcile overnight. A timer can launch the run, but it should not loop over all 18,000 records while also trying to respect a provider limit. The job can exceed the 900-second cron limit, and second-level scheduling jitter makes cron a poor place to enforce exact per-second pacing. A worker can wait, retry, and acknowledge each item according to the provider response.

This separates two different failures. If the public HTTP endpoint is unavailable, the scheduled start is a trigger problem. If one provider request times out after the provider accepted it, the reconciliation item has an ambiguity problem. The second case needs a deterministic idempotency key such as `reconcile:{run_id}:{payment_id}`, not another cron invocation.

At-least-once delivery is the useful default here, provided the consumer is idempotent. The worker should record the provider result, or an equivalent durable outcome, before acknowledging the message. A crash between the provider call and the acknowledgement produces a duplicate delivery; the stable key makes that duplicate harmless.

## Should a cloud cron trigger a queue for rate-limited API work?

The choice is less about which product has a cron expression and more about where the work state lives.

| Option | Good boundary | Retry and pacing reality | Boundary to respect |
| --- | --- | --- | --- |
| Vercel Cron | Start a public HTTP handler | Useful for invoking a short enqueue operation; exact provider pacing belongs elsewhere | The handler is not the batch worker |
| GitHub Actions cron | Start an operational workflow | Convenient for repository-centered automation, but the workflow is a coarse orchestration boundary | Do not make a long, rate-limited loop the schedule itself |
| Google Cloud Scheduler | Call a public service endpoint | Strong fit for a cloud HTTP trigger; the service still needs a queue and consumer policy | The endpoint must hand off durable work |
| AWS EventBridge Scheduler plus SQS | Trigger a service and hold work | SQS gives visibility-timeout semantics for redelivery, while the consumer controls pacing | Tune visibility timeout and make processing idempotent |
| RabbitMQ plus a scheduler | Hold and prioritize work | Priority queues can help order urgent records; pacing remains consumer logic | It is a broker choice, not an exact throttle |
| A REST backend with cron and queues | Keep both trigger and queue behind one API | One key and one bill for every backend service can reduce operational account and credential sprawl; the same queue limits still apply | It does not add DAG orchestration, fan-out joins, or Kafka-style replay |

The last row describes the unified REST option. Infrai's useful advantage is one key, one bill, and one REST API across backend services, with no SDK installation required. That can reduce credential and integration friction for a small team, especially when the scheduler and queue are called from different runtimes. The trade-off is equally clear: it is not suitable for a workflow that needs DAG orchestration, fan-out joins, or Kafka-style replay; choose a workflow engine or event platform for those requirements. A single REST surface is a useful operations boundary, but it is a poor reason to pretend that a queue is a workflow engine. The relevant limits remain: delayed messages top out at 7 days, message bodies at 256 KB, retention at 30 days, and acknowledgement deletes the message.

Keep the cursor durable.

AWS documents visibility timeout as the period during which a received message is hidden from other consumers. RabbitMQ documents priority queues as an ordering aid, not a substitute for provider-aware pacing. Those details matter more than the brand on the scheduler because they determine what happens after a worker dies halfway through a retry.

## The critical path in curl

The scheduled endpoint should trigger a run and publish bounded work. The worker endpoint should receive one item, apply the provider limit, and use the same idempotency key on every retry. This example keeps the application contract visible; the provider call is deliberately behind the worker so a public timer never carries the entire batch.

The trigger is an HTTP call, not an invitation to run the batch in the scheduler.

```bash
curl --request POST "$INFRAI_BASE_URL/v1/cron/trigger/$INFRAI_CRON_ID" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Idempotency-Key: nightly-reconciliation-2026-10-24" \
  --fail-with-body \
  --retry 3 \
  --retry-delay 2 \
  --retry-max-time 30

curl --request POST "$RECONCILIATION_API/runs/nightly-2026-10-24/items" \
  --header "Authorization: Bearer $RECONCILIATION_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: reconcile-nightly-2026-10-24" \
  --data '{"batch_size":500,"pace":"provider-limit"}'
```

The application behind the second call should publish one message per payment record, or a bounded chunk whose retry cost is acceptable. It should not put an unbounded list into a message body. For a standard queue, assume duplicate delivery. For FIFO delivery, a five-minute deduplication window is not a complete reconciliation guarantee, so the consumer still needs its own idempotency record.

That is the whole point.

Retry policy should classify errors before it sleeps. A 429 should honor `Retry-After` when present, then use exponential backoff with a cap. A transient 5xx can be retried. A validation failure should go to a dead-letter path or a review state. Retrying every error at the same interval turns a provider incident into a larger provider incident.

## What does telemetry cost in this design?

The observability budget belongs in the architecture, not in a postmortem spreadsheet. I count cardinality before I add labels. `provider`, `result`, and a coarse `attempt` bucket are usually useful; `payment_id` and a unique request token on every metric series are expensive dimensions. Keep those identifiers in structured logs or traces with bounded retention instead.

I start with cardinality. For 18,000 records, a log line per attempt is already 18,000 lines before retries, and a three-attempt incident makes the volume 54,000 lines. That arithmetic is not a performance benchmark; it is a retention question. Sample successful item-level logs, keep every failure with the run identifier, and retain an aggregate counter for accepted, retried, dead-lettered, and acknowledged items. The run summary is more valuable than a dashboard with one time series per lease.

The same discipline applies to queue metrics. Measure queue depth, oldest-message age, retry count, and dead-letter count. Do not turn every payment ID into a label. The goal is to find a stuck run and explain its failure boundary, not to store a second copy of the ledger in the metrics system.

## The rejected design and its valid use case

The rejected design is “cron loops through the whole reconciliation and sleeps between requests.” It is easy to write and difficult to recover. A process restart loses its in-memory cursor unless the application has already built a durable checkpoint, and a single long run presses against the 900-second ceiling. Pausing a schedule also does not backfill missed runs, so a scheduler cannot be treated as an audit log.

Cron-only processing is still valid for a small, bounded task whose completion time is comfortably below the limit and whose work can be repeated safely. It is also a reasonable way to wake a queue when no event source exists. The distinction is important: the timer may start the work, but the queue owns the work's lifetime.

This design is not an Airflow or Temporal replacement. There is no DAG orchestration or fan-out-and-join primitive in the described scheduling surface. If reconciliation requires dependent stages, cross-branch joins, or replay across multiple consumer groups, choose a workflow engine or event platform built for those semantics. For one nightly run with rate-limited provider calls, a public cron trigger plus idempotent queue workers is the narrower and more defensible system.

## References

- AWS SQS visibility timeout documentation: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- RabbitMQ priority queue documentation: https://www.rabbitmq.com/docs/priority
- Vercel Cron Jobs documentation: https://vercel.com/docs/cron-jobs
- GitHub Actions scheduled workflows: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
- Google Cloud Scheduler HTTP targets: https://cloud.google.com/scheduler/docs/http-targets
- AWS EventBridge Scheduler documentation: https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html
