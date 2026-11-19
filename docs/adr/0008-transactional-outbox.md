# ADR-0008: Publish events through a transactional outbox with a polling relay

- Status: Accepted
- Date: 2026-11-15
- Phase: 2

## Context

When a monitor is confirmed down, the worker must do two things: update `monitor_db` and publish `monitor.state.changed.v1`. These are two different systems (PostgreSQL and RabbitMQ) with no shared transaction. This is the **dual-write problem**.

| Order | Failure | Result |
|-------|---------|--------|
| Commit DB, then publish | Crash or broker outage after the commit | Monitor is `DOWN` in the database; **no event**; no incident, nobody paged. Silent and permanent |
| Publish, then commit DB | Commit fails or crashes after publishing | Event says `DOWN`; database says `UP`. An incident for something that, per our own records, never happened |

Retrying in memory does not help: the process that knows it must retry is the one that crashed. Every service that publishes events has this problem (auth, monitor, incident, notification).

## Decision drivers

- A lost `monitor.state.changed.v1` or `incident.page.requested.v1` means a missed page: the worst failure this product can have.
- Duplicates are tolerable because consumers are idempotent ([ADR-0012](0012-idempotent-consumers.md)).
- Memory budget and operational simplicity.
- It is one of the most common distributed-systems interview questions.

## Options considered

### A. Publish after commit and hope ("best effort")
- Pro: trivial.
- Con: loses events on every crash or broker blip between the two steps. Unacceptable here.

### B. Distributed transaction (two-phase commit, XA)
- Pro: atomic across both systems in theory.
- Con: RabbitMQ does not take part in XA transactions; 2PC blocks when the coordinator fails; nobody builds new systems this way.

### C. Transactional outbox with a **polling relay** (chosen)
The service writes the event as a row in an `outbox` table **in the same database transaction** as the domain change. A background loop (the **relay**) reads unpublished rows, publishes them, and marks them published.
- Pro: atomicity comes from the database's own transaction. If the change committed, the event row exists; if it rolled back, it does not.
- Pro: no new infrastructure; plain SQL; easy to observe (`SELECT count(*) … WHERE published_at IS NULL`).
- Con: at-least-once: a crash after publishing but before marking republishes the row.
- Con: latency up to one poll interval; constant polling load on the database.
- Con: with several relay instances, strict global order is not guaranteed.

### D. Transactional outbox with **change data capture** (Debezium)
A CDC tool reads the database's write-ahead log and publishes outbox inserts.
- Pro: no polling; lower latency; preserves commit order; no relay code in the service.
- Con: Debezium is a JVM service (Kafka Connect, or Debezium Server with a RabbitMQ sink): another container of several hundred MB, logical replication slots to configure and monitor, and a new failure mode (a stalled slot makes PostgreSQL retain WAL until the disk fills).
- Con: more moving parts than this system's ~20 events per second justify.

### E. Event sourcing (the event store *is* the database)
- Pro: no dual write at all for state; events are the source of truth.
- Con: publishing to the broker is still a second write, and it changes the whole persistence model. Rejected for incident in [ADR-0011](0011-incident-state-machine-not-event-sourcing.md).

### F. Listen-to-yourself (publish first, update state by consuming your own event)
- Pro: one write.
- Con: the service cannot read its own write immediately; awkward for an API that must return the created resource.

## Decision

Option C in every publishing service, implemented once in `@microsentinel/messaging`.

**Write side.** `OutboxService.add(manager, envelope)` inserts into `outbox` using the caller's transaction ([ADR-0005](0005-typeorm-and-class-validator-usage.md)). The row stores the full envelope, the routing key and the current trace context. DDL: [messaging tables](../data/messaging-tables.md).

**Relay.** One loop per process, every 500 ms (immediately again if the last batch was full):

```
BEGIN
  rows = SELECT id, routing_key, envelope, headers
           FROM outbox
          WHERE published_at IS NULL
          ORDER BY id
          LIMIT 100
          FOR UPDATE SKIP LOCKED
  for each row: publish to ms.events on a confirm channel (persistent)
  wait for all broker confirms (timeout 5 s)
  UPDATE outbox SET published_at = now() WHERE id = ANY(confirmed ids)
COMMIT
```

- `FOR UPDATE SKIP LOCKED` lets several replicas run the relay without publishing the same row at the same time and without waiting on each other.
- If a confirm is missing or the broker is down, the transaction rolls back and the rows are retried on the next tick. Events queue up in the table; nothing is lost.
- A crash after confirms but before `COMMIT` republishes those rows: a duplicate with the same envelope `id`, which consumers discard.
- The `id` column is a UUIDv7, so `ORDER BY id` is approximately insertion order. Exact order across concurrent transactions and relay replicas is **not** guaranteed; consumers that care use version numbers ([ADR-0017](0017-ordering-and-replay.md)).
- Published rows are deleted after 24 hours by a janitor job.

**Where the relay runs.** In every process of auth, incident and notification; in `monitor-worker` only for the monitor service (the API process writes rows; workers publish them).

**High-volume events too.** `monitor.check.completed.v1` also goes through the outbox. The worker already runs a transaction per check; one more insert is cheap at 17 per second, and analytics gets a complete record.

## Consequences

- Positive: no lost events under any single crash or broker outage. The broker being down for ten minutes only delays events.
- Positive: outbox depth (`published_at IS NULL`) is a direct health metric.
- Negative: every consumer must be idempotent. This is a requirement, not an optimisation.
- Negative: up to ~500 ms added latency; a steady trickle of polling queries.
- Negative: a relay that stops silently delays everything. Mitigation: readiness includes the relay loop's last successful tick; alert on outbox age.
- Harder later: at high volume, polling becomes the bottleneck and CDC becomes attractive.

## Revisit when

- Outbox inserts exceed ~1 000 per second or p95 publish latency must be under 100 ms: move to CDC, or `LISTEN/NOTIFY` to wake the relay.
- Strict publish order per aggregate becomes necessary: run the relay as a single leader ([leader election](../learning/concepts/leader-election.md)).

## Learning choice?

No. The polling outbox is the standard pragmatic solution. (Sending high-volume check results through it is conservative; a team under load would publish those directly and accept rare loss.)

## How to explain it in an interview

> "The problem is a dual write: I update my database and publish an event, and either can fail independently, so I can end up with a state change nobody hears about. I use a transactional outbox: the event is inserted into an outbox table in the same transaction as the business change, so they commit or roll back together. A relay polls that table with `FOR UPDATE SKIP LOCKED`, publishes with publisher confirms, and marks rows as published. It can publish twice if it crashes between confirm and commit, so delivery is at-least-once and consumers have to be idempotent. The alternative is CDC with Debezium, which avoids polling but adds a JVM service and replication slots; not worth it at my volume."
