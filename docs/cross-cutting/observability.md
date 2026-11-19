# Observability

How to see what the system is doing: logs, metrics and traces, how trace context crosses HTTP and the broker, and which dashboards exist. Decision: [ADR-0025](../adr/0025-observability-stack.md). Concept: [distributed tracing](../learning/concepts/distributed-tracing.md).

## The three signals

| Signal | Answers | Tool | From phase |
|--------|---------|------|-----------|
| **Logs** | What happened, with detail, at one point | pino JSON → stdout and OTLP → Loki | 0 |
| **Metrics** | How much, how fast, how often, over time | OTel metrics → Prometheus | 4 (RabbitMQ's own from 2) |
| **Traces** | Where the time went across services for one operation | OTel traces → Tempo | 4 (context carried from 2) |

All three are viewed in Grafana at `http://localhost:3030` (`obs` profile), with links: from a log line to its trace (`trace_id`), from a span to its logs, from a metric spike to example traces (exemplars, where supported).

## Setup per service

1. `src/instrumentation.ts` creates the OTel `NodeSDK` with auto-instrumentations (HTTP, Express, `pg`, `amqplib`, `ioredis`, pino) and the OTLP exporter, and calls `start()`.
2. It is loaded **before anything else**: `node --import ./dist/instrumentation.js dist/main.js`. Instrumentation patches modules as they are loaded; anything imported first is invisible to it. With ESM this preload flag is the only reliable way.
3. Configuration by standard environment variables: `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT=http://lgtm:4318`, `OTEL_TRACES_SAMPLER=always_on`. With the `obs` profile off: `OTEL_SDK_DISABLED=true`.
4. Resource attributes: `service.name`, `service.version` (git SHA), `deployment.environment=local`, `service.instance.id` (hostname).

## Logs

Every line is one JSON object (pino via `nestjs-pino`).

| Field | Always | Example |
|-------|--------|---------|
| `time`, `level`, `msg` | yes | |
| `service` | yes | `incident` |
| `request_id` | for HTTP-initiated work | `0f8c2a5e-…` |
| `trace_id`, `span_id` | Phase 4+ | injected by the pino instrumentation |
| `org_id`, `user_id` | when known | |
| `message_id`, `queue`, `attempt` | in consumers | |
| Domain IDs | as relevant | `monitor_id`, `incident_id`, `delivery_id` |

Levels: `error` = someone should look (handler failed permanently, unexpected exception); `warn` = degraded but handled (retry scheduled, breaker open, refresh reuse detected); `info` = state changes worth a line (incident opened, service ready); `debug` = off by default.

**Never logged:** passwords, tokens, refresh cookies, API keys, `Authorization` headers, signing secrets, full phone numbers or email addresses in non-auth services (masked: `a***@acme.test`). A pino `redact` list enforces the obvious paths; code review enforces the rest ([security](security.md)).

## Trace propagation

### Over HTTP

W3C Trace Context headers `traceparent` (`00-<trace-id>-<span-id>-<flags>`) and `tracestate`, injected by the HTTP client instrumentation and extracted by the server instrumentation. The gateway forwards them like any other header. The browser does not start traces; the gateway's server span is the root.

### Over RabbitMQ

The same two headers in AMQP message headers. The auto-instrumentation for `amqplib` handles simple publish/consume, but the **outbox breaks the chain**, because the publish happens later, in another loop and often another process. So `@microsentinel/messaging` does it explicitly:

```
write side (inside the business transaction):
  headers = {}; propagation.inject(context.active(), headers)
  INSERT INTO outbox (…, headers) VALUES (…, headers)

relay:
  parent = propagation.extract(ROOT_CONTEXT, row.headers)
  span = tracer.startSpan('ms.events publish', { kind: PRODUCER, links: [] }, parent)
  msgHeaders = {}; propagation.inject(trace.setSpan(parent, span), msgHeaders)
  channel.publish('ms.events', row.routing_key, body, { headers: msgHeaders, … })

consumer wrapper:
  parent = propagation.extract(ROOT_CONTEXT, msg.properties.headers)
  span = tracer.startSpan('<queue> process', { kind: CONSUMER }, parent)
  context.with(trace.setSpan(parent, span), () => handler(…))
```

Rules:

- Retry copies keep the original `traceparent`, so all attempts are in one trace ([retry flow](../flows/notification-retry-and-dlq.md#expected-trace)).
- Batch consumers (analytics) start a span per batch with **links** to each message's context (capped at 128).
- The realtime fan-out span is a child of the publish span; the long-lived SSE response is not traced as one span.
- If tracing is disabled, `inject` writes nothing and everything still works.

### Span names and attributes

| Span | Kind | Key attributes |
|------|------|----------------|
| `GET /v1/incidents` (route template, never the raw URL) | server | `http.route`, `http.response.status_code`, `ms.org_id` |
| `ms.events publish` | producer | `messaging.system=rabbitmq`, `messaging.destination.name=ms.events`, `messaging.rabbitmq.destination.routing_key`, `messaging.message.id` |
| `<queue> process` | consumer | same + `ms.attempt`, `ms.duplicate`, `ms.stale`, `ms.outcome` |
| `monitor.check` | internal (root) | `ms.monitor_id`, `ms.outcome`, `ms.failure_reason`, `ms.lease_token` |
| `monitor.apply_result` | internal | `ms.fenced`, `ms.state_changed` |
| `incident.open`, `escalation.fire` | internal | `ms.incident_id`, `ms.escalation_level` |
| `notification.plan`, `notification.send` | internal | `ms.delivery_id`, `ms.channel_type`, `ms.outcome` |

Messaging attribute names follow OpenTelemetry's messaging semantic conventions; check the current convention version when implementing, as they have been renamed between releases.

Each [flow](../flows/) lists the expected span tree. Comparing the real trace with that tree is the Phase 4 Definition of Done.

## Metrics

### Methods

- **RED** for anything that serves requests or messages: **R**ate, **E**rrors, **D**uration.
- **USE** for resources: **U**tilisation, **S**aturation, **E**rrors.

### Standard per service (from shared packages)

| Metric | Type | Labels |
|--------|------|--------|
| `http.server.request.duration` | histogram (s) | `http.route`, `http.request.method`, `http.response.status_code` |
| `messaging.process.duration` | histogram | `queue`, `outcome` (`ok`, `duplicate`, `stale`, `retry`, `dead`) |
| `consumer.duplicates` | counter | `queue` |
| `outbox.pending` | gauge | — |
| `outbox.oldest_age_seconds` | gauge | — |
| `db.client.connection.count` | gauge | `state` (`idle`, `used`), pool max as an attribute |
| `db.client.connection.wait_time` | histogram | — |
| `nodejs.eventloop.delay` | histogram/gauge (p50, p99) | — |
| `process.memory.usage` | gauge | — |

Service-specific metrics are listed in each [service doc](../services/), "Observability".

### From RabbitMQ

The `rabbitmq_prometheus` plugin exposes `:15692/metrics`; the LGTM collector scrapes it. Most useful: `rabbitmq_queue_messages_ready`, `rabbitmq_queue_messages_unacked`, `rabbitmq_queue_consumers`, per queue. Consumer **lag** for queues is "messages ready"; for streams it is computed by analytics (`analytics.stream.lag_seconds`).

### Cardinality rule

Labels are bounded sets: route templates, queue names, outcomes, channel types. **Never** a user ID, monitor ID, URL or message ID as a metric label; those belong on spans and logs. One label with a million values creates a million time series.

## Dashboards

Provisioned from `infra/lgtm/dashboards/*.json` (version-controlled).

| Dashboard | Panels |
|-----------|--------|
| **System overview** | Request rate and error rate per service; p95 latency per service; health of each container; checks/min; open incidents |
| **Messaging** | Ready and unacked per queue; consumers per queue; DLQ depth (all queues, red when > 0); outbox pending and oldest age per service; publish rate by routing key |
| **Monitoring pipeline** | Checks by outcome; check duration p50/p95; schedule lag; in-flight checks per worker; fenced completions; event-loop delay per worker |
| **Incidents and notifications** | Incidents opened/resolved; time-to-open; deliveries by status and channel; attempts distribution; circuit-breaker state; page-to-first-attempt latency |
| **Analytics** | Stream lag; batch size and duration; dirty buckets; quarantined events |

## Alerts (local, informational)

Grafana alert rules; they show in the UI, nothing is sent anywhere.

| Alert | Condition | Why it matters |
|-------|-----------|----------------|
| No checks | `rate(monitor.checks) == 0` for 2 min with active monitors | The product is blind |
| Outbox stuck | `outbox.oldest_age_seconds > 30` for 1 min | Events not flowing; a relay or the broker is down |
| DLQ not empty | any `*.dlq` ready > 0 | Something needs a human |
| Consumer missing | `rabbitmq_queue_consumers == 0` on a work queue for 1 min | A service is down; work is piling up |
| Schedule lag | `monitor.schedule.lag_seconds` p95 > 15 s | Workers saturated |
| Event loop | `nodejs.eventloop.delay` p99 > 200 ms for 1 min | Something is blocking a process |

## Correlation in practice

A user reports "acknowledging incident X failed". Path:

1. The error screen shows `requestId` ([web app](../frontend/web-app.md)).
2. Loki: `{service=~".+"} | json | request_id="…"` → lines from gateway and incident, with `trace_id`.
3. Tempo: open the trace → the incident server span → the `UPDATE` span shows 0 rows → the timeline shows the incident was resolved 200 ms earlier → `409` was correct.

Practise this path once in Phase 4; it is a good interview story.

## Cost of all this

The `lgtm` container is ~1 GB. Instrumentation adds a few percent of CPU and ~20–40 MB per Node process. Locally sampled at 100 %; a real system would sample ([ADR-0025](../adr/0025-observability-stack.md)).
