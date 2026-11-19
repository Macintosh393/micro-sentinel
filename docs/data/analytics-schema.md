# analytics_db schema

Owner: [analytics](../services/analytics.md). Role: `analytics_svc`.

Everything here is a **read model**: derived from events, never the source of truth, safe to truncate and rebuild ([analytics-rebuild](../flows/analytics-rebuild.md)). Store choice: [ADR-0021](../adr/0021-postgres-analytics-store.md).

No `outbox` (analytics publishes nothing) and no `processed_messages` (stream offsets do that job).

## Tables

```sql
CREATE TABLE monitors_dim (                    -- dimension: what a monitor_id means
  monitor_id          uuid    PRIMARY KEY,
  org_id              uuid    NOT NULL,
  name                text    NOT NULL,
  url                 text    NOT NULL,
  deleted             boolean NOT NULL DEFAULT false,
  definition_version  integer NOT NULL
);
CREATE INDEX monitors_dim_org_idx ON monitors_dim (org_id) WHERE NOT deleted;

CREATE TABLE check_results (                   -- fact: one row per check
  monitor_id      uuid        NOT NULL,
  checked_at      timestamptz NOT NULL,
  check_id        uuid        NOT NULL,
  org_id          uuid        NOT NULL,
  outcome         text        NOT NULL CHECK (outcome IN ('SUCCESS', 'FAILURE')),
  status_code     smallint,
  latency_ms      integer,
  failure_reason  text,
  PRIMARY KEY (monitor_id, checked_at, check_id)
) PARTITION BY RANGE (checked_at);

-- One partition per day, created ahead of time by a job:
CREATE TABLE check_results_2026_11_15 PARTITION OF check_results
  FOR VALUES FROM ('2026-11-15 00:00+00') TO ('2026-11-16 00:00+00');
-- A catch-all so an unexpected timestamp never fails an insert:
CREATE TABLE check_results_default PARTITION OF check_results DEFAULT;

CREATE TABLE check_rollups_hourly (            -- aggregate: one row per monitor per hour
  monitor_id      uuid        NOT NULL,
  bucket          timestamptz NOT NULL,        -- start of the hour, UTC
  org_id          uuid        NOT NULL,
  checks_total    integer     NOT NULL,
  checks_success  integer     NOT NULL,
  latency_avg_ms  integer,                     -- over successful checks
  latency_p95_ms  integer,
  latency_max_ms  integer,
  computed_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (monitor_id, bucket)
);
CREATE INDEX check_rollups_org_bucket_idx ON check_rollups_hourly (org_id, bucket);

CREATE TABLE dirty_buckets (                   -- hours whose rollup must be recomputed
  monitor_id  uuid        NOT NULL,
  bucket      timestamptz NOT NULL,
  PRIMARY KEY (monitor_id, bucket)
);

CREATE TABLE incident_facts (                  -- one row per incident
  incident_id      uuid        PRIMARY KEY,
  org_id           uuid        NOT NULL,
  monitor_id       uuid        NOT NULL,
  opened_at        timestamptz,
  acknowledged_at  timestamptz,
  resolved_at      timestamptz,
  resolution       text,
  max_escalation_level smallint NOT NULL DEFAULT 1
);
CREATE INDEX incident_facts_org_opened_idx     ON incident_facts (org_id, opened_at);
CREATE INDEX incident_facts_monitor_opened_idx ON incident_facts (monitor_id, opened_at);

CREATE TABLE stream_offsets (                  -- how far each stream has been processed
  stream       text        PRIMARY KEY,        -- 'ms.archive.checks' | 'ms.archive.lifecycle'
  last_offset  bigint      NOT NULL,           -- -1 = nothing processed yet
  updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE analytics_state (                 -- one row: projection state survives restarts
  id          smallint    PRIMARY KEY DEFAULT 1 CHECK (id = 1),
  state       text        NOT NULL DEFAULT 'RUNNING' CHECK (state IN ('RUNNING', 'REBUILDING', 'STOPPED')),
  changed_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE quarantined_events (              -- messages that could not be projected
  stream          text        NOT NULL,
  stream_offset   bigint      NOT NULL,
  event_type      text,
  error           text        NOT NULL,
  body            jsonb,
  quarantined_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (stream, stream_offset)
);
```

## Design notes

**Why partition `check_results`.** Partitioning splits one logical table into physical child tables by a key. It buys two things here:

1. **Retention is `DROP TABLE check_results_2026_10_15`**, which is instant and leaves no dead rows. Deleting 1.4 M rows with `DELETE` would write that many tombstones, bloat the table and keep autovacuum busy.
2. **Partition pruning**: a query for the last 24 hours only touches two partitions.

The partition key must be part of the primary key, which is why `checked_at` is in it.

**Idempotent ingestion.** A batch from `ms.archive.checks` is one transaction:

```sql
BEGIN;
INSERT INTO check_results (…) VALUES (…), (…), …
ON CONFLICT DO NOTHING;                               -- duplicate events are ignored

INSERT INTO dirty_buckets (monitor_id, bucket)
SELECT DISTINCT monitor_id, date_trunc('hour', checked_at) FROM (VALUES …) v
ON CONFLICT DO NOTHING;

UPDATE stream_offsets SET last_offset = $last, updated_at = now()
 WHERE stream = 'ms.archive.checks';
COMMIT;
-- then broker-ack the batch
```

The offset and the data commit together. After a crash the consumer resubscribes from `last_offset + 1`; anything re-read is absorbed by `ON CONFLICT`. That is effectively-once processing ([ADR-0012](../adr/0012-idempotent-consumers.md)).

**Rollups are recomputed from raw rows**, never incremented:

```sql
WITH todo AS (
  DELETE FROM dirty_buckets
   WHERE (monitor_id, bucket) IN (
           SELECT monitor_id, bucket FROM dirty_buckets
            ORDER BY bucket LIMIT 500 FOR UPDATE SKIP LOCKED)
  RETURNING monitor_id, bucket)
INSERT INTO check_rollups_hourly
       (monitor_id, bucket, org_id, checks_total, checks_success,
        latency_avg_ms, latency_p95_ms, latency_max_ms)
SELECT c.monitor_id, t.bucket, min(c.org_id::text)::uuid,
       count(*),
       count(*) FILTER (WHERE c.outcome = 'SUCCESS'),
       avg(c.latency_ms)  FILTER (WHERE c.outcome = 'SUCCESS'),
       percentile_cont(0.95) WITHIN GROUP (ORDER BY c.latency_ms)
                          FILTER (WHERE c.outcome = 'SUCCESS'),
       max(c.latency_ms)  FILTER (WHERE c.outcome = 'SUCCESS')
  FROM todo t
  JOIN check_results c
    ON c.monitor_id = t.monitor_id
   AND c.checked_at >= t.bucket AND c.checked_at < t.bucket + interval '1 hour'
 GROUP BY c.monitor_id, t.bucket
ON CONFLICT (monitor_id, bucket) DO UPDATE
   SET checks_total = EXCLUDED.checks_total, checks_success = EXCLUDED.checks_success,
       latency_avg_ms = EXCLUDED.latency_avg_ms, latency_p95_ms = EXCLUDED.latency_p95_ms,
       latency_max_ms = EXCLUDED.latency_max_ms, computed_at = now();
```

This one statement is a compact tour of useful SQL: a data-modifying CTE, `SKIP LOCKED`, `FILTER`, an ordered-set aggregate, and an upsert. Recomputing is idempotent; incrementing would double-count on any replay.

**Incident facts are commutative upserts.** Each `incident.*` event sets only its own columns:

```sql
-- incident.acknowledged.v1
INSERT INTO incident_facts (incident_id, org_id, monitor_id, acknowledged_at)
VALUES ($1, $2, $3, $4)
ON CONFLICT (incident_id) DO UPDATE SET acknowledged_at = EXCLUDED.acknowledged_at;
```

Whichever order `opened`, `acknowledged` and `resolved` arrive in, the row ends up the same. No version check is needed.

**`monitors_dim`** uses the versioned snapshot upsert (`WHERE monitors_dim.definition_version < EXCLUDED.definition_version`). Deleted monitors stay as rows with `deleted = true` so history remains labelled.

## Queries behind the API

| Endpoint | Query shape |
|----------|-------------|
| Uptime per bucket | `SELECT bucket, sum(checks_success)::float / sum(checks_total) … FROM check_rollups_hourly WHERE monitor_id = $1 AND bucket BETWEEN … GROUP BY date_trunc($unit, bucket)` |
| Organization overview | Uptime from `check_rollups_hourly` by `org_id`; incident count, `avg(acknowledged_at − opened_at)`, `avg(resolved_at − opened_at)` from `incident_facts` |
| Latency series | `latency_avg_ms`, `latency_p95_ms` from `check_rollups_hourly` |
| Current hour | The last, still-changing bucket is included; its rollup is at most ~60 s old |

MTTA excludes incidents never acknowledged; MTTR excludes unresolved incidents. Both are stated in the API description and the UI.

## Partition maintenance

A job runs every hour under `pg_try_advisory_xact_lock(hashtext('analytics.partitions'))`:

1. Create partitions for today + 2 days if missing.
2. Drop partitions older than 30 days.
3. If `check_results_default` contains rows, log a warning (a timestamp fell outside every prepared partition).

## Retention

| Data | Rule |
|------|------|
| `check_results` | 30 days, by dropping partitions |
| `check_rollups_hourly` | 13 months |
| `incident_facts`, `monitors_dim` | Kept |
| `quarantined_events` | 30 days |

Raw retention (30 d) is shorter than the checks stream's retention (35 d), so a rebuild can always restore every raw row that existed before it ([ADR-0017](../adr/0017-ordering-and-replay.md)). Rollups older than the stream horizon **cannot** be rebuilt and are preserved, not truncated, during a rebuild.

## Migrations

`apps/analytics/src/migrations/`, run by `analytics-migrate`. The partitioned table and its first partitions are raw SQL in the migration. Approach: [messaging tables](messaging-tables.md#migration-approach-all-services).
