# ADR-0021: Store the analytics read model in PostgreSQL with partitions and rollups

- Status: Accepted
- Date: 2026-11-15
- Phase: 4

## Context

Analytics answers "how reliable has this been?": uptime %, latency, incident counts, MTTA, MTTR, per monitor and per organization, over 24 hours to 13 months. It is the read side of **CQRS** (command query responsibility segregation): data is written by other services in shapes suited to their transactions, and analytics keeps a separate copy shaped for queries, fed by events ([CQRS guide](../learning/concepts/cqrs.md)).

The data:

| Dataset | Volume at design scale | Shape |
|---------|------------------------|-------|
| Check results | ~1.44 M rows/day (500 monitors, 30 s); ~43 M rows for 30 days | Append-only time series |
| Hourly rollups | 12 000 rows/day; ~4.7 M for 13 months | Small, pre-aggregated |
| Incident facts | Tens per day | Tiny |

At the volume this project will actually see locally (20 monitors at 10 s) it is ~170 000 rows per day.

## Decision drivers

- Dashboard queries under 200 ms.
- Memory: the `analytics` profile should add ~200 MB, not ~1 GB.
- The read model must be rebuildable ([ADR-0017](0017-ordering-and-replay.md)).
- SQL and PostgreSQL skills are tested in junior interviews; column-store internals are not.
- The store should be replaceable without touching other services.

## Options considered

| | Plain PostgreSQL (partitions + rollups) | TimescaleDB | ClickHouse |
|-|------------------------------------------|-------------|------------|
| What it is | The database already running | A PostgreSQL extension for time series | A column-oriented analytical database |
| Fit for time series | Good with declarative partitioning and manual rollup tables | Very good: automatic partitioning (hypertables), continuous aggregates, compression, retention policies | Excellent: built for aggregations over billions of rows |
| Extra memory | 0 (same server) | 0–small if it replaces the server image; it is a different image for the shared server | ~500 MB–1 GB minimum |
| Query language | SQL | SQL | SQL dialect with different semantics (no transactions, eventual merges, different upsert model) |
| Ops and concepts | Partition maintenance and rollup jobs are ours to write | Policies do it for us; some features are under the Timescale licence rather than Apache 2 | New system: MergeTree engines, materialized views, a separate client, different idempotency approach |
| Idempotent writes | `ON CONFLICT`, transactions with the offset | Same as PostgreSQL | Possible (ReplacingMergeTree, dedup tokens) but different |
| Ceiling | Comfortable to tens of millions of rows per table with partitioning; hundreds of millions gets painful | Billions with compression | Far beyond anything here |
| Learning value for target roles | High: partitioning, indexes, `GROUP BY`, window functions, `EXPLAIN` | Medium: mostly "it is done for you" | High for data engineering, low for junior full-stack |

Where an alternative would be the better call: **ClickHouse** from roughly 50–100 million raw rows per day, or when ad-hoc analytical queries over raw data matter. **TimescaleDB** when a team is committed to PostgreSQL and wants retention, compression and continuous aggregates without writing them.

## Decision

**Plain PostgreSQL 18**: database `analytics_db` on the shared server ([ADR-0018](0018-single-postgres-and-ram-budget.md)).

| Table | Purpose | Key design |
|-------|---------|-----------|
| `check_results` | Raw checks, 30 days | Range-partitioned by day on `checked_at`; primary key `(monitor_id, checked_at, check_id)`; inserts use `ON CONFLICT DO NOTHING` |
| `check_rollups_hourly` | Uptime and latency per monitor per hour, 13 months | Primary key `(monitor_id, bucket)`; filled by `INSERT … SELECT … GROUP BY … ON CONFLICT DO UPDATE` |
| `incident_facts` | One row per incident with `opened_at`, `acknowledged_at`, `resolved_at` | Upserted column by column from `incident.*` events |
| `monitors_dim` | Monitor name, URL, organization | Versioned snapshot upsert |
| `stream_offsets` | Last processed offset per stream | Updated in the same transaction as each batch |

DDL: [analytics-schema](../data/analytics-schema.md).

**Rollups are recomputed, not incremented.** Every 60 s a job recomputes the hourly buckets touched since its last run (tracked by a small "dirty buckets" set) directly from `check_results`. Recomputing a bucket is **idempotent**: running it twice, or after duplicate inserts were ignored, gives the same numbers. Incrementing counters per event would double-count on any redelivery. Daily and longer views are computed from hourly rows at query time (at most ~9 500 rows per monitor for 13 months).

**Partition maintenance** (create tomorrow's partition, drop those older than 30 days) and rollup jobs run under `pg_try_advisory_xact_lock`, so that if analytics ever has two replicas only one runs each job ([leader election](../learning/concepts/leader-election.md)).

**p95 latency** per bucket is computed with `percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms)` during rollup. Percentiles cannot be combined from other percentiles, so the daily view shows the maximum of hourly p95 values and says so in the UI.

**Replaceability.** Analytics exposes a small read API ([OpenAPI](../contracts/openapi/analytics.yaml)); nothing else knows how the data is stored. Swapping in ClickHouse is a Phase 5 stretch exercise that touches only this service: a second projection reading the same streams, then a switch.

## Consequences

- Positive: no new infrastructure; transactions make offset-plus-batch writes atomic, which keeps the consumer simple and effectively-once.
- Positive: exercises partitioning, upserts, aggregation and `EXPLAIN`: directly useful SQL.
- Negative: analytics shares CPU, memory and I/O with the transactional databases. A heavy dashboard query can slow incident. This is exactly the interference a separate analytical store exists to prevent.
- Negative: partition and rollup jobs are code we maintain.
- Negative: row storage is far less compact than columnar; 43 M rows of raw checks is several GB with indexes at full design scale (not reached locally).
- Harder later: none; the read model is disposable by design.

## Revisit when

- Raw checks exceed ~50 M rows per day, or a dashboard query's p95 exceeds 500 ms after indexing.
- Analytics queries measurably affect other services' latency: move `analytics_db` to its own server first, then consider ClickHouse.
- Users want ad-hoc slicing of raw data.

## Learning choice?

No. This is the pragmatic choice at this size. (ClickHouse would have been the learning choice, at the cost of 1 GB and time better spent elsewhere.)

## How to explain it in an interview

> "Analytics is a CQRS read model: it consumes events and keeps its own tables shaped for dashboard queries. I used plain Postgres: raw check results partitioned by day with thirty-day retention, plus hourly rollups that I recompute from the raw rows rather than increment, so reprocessing an event never double-counts. I looked at ClickHouse and TimescaleDB. At a million and a half rows a day Postgres is comfortable, it's already running, and I can store the stream offset in the same transaction as the data. Because the read model can be rebuilt from the event stream, switching to ClickHouse later only touches that one service."
