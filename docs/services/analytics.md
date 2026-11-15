# analytics (Analytics / SLA)

**Purpose:** answer "how reliable has this been?" from a read model built entirely from events. **Bounded context:** Analytics, read-only and conformist ([domain model](../02-domain-model.md)).

Decision records: [ADR-0021](../adr/0021-postgres-analytics-store.md) (store), [ADR-0017](../adr/0017-ordering-and-replay.md) (streams, replay), [ADR-0012](../adr/0012-idempotent-consumers.md) (offsets as inbox). Concept: [CQRS](../learning/concepts/cqrs.md).

## Responsibilities

- Consume `ms.archive.checks` and `ms.archive.lifecycle` and project them into query-friendly tables.
- Maintain hourly rollups of uptime and latency.
- Serve uptime %, latency series, incident counts, MTTA and MTTR.
- Enforce retention on its own data (partitions, rollups).
- Rebuild itself from the streams on request.

## Explicitly not responsible for

- **Being a source of truth for anything.** If analytics and a source service disagree, the source is right.
- **Writing anything other services depend on.** It publishes no events.
- **Real-time status.** "Is it down now?" is monitor's answer; analytics is seconds behind by design.
- **Alerting on SLA breaches** (not in scope).
- **Reading other services' databases.** Events only ([ADR-0018](../adr/0018-single-postgres-and-ram-budget.md)).

## Owned data

`analytics_db`: `monitors_dim`, `check_results` (daily partitions), `check_rollups_hourly`, `dirty_buckets`, `incident_facts`, `stream_offsets`, `quarantined_events`. All disposable. Schema and SQL: [analytics-schema](../data/analytics-schema.md).

## Sync APIs

**Exposed** ([analytics.yaml](../contracts/openapi/analytics.yaml)):

| Endpoint | For |
|----------|-----|
| `GET /v1/analytics/overview?window=` | Dashboard: organization summary + one row per monitor |
| `GET /v1/analytics/monitors/{monitorId}/summary?window=` | Monitor page header |
| `GET /v1/analytics/monitors/{monitorId}/series?from=&to=&bucket=` | Charts |
| `GET /internal/v1/analytics/status` | Offsets, lag, state (not routed by the gateway) |
| `POST /internal/v1/analytics/rebuild` | Start a rebuild (not routed by the gateway) |

Every public response includes `asOf` so the UI can show how fresh the numbers are.

**Consumed:** JWKS from auth (cached).

## Events

**Consumed** (from streams, not queues):

| Stream | Events projected | Into |
|--------|------------------|------|
| `ms.archive.checks` | `monitor.check.completed.v1` | `check_results`, `dirty_buckets` |
| `ms.archive.lifecycle` | `monitor.definition.changed.v1` | `monitors_dim` |
| | `incident.opened.v1`, `.acknowledged.v1`, `.escalated.v1`, `.resolved.v1` | `incident_facts` |
| | everything else on the stream | skipped |

**Published:** none.

## Internal components

| Component | Job |
|-----------|-----|
| `StreamConsumer` (×2) | Subscribes with `x-stream-offset = last_offset + 1` (or `first`), prefetch 500 / 100; collects messages into a batch (up to the prefetch, or 250 ms); hands the batch to a projector; acks the batch after commit |
| `CheckProjector` | One transaction: bulk insert `ON CONFLICT DO NOTHING`, mark dirty buckets, update offset |
| `LifecycleProjector` | One transaction: per-event upserts (dimension by version; incident facts by column), update offset |
| `RollupJob` | Every 60 s: recompute up to 500 dirty buckets from raw rows |
| `PartitionJob` | Hourly: create upcoming partitions, drop those older than 30 days |
| `QueryService` | The read queries; picks hourly rows and aggregates to the requested bucket |
| `RebuildService` | Stop consumers → truncate → reset offsets → start; state `REBUILDING` |

### Why a batch per transaction

One transaction per message at 17 messages/s is fine; during a rebuild it is millions of messages. Batching turns 500 commits into one, and the offset stored in the same transaction makes a crash safe: the batch either fully applied with its offset, or not at all. This is the stream equivalent of the inbox pattern.

### Poison events

A message that fails validation or throws a non-transient error is written to `quarantined_events` with its offset and **skipped**; the batch continues. A stream cannot dead-letter, and stopping on one bad event would stall all analytics. A transient error (database unavailable) aborts the batch; the consumer restarts from the stored offset after a delay with backoff.

## Scaling model

**One active consumer.** The offset-in-database design assumes a single reader per stream; a second replica would process everything twice (harmlessly, thanks to upserts, but wastefully) and fight over offsets. If replicas are run for API capacity, consumption is guarded by a session-level advisory lock so only one consumes; jobs use transaction-level advisory locks. Scaling consumption itself would need partitioned streams (super streams) and one consumer per partition ([ADR-0017](../adr/0017-ordering-and-replay.md)). Locally: one replica, behind the `analytics` profile.

## Failure modes

| Dependency down | Effect |
|-----------------|--------|
| PostgreSQL | API `503`; consumers back off; nothing lost (the stream keeps everything) |
| RabbitMQ | API serves existing data, getting staler; `asOf` shows it |
| A source service | Fewer events arrive; numbers for that area stop moving |

**When analytics is down, or not started at all:** nothing else is affected. Events accumulate in the streams regardless (they are retained whether or not anyone reads them). On start it resumes from its stored offset and catches up. The dashboard's analytics panels show an error state; the rest of the UI works.

**During a rebuild:** public endpoints return `503` `READ_MODEL_REBUILDING` with `Retry-After`. Flow: [analytics-rebuild](../flows/analytics-rebuild.md).

## Observability

| Kind | Items |
|------|-------|
| Metrics | `analytics.stream.lag_seconds` per stream (now − `time` of last projected event); `analytics.stream.offset`; `analytics.batch.size`, `analytics.batch.duration`; `analytics.events.skipped` by type; `analytics.events.quarantined`; `analytics.dirty_buckets`; `analytics.rollup.duration`; `analytics.query.duration` by endpoint |
| Spans | `ms.archive.checks process` — one span per **batch** with span **links** to the producers' trace contexts (a batch has many parents); HTTP server spans for queries |
| Log fields | `stream`, `offset_from`, `offset_to`, `batch_size`, `state` |

## Patterns applied

[CQRS](../learning/concepts/cqrs.md) · [event sourcing](../learning/concepts/event-sourcing.md) (replaying a log to build state, without the write side being event-sourced) · [partitioning and per-key ordering](../learning/concepts/partitioning-and-ordering.md) · [idempotency](../learning/concepts/idempotency-and-at-least-once.md) · [leader election](../learning/concepts/leader-election.md) (advisory locks for singleton jobs) · [CAP and eventual consistency](../learning/concepts/cap-pacelc-eventual-consistency.md) · [backpressure](../learning/concepts/backpressure.md) (prefetch as credit)

## Framework and Nest structure

**NestJS 12.**

| Nest element | Use |
|--------------|-----|
| Modules | `ProjectionModule` (consumers, projectors), `RollupModule`, `QueryModule` (controllers), `MaintenanceModule` (partitions, rebuild), `HealthModule`; shared `MessagingModule` (stream consume only), `AuthGuardModule`, `ObservabilityModule` |
| Data access | Almost entirely raw SQL through `manager.query()`: bulk inserts, upserts, aggregates. Entities exist mainly so migrations and typing have something to hold on to ([ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)) |
| Guards | `JwtAuthGuard` global; `RolesGuard('ADMIN')` on `/internal` |
| Pipes | Global `ValidationPipe`; a `TimeRangePipe` enforcing `from < to` and the maximum range per bucket |
| Interceptors | Logging/timing; a `RebuildingInterceptor` returning `503` while state is `REBUILDING` |
| Exception filter | `ProblemDetailsFilter` |
| Scheduling | `@nestjs/schedule` intervals for rollup and partition jobs, each wrapped in an advisory-lock check |

## Phase introduced

Phase 4.

## Related

Flows: [analytics-rebuild](../flows/analytics-rebuild.md). Frontend: [web-app](../frontend/web-app.md) dashboards.
