# ADR-0025: Instrument with OpenTelemetry and view everything in a single LGTM container

- Status: Accepted
- Date: 2026-11-15
- Phase: 0 (structured logs, request IDs), 2 (trace context carried in events), 4 (full tracing, metrics, dashboards)

## Context

One outage touches a worker, the broker, incident, notification, an SMTP server and a browser. Without correlation, debugging means reading seven log streams and guessing which lines belong together. **Observability** rests on three signals:

- **Logs**: discrete events with context.
- **Metrics**: numbers over time (rates, errors, durations, queue depths).
- **Traces**: the path of one operation across services, as a tree of timed **spans**.

Two decisions: how services are **instrumented**, and what **backend** stores and displays the signals within the memory budget.

## Decision drivers

- A trace must continue across RabbitMQ and across the outbox, where the publish happens later and in a different process than the original write.
- Vendor-neutral, self-hosted, offline.
- ~1 GB at most, optional.
- Tracing is a named learning objective.

## Options considered

### Instrumentation

| Option | Pros | Cons |
|--------|------|------|
| **OpenTelemetry (OTel) SDK + auto-instrumentations** (chosen) | The open standard; one API for traces, metrics and logs; W3C Trace Context propagation; auto-instruments HTTP, Express, `pg`, `amqplib`, `ioredis`, pino; any backend | The SDK must load before the app's imports; version churn across many `@opentelemetry/*` packages; manual spans needed around the outbox |
| `@nestjs/observe` (Nest 12's own observability SDK) | First-party; reports in terms of controllers, providers and consumers | New; tied to Nest's own model; not evaluated in depth here. OTel is the transferable skill and already covers the non-Nest pieces |
| Vendor agents | Turnkey | Need a SaaS account; excluded by the hosting constraint |
| Logs with a correlation ID only | Trivial | No timing tree; no metrics; nothing across the broker unless hand-propagated |

### Backend

| Option | Memory | Pros | Cons |
|--------|--------|------|------|
| Jaeger all-in-one (traces only) | ~150–250 MB | Small; good trace UI | No metrics or logs; a second UI needed for those |
| Separate OTel Collector + Tempo + Prometheus + Loki + Grafana | ~1.5–2 GB, five containers | Realistic; each part configurable | Heavy for the budget; a lot of YAML |
| **`grafana/otel-lgtm`** (chosen) | ~0.9–1.3 GB, one container | Collector, Tempo, Prometheus, Loki and Grafana preconfigured and linked (jump from a log line to its trace); accepts OTLP on 4317/4318; made for development and demos | Not for production; data is ephemeral; less to learn about operating each tool |
| SigNoz, Uptrace and similar | 1.5 GB+ (they bring ClickHouse) | Integrated product | Too heavy |

## Decision

**OpenTelemetry in every Node service, exporting OTLP to one `grafana/otel-lgtm:0.41.0` container** behind the `obs` Compose profile. With the profile off, the SDK's exporter is disabled by configuration and services run normally.

### Phasing

| Phase | What is in place |
|-------|------------------|
| 0 | JSON logs with pino (`nestjs-pino`); `X-Request-Id` generated at the gateway, propagated and logged everywhere |
| 2 | The outbox stores `traceparent`; messages carry it in AMQP headers. No SDK yet: the plumbing exists so Phase 4 is switch-on, not retrofit |
| 4 | OTel SDK, auto-instrumentation, manual spans, metrics, dashboards |

### Trace propagation

- **HTTP:** W3C `traceparent` / `tracestate` headers, handled by auto-instrumentation.
- **Broker:** the same headers in AMQP message headers.
- **Across the outbox:** the difficult part. The business transaction and the publish are separated in time and often in process. When a row is written, the active trace context is stored in `outbox.headers`. When the relay publishes, it starts a `publish` span whose **parent is the stored context**, and injects that span's context into the message. The consumer's `process` span is a child of the publish span. Result: one trace from the failed check to the delivered email, with the outbox wait visible as a gap.
- **Batch consumers** (analytics reading a stream) create one span per batch with **span links** to the producers' contexts instead of pretending to have one parent.
- **Browser:** not instrumented. The trace starts at the gateway.

Span naming follows OTel messaging conventions: `ms.events publish`, `incident.monitor-events process`. Each [flow doc](../flows/) lists its expected span tree.

### Signals and methods

- **Services** are described with **RED**: Rate, Errors, Duration of requests and of message handling.
- **Resources** (pools, queues, event loop) are described with **USE**: Utilisation, Saturation, Errors.
- Logs are JSON to stdout **and** exported over OTLP with `trace_id` and `span_id` injected by the pino instrumentation, so Grafana links logs and traces.
- Custom metrics are listed per service in [observability](../cross-cutting/observability.md): outbox depth and age, consumer lag, DLQ depth, check duration, lease-fencing rejections, delivery outcomes, event-loop delay.

### Sampling
100 % of traces (`AlwaysOn`). Local volume is tiny. Production would use parent-based ratio sampling or tail sampling in the Collector; noted, not built.

## Consequences

- Positive: one optional container gives traces, metrics, logs and dashboards, cross-linked.
- Positive: instrumentation is standard OTel; pointing it at another backend is an environment variable.
- Positive: carrying trace context from Phase 2 makes Phase 4 mostly configuration plus a handful of manual spans.
- Negative: ~1 GB when on; it is the largest container.
- Negative: the all-in-one image hides how Collector, Tempo, Prometheus and Loki are wired; that knowledge is not gained here.
- Negative: OTel's Node packages are numerous and version-sensitive; upgrades need care.
- Negative: manual span code in the relay and consumer wrapper must be right or traces silently split.
- Harder later: production-grade telemetry (retention, sampling, alert routing) is a separate project.

## Revisit when

- A break-it experiment needs metrics history across restarts: mount a volume or run Prometheus separately.
- Memory is tight during Phase 4: use Jaeger all-in-one for traces and read metrics from each service's Prometheus endpoint.
- `@nestjs/observe` matures and exports standard OTLP: re-evaluate for the Nest-specific spans.

## Learning choice?

No for OpenTelemetry (it is the industry direction). The all-in-one backend is a convenience choice for a laptop; a team would run managed or separately deployed backends.

## How to explain it in an interview

> "Every service is instrumented with OpenTelemetry and exports traces, metrics and logs over OTLP to a single Grafana LGTM container locally. HTTP propagation is automatic through the traceparent header. The interesting part is the broker: I put the same trace context in the message headers, and because I publish through an outbox, I store the trace context in the outbox row and make the relay's publish span a child of it. So one outage shows up as one trace: failed check, state change, incident opened, page requested, email sent. For metrics I think in RED for services and USE for resources like connection pools and queues."
