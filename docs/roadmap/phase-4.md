# Phase 4: Observability and CQRS

**Effort:** 35–45 h (4 weeks). Overview: [phases](phases.md).

## Goal

See the whole system: one trace from failed check to delivered email, metrics and dashboards for every service and queue. Build the analytics read model from the event streams, prove it can be rebuilt, and add the contract and browser tests that hold the system together.

## Scope

**In:**

| Area | Content |
|------|---------|
| `observability` package | OTel SDK bootstrap (`--import`), auto-instrumentations, pino trace correlation, standard metrics |
| `messaging` | Producer/consumer spans with stored context through the outbox; span links for batches |
| Manual spans and metrics | Per service, as listed in each [service doc](../services/) |
| lgtm | `obs` profile; provisioned dashboards and alert rules ([observability](../cross-cutting/observability.md)) |
| RabbitMQ | Prometheus plugin scraped |
| analytics | New service: stream consumers, projectors, rollups, partitions, query API, rebuild ([analytics](../services/analytics.md)) |
| Tests | Pact message pacts (2), OpenAPI drift tests, Playwright J1–J5, two automated chaos tests |

**Deferred:** everything in Phase 5.

**Frontend slice:** dashboard analytics (organization uptime, MTTA, MTTR, per-monitor table with window selector); monitor detail charts (uptime bars, latency avg/p95 lines) with "as of" and independent error states; Playwright suite.

## Tasks (one per sitting)

1. `observability`: SDK bootstrap, exporters, resource attributes; switch on in one service; see a trace in Tempo.
2. Roll out to all services; pino correlation; `OTEL_SDK_DISABLED` without the profile.
3. `messaging`: inject/extract through outbox and headers; consumer spans; check the outage trace matches [outage-to-resolution](../flows/outage-to-resolution.md#expected-trace-phase-4).
4. Metrics: standard set + service-specific; RabbitMQ scrape.
5. Dashboards (5) and alert rules as provisioned JSON.
6. analytics: skeleton, migrations (partitioned table), `analytics-migrate`, profile.
7. analytics: checks stream consumer + batch projector + offset; integration test with a crash mid-batch.
8. analytics: lifecycle consumer + dimension and incident facts; quarantine.
9. analytics: rollup job + partition job under advisory locks.
10. analytics: query API (overview, summary, series); API tests incl. tenancy.
11. analytics: rebuild + status endpoints; run the [rebuild flow](../flows/analytics-rebuild.md) and compare numbers.
12. web: dashboard analytics; monitor charts with Recharts.
13. Pact: consumer and provider tests for both pacts; OpenAPI drift tests for all six services.
14. Playwright: setup against Compose, J1–J3.
15. Playwright: J4–J5; automated chaos tests (notification offline, duplicate delivery).
16. Break-it experiments 11 and a full re-run; measure memory with everything on; record the demo.

## Definition of Done

| # | Check | Expected |
|---|-------|----------|
| 1 | `pnpm ci:local && pnpm e2e` | Both exit 0 |
| 2 | `COMPOSE_PROFILES=obs,analytics docker compose up -d --wait` | All healthy |
| 3 | Cause an outage; in Grafana Explore → Tempo, search `service.name=monitor-worker name=monitor.check status=error` | One trace containing spans from monitor-worker, incident, notification (including the SMTP client span) and realtime, matching the expected tree |
| 4 | Click a log line's `trace_id` in Loki | Opens the same trace |
| 5 | Messaging dashboard during `docker compose stop notification` | `notification.incident-events` ready count rising; consumer count 0; alert "Consumer missing" firing after 1 min |
| 6 | `curl -s -H "authorization: Bearer $T" localhost:8080/api/v1/analytics/overview?window=24h \| jq .organization` | `uptimePercent` < 100 after the outage; `mttaSeconds` and `mttrSeconds` non-null |
| 7 | Run the [rebuild](../flows/analytics-rebuild.md) with the three verification queries before and after | Identical results (except the current hour) |
| 8 | `SELECT count(*) FROM check_results_default` | 0 |
| 9 | Break the provider: rename `stateVersion` in monitor's event builder; run the provider Pact test | Fails naming the field incident needs; revert |
| 10 | Change a response field in incident without updating the spec | OpenAPI drift test fails |
| 11 | Stop analytics; UI | Analytics panels show an error state; everything else works |
| 12 | `docker stats --no-stream` with every profile | < 4.5 GB |

## Demo script (5 minutes)

1. Outage → incident → email, then open the trace and walk the spans; point at the outbox gap and the broker hop.
2. Messaging dashboard while stopping and restarting notification.
3. Analytics dashboard: downtime, MTTA, MTTR.
4. Rebuild the read model; show the numbers come back identical: "the read model is a disposable projection of the event history".
5. Break a contract deliberately and show the Pact failure.

## Concepts learned

[Distributed tracing](../learning/concepts/distributed-tracing.md) · [CQRS](../learning/concepts/cqrs.md) · [event sourcing](../learning/concepts/event-sourcing.md) (replay to build state) · [contract testing](../learning/concepts/contract-testing.md) · [CAP, PACELC](../learning/concepts/cap-pacelc-eventual-consistency.md) (stale read models, `asOf`) · [partitioning](../learning/concepts/partitioning-and-ordering.md) (table partitioning, stream offsets)

## Interview talking points unlocked

- Trace context propagation over HTTP and through a broker, and the outbox complication.
- RED and USE; metric cardinality.
- CQRS: when it helps, when it is overkill; read-model lag.
- Replaying a log vs rebuilding from sources; retention limits.
- Postgres partitioning, upserts, recomputing aggregates idempotently.
- Consumer-driven contracts vs shared types.

## Shortcuts taken

| Shortcut | Paid off in |
|----------|-------------|
| 100 % sampling | Phase 5 (tail sampling in a separate collector, optional) |
| Ephemeral telemetry storage | Accepted |
| Single analytics consumer | Phase 5 (super streams, optional) |
| PostgreSQL instead of a column store | Phase 5 (ClickHouse projection, optional) |
