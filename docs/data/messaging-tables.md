# Messaging tables (shared DDL)

Two tables appear, with identical structure, in the database of every service that publishes or consumes events. They are created by each service's own migrations (copied from `@microsentinel/messaging`); they are **not** shared tables. Each service has its own.

| Table | In | Purpose | Decision |
|-------|----|---------|----------|
| `outbox` | auth_db, monitor_db, incident_db, notification_db | Events waiting to be published | [ADR-0008](../adr/0008-transactional-outbox.md) |
| `processed_messages` | incident_db, notification_db | Inbox: IDs of messages already handled | [ADR-0012](../adr/0012-idempotent-consumers.md) |

analytics uses neither: it publishes nothing and tracks stream offsets instead ([analytics-schema](analytics-schema.md)).

## `outbox`

```sql
CREATE TABLE outbox (
  id            uuid        PRIMARY KEY DEFAULT uuidv7(),  -- also the envelope "id"
  exchange      text        NOT NULL DEFAULT 'ms.events',
  routing_key   text        NOT NULL,                       -- = envelope "type" for events
  envelope      jsonb       NOT NULL,                       -- the full message body
  headers       jsonb       NOT NULL DEFAULT '{}'::jsonb,   -- traceparent, tracestate
  created_at    timestamptz NOT NULL DEFAULT now(),
  published_at  timestamptz                                 -- NULL = not yet published
);

-- The relay's query: small, hot, only unpublished rows.
CREATE INDEX outbox_unpublished_idx ON outbox (id) WHERE published_at IS NULL;

-- The janitor's query.
CREATE INDEX outbox_published_idx ON outbox (published_at) WHERE published_at IS NOT NULL;
```

Notes:

- **`id` is a UUIDv7**, which sorts by creation time. `ORDER BY id` gives approximate insertion order without a sequence. It is "approximate" because two transactions can commit in a different order than their IDs were generated; consumers never rely on publish order ([ADR-0017](../adr/0017-ordering-and-replay.md)).
- **`exchange`** is `ms.events` for every event. The one exception is notification's internal command, which uses `notification.work`.
- **The partial index** (`WHERE published_at IS NULL`) stays tiny no matter how many published rows exist. This is the reason published rows can be kept for 24 hours for debugging without slowing the relay.
- **`headers.traceparent`** is the trace context that was active when the row was written; the relay continues that trace ([ADR-0025](../adr/0025-observability-stack.md)).

### Relay query

```sql
BEGIN;
SELECT id, exchange, routing_key, envelope, headers
  FROM outbox
 WHERE published_at IS NULL
 ORDER BY id
 LIMIT 100
   FOR UPDATE SKIP LOCKED;
-- publish each row; wait for broker confirms
UPDATE outbox SET published_at = now() WHERE id = ANY($1);
COMMIT;
```

`FOR UPDATE` locks the selected rows until commit. `SKIP LOCKED` makes a second relay skip rows the first has locked instead of waiting, so replicas share the work without publishing the same row concurrently. See [transactional outbox](../learning/concepts/transactional-outbox.md).

### Janitor

```sql
DELETE FROM outbox
 WHERE published_at < now() - interval '24 hours';
```

Runs hourly per service, guarded by `pg_try_advisory_xact_lock` so only one replica does it.

### Metrics derived from this table

| Metric | Query | Alert when |
|--------|-------|-----------|
| `outbox.pending` | `count(*) WHERE published_at IS NULL` | > 500 for 1 min |
| `outbox.oldest_age_seconds` | `now() - min(created_at) WHERE published_at IS NULL` | > 30 s |

## `processed_messages`

```sql
CREATE TABLE processed_messages (
  consumer      text        NOT NULL,          -- the queue name, e.g. 'incident.monitor-events'
  message_id    uuid        NOT NULL,          -- the envelope "id"
  processed_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer, message_id)
);

CREATE INDEX processed_messages_processed_at_idx ON processed_messages (processed_at);
```

Notes:

- The primary key includes `consumer` because one service can consume the same event on two queues for two purposes; each purpose must process it once.
- The insert that detects duplicates:

```sql
INSERT INTO processed_messages (consumer, message_id)
VALUES ($1, $2)
ON CONFLICT DO NOTHING;
-- rowCount = 1 → first time: run the handler in this same transaction
-- rowCount = 0 → duplicate: roll back, broker-ack, done
```

- **It must be in the same transaction as the handler's writes.** If the handler fails and rolls back, the inbox row disappears too, so the retry is processed. If the transaction commits and the broker ack is lost, the redelivery is recognised.

### Janitor

```sql
DELETE FROM processed_messages
 WHERE processed_at < now() - interval '7 days';
```

Seven days is far longer than the longest retry schedule (about 21 minutes for delivery jobs), so a legitimate redelivery never arrives after its inbox row was deleted.

## Why these are not in one shared database

A single shared `outbox` table would be simpler to operate and would break the property the pattern depends on: the outbox insert must be in the **same transaction** as the service's own data, which means the same database. A shared table would also be a shared schema between services ([data ownership](../architecture/data-ownership.md)).

## Migration approach (all services)

- TypeORM migrations, generated then hand-edited, one folder per service: `apps/<service>/src/migrations/` ([ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)).
- File names: `<timestamp>-<what-it-does>.ts`. One logical change per migration.
- Run by the `<service>-migrate` one-shot container before the service starts.
- Every migration has a working `down()` during development. In a real deployment, roll forward instead.
- Partial indexes, `CHECK` constraints, partitions and extensions are written as raw SQL inside the migration, because the entity-diff generator does not produce them.
- Changes follow expand → migrate → contract so that old and new code can run against the same schema during a deploy.
