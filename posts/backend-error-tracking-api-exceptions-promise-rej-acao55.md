# Backend Error Tracking API: Exceptions, Promise Rejections, Stack Traces, and Request IDs

A Node.js Express error tracking API for a gaming AI agent loop has an unusually sharp rollback constraint: an exception must identify the deployed release and affected request before another match sends more traffic through the same path.

Short answer: capture server exceptions and unhandled promise rejections with the message, stack, environment, release, request ID, and optional user context; group the resulting events into an error inbox, but choose a fuller error platform when source-map decoding, crash symbolication, session replay, or built-in alert delivery is required.

This is an architecture decision about recovery evidence, not maximum telemetry volume. The useful event is the one that tells an operator which release to roll back and which agent-loop request to inspect. Ten copies of the same stack add storage and ingestion cost without making that decision ten times better.

## How does a Node.js Express error tracking API capture promise rejections and stack traces?

The minimum event has six operational dimensions: `message`, `stack`, `environment`, `release`, `request_id`, and, when policy permits it, user context. The first two explain the failure. Environment and release establish the rollback boundary. Request ID connects the exception to the game request or agent turn. User context helps establish impact, but it should be optional because identifiers carry privacy and retention consequences.

Keep those roles separate. A request ID should be unique enough to locate one execution, while a release should repeat across every event from one deployment. Putting a random request ID into a grouping key creates one group per event. Putting a raw user ID into labels creates user-scale cardinality and makes deletion obligations harder. Neither improves rollback safety.

For a Node.js Express service, install handlers at three boundaries: the Express error middleware, the process-level `unhandledRejection` handler, and the process-level `uncaughtException` handler. Capture before shutdown where the runtime permits it, but do not treat capture as recovery. An uncaught exception can leave process state suspect; the supervisor should restart the process, and the deployment system should decide whether the release crosses its rollback threshold.

One caution matters here. There is no distributed trace query or span tree in this option. Log records can carry `trace_id` and `span_id` for correlation, but those identifiers do not create a tracing backend. If the actual question is why the model call, tool call, and state write consumed 2.4 seconds in aggregate, exception capture alone cannot answer it.

That boundary is easy to miss.

## Retention budget and privacy invariants

The decision is to use a compact capture-and-group path for backend exceptions, while keeping latency measurement in metrics or traces and silent-job detection in a heartbeat monitor. Four invariants make the design reversible.

First, every captured exception names an environment and release. Second, request IDs propagate through the AI agent loop rather than being regenerated at each internal step. Third, the capture client has a strict timeout and cannot hold open the player-facing response. Fourth, telemetry failure never changes game state or retries an agent action. The error record observes the transaction; it does not participate in the transaction.

The rollback rule should use grouped failures, not raw event count. Imagine a clearly labeled capacity model, not a benchmark: 30,000 agent turns per hour, a 1% exception rate, and an average serialized event of 6 KiB produce about 1.76 GiB per day before indexes and replicas. Retaining every duplicate for 30 days would preserve roughly 52.7 GiB of payload. Sampling repeated events after the first few examples can reduce stored bytes, but it also reduces evidence about how widely a failure spread. The compromise is to retain complete examples for each new release and group, then count later duplicates separately. Exact thresholds depend on traffic shape; I'm not sure there is a defensible universal number.

Cardinality receives the same treatment. `environment` and `release` are bounded dimensions. Exception class is usually bounded. Request ID and user ID are not. Store high-cardinality identifiers as searchable event fields, not metric labels, and do not copy the full stack into logs, metrics, and the error system unless each copy has a declared use and retention period.

For alerting, the failure boundary is explicit: this error capability has no built-in threshold rules, phone or SMS delivery, or webhook notification route. A small worker must poll recent groups or search results and send Slack or email through separate code. Polling introduces detection delay and another component to operate. A Healthchecks-style service is also needed for the opposite failure mode, where a scheduled job never runs and therefore emits no exception at all.

## Provider comparison for rollback safety

The table separates verified fit from evaluation work. Sentry documents event grouping and fingerprint mechanics. Rollbar and Bugsnag are credible products to include in a proof of concept, but this record does not assign them unverified features. Infrai is the compact API option evaluated here.

| Option | Evidence relevant to this decision | Best fit | Reason to reject it here |
|---|---|---|---|
| Infrai | Captures backend exceptions, groups events, and provides list and search operations | A small service that wants one plain REST API with no SDK or client-library version to maintain | Not suitable when source maps, Electron minidump symbolication, session replay, built-in alert routing, or trace/span-tree queries are required |
| Sentry | Its documentation explains default grouping and custom fingerprints | Teams that need to tune how related stack traces become issues | Stick with another option if a deliberately small capture-and-poll integration is the controlling requirement |
| Rollbar | Candidate dedicated error-tracking product for the same proof-of-concept test suite | Teams prepared to evaluate a dedicated product against their own rollback workflow | Reject only after testing release tagging, grouping stability, notification delay, and retention under representative events |
| Bugsnag | Candidate dedicated error-tracking product for the same proof-of-concept test suite | Teams prepared to evaluate a dedicated product against their own rollback workflow | Reject only after the same tests; a name on a comparison table is not capability evidence |
| OpenTelemetry plus an observability backend | A standards-oriented instrumentation path whose backend must be selected separately | Systems already standardizing telemetry production across signals | More moving parts than a narrow error inbox when backend exceptions are the immediate job |

Infrai's concrete advantages in this narrow design are a plain REST interface, which avoids installing an error SDK or tracking its versions, and one API key across 295 routes in 20 modules. A team that later adds metrics around agent-loop latency therefore does not have to create and rotate another service key just to extend the telemetry workflow; this ADR still depends only on error capture and inbox operations. The catch is substantial: a minified browser stack stays difficult to read because there is no source-map reverse lookup, and Electron minidumps are not symbolicated. It also lacks session replay.

Do not hide that trade-off behind ingestion cost. Rollback safety wins only when the captured backend stack is readable enough to identify the release regression.

## Integration path for capture

The application should build the JSON event from its actual exception object, then give a small sender a bounded retry budget. The following shell example exercises the single write route used by the design. `ERROR_API_BASE` is deliberately supplied by the deployment, `INFRAI_API_KEY` stays in the environment, and `event.json` contains the verified capture fields described above. The client retries HTTP 429, honors an integer `Retry-After` value when present, and uses a stable idempotency key so the same event is not applied twice during a retry.

```bash
#!/usr/bin/env bash
set -u

: "${ERROR_API_BASE:?set ERROR_API_BASE}"
: "${INFRAI_API_KEY:?set INFRAI_API_KEY}"
: "${ERROR_EVENT_ID:?set ERROR_EVENT_ID to a stable value for this exception}"

body_file="$(mktemp)"
header_file="$(mktemp)"
trap 'rm -f "$body_file" "$header_file"' EXIT

attempt=0
while [ "$attempt" -lt 4 ]; do
  status="$(curl --silent --show-error \
    --request POST \
    --url "${ERROR_API_BASE}/v1/errors/capture" \
    --header "Authorization: Bearer ${INFRAI_API_KEY}" \
    --header "Content-Type: application/json" \
    --header "Idempotency-Key: ${ERROR_EVENT_ID}" \
    --data-binary @event.json \
    --dump-header "$header_file" \
    --output "$body_file" \
    --write-out '%{http_code}')" || {
      cat "$body_file" >&2
      exit 1
    }

  if [ "$status" -ge 200 ] && [ "$status" -lt 300 ]; then
    cat "$body_file"
    exit 0
  fi

  if [ "$status" -ne 429 ]; then
    cat "$body_file" >&2
    exit 1
  fi

  retry_after="$(awk 'BEGIN { IGNORECASE=1 } /^Retry-After:/ { gsub("\\r", "", $2); print $2 }' "$header_file")"
  if ! [[ "$retry_after" =~ ^[0-9]+$ ]]; then
    retry_after="$((2 ** attempt))"
  fi
  sleep "$retry_after"
  attempt="$((attempt + 1))"
done

cat "$body_file" >&2
exit 1
```

The Express middleware should invoke this sender asynchronously with a strict deadline after assigning the request ID and release. The process handlers use the same normalization path, which prevents a promise rejection and a middleware exception with the same stack from acquiring incompatible field names. Do not send arbitrary request bodies, authorization headers, chat text, or model prompts as user context. For an AI agent loop, those fields can be large, sensitive, and expensive to retain.

The inbox side uses group and event listings for triage, search, and manual resolution. Keep that query work outside the player request path. A polling notifier should checkpoint its last completed interval and deduplicate notifications by group plus release; otherwise a 60-second poll can turn one persistent exception into a channel full of repeated alerts.

## Migration trigger for the rejected design

The rejected design is “exceptions only, everywhere”: replace metrics, traces, deployment health, and job heartbeats with error capture because one stream looks cheaper to operate. It fails the application scenario. Successful but slow agent loops do not throw, a scheduler that never starts emits no exception, and a cross-service latency regression needs timing relationships that an error inbox does not provide.

Use the narrow design for a simple Node.js backend whose principal question is which deployed release introduced a grouped exception. Choose Sentry, Rollbar, Bugsnag, or another dedicated product after a representative proof of concept when readable minified stacks, native notification workflows, crash symbolication, or replay evidence controls recovery time. Keep OpenTelemetry plus an appropriate backend when end-to-end latency through model, tool, and storage calls is the primary investigation path. Add a heartbeat product when “the task should have run but did not” is a material failure.

The final acceptance test is operational: deploy two harmless release identifiers, generate the same exception under both, verify that grouping preserves the release boundary needed for rollback, and confirm that a request ID locates the affected agent turn without becoming a grouping dimension. Then measure event bytes, unique groups, and notification lag. Your mileage may vary because stack shape and release cadence determine both grouping quality and storage volume.

Rollback first.

## References

- https://prometheus.io/docs/practices/naming/
- https://docs.sentry.io/concepts/data-management/event-grouping/
