# ADR-0012: Make every consumer idempotent with an inbox table and natural keys

- Status: Accepted
- Date: 2026-11-15
- Phase: 2

## Context

The outbox relay can publish a message twice ([ADR-0008](0008-transactional-outbox.md)). RabbitMQ redelivers any message whose consumer disconnected before acking. Retries re-inject messages later, out of their original order. So every consumer will, sooner or later, see **duplicates** and **reordering**.

### Why "exactly-once delivery" is a myth

A consumer does two things: perform an effect (write to its database) and tell the broker "done" (ack). They are two operations on two systems. A crash can always land between them:

- Effect, then crash before ack → broker redelivers → the effect happens **twice** unless the consumer detects it. (**At-least-once**.)
- Ack, then crash before the effect → the message is gone → the effect happens **zero times**. (**At-most-once**.)

No protocol removes the gap between two systems. What is achievable is **effectively-once processing**: at-least-once delivery plus a consumer whose effect is **idempotent** (doing it twice equals doing it once). When a product advertises "exactly-once", it means this, usually within a boundary where the effect and the progress marker share one transaction.

## Decision drivers

- A duplicate `monitor.state.changed.v1` must not open a second incident; a duplicate `incident.page.requested.v1` must not page twice.
- The mechanism should be uniform so it lives in the shared messaging module.
- It must also be correct under reordering, which deduplication alone does not solve.

## Options considered

### A. Rely on broker features
- RabbitMQ has no built-in consumer-side deduplication for queues. Streams can deduplicate *publishes* per producer name, which does not cover consumer redelivery.
- Rejected: does not exist for our case.

### B. Naturally idempotent operations only
Design each handler so repeating it is harmless: `INSERT … ON CONFLICT DO NOTHING`, `UPDATE … WHERE status = 'OPEN'`, upserts guarded by a version.
- Pro: no extra table; also handles some reordering.
- Con: not every effect is naturally idempotent (appending a timeline row; inserting outbox rows as a consequence). Easy to get subtly wrong handler by handler.

### C. Inbox table: record processed message IDs in the same transaction as the effect
- Pro: uniform; works for any handler; the duplicate check and the effect commit atomically.
- Con: one extra insert per message; the table needs pruning; it handles duplicates but **not** reordering.

### D. Dedupe cache in Redis (`SET id NX EX`)
- Pro: fast; self-expiring.
- Con: not atomic with the database effect. Mark-then-crash loses the message; effect-then-crash-before-mark duplicates it. This reintroduces the gap the pattern exists to close.

### E. Combination of B and C with per-aggregate version checks (chosen)

## Decision

Three layers. Each covers a failure the others miss.

**1. Inbox (duplicates).** Every consumer has a `processed_messages (consumer, message_id, processed_at)` table with primary key `(consumer, message_id)` ([DDL](../data/messaging-tables.md)). The shared `Consumer` wrapper runs each handler like this:

```
BEGIN
  INSERT INTO processed_messages (consumer, message_id) VALUES ($queue, $envelope.id)
    ON CONFLICT DO NOTHING
  if 0 rows inserted → ROLLBACK; broker-ack; return        -- duplicate
  run handler(manager, envelope)                            -- domain writes + outbox inserts
COMMIT
broker-ack
```

- The **idempotency key** is the envelope `id` (a UUIDv7 assigned once, when the event row is written to the producer's outbox; a republish reuses it).
- If the handler throws, the transaction rolls back *including the inbox row*, so a retry is processed normally.
- If the process crashes after `COMMIT` but before the broker ack, the redelivery hits the inbox row and is acked without effect.
- Rows are deleted after 7 days. That is safe because no message is redelivered later than the retry schedule (minutes) and stream replays go through a different path (below).

**2. Version checks (reordering and stale data).**

| Flow | Rule |
|------|------|
| `monitor.state.changed.v1` in incident | Keep `last_state_version` per monitor. Ignore an event whose `stateVersion` ≤ the stored value. |
| Snapshot events (`auth.user.changed.v1`, `auth.membership.changed.v1`, `monitor.definition.changed.v1`) | Upsert only if the incoming `version` > the stored `version`. |
| `incident.*` in analytics | Order-independent by construction: each event sets its own columns (`opened_at`, `acknowledged_at`, `resolved_at`) with an upsert. |

**3. Database constraints (the last line).**

| Constraint | Prevents |
|------------|----------|
| `UNIQUE (monitor_id) WHERE status <> 'RESOLVED'` on `incidents` | Two unresolved incidents for one monitor |
| `UNIQUE (dedupe_key)` on `deliveries` | Two deliveries for the same event, channel and target |
| `UNIQUE (trigger_event_id)` on `incidents` | The same state change opening two incidents |
| Primary key on `check_results (monitor_id, checked_at, check_id)` with `ON CONFLICT DO NOTHING` | Double-counted checks in analytics |

**Stream consumers** (analytics) do not use the inbox. They store the last processed **stream offset** in their own database in the same transaction as the batch, and all their writes are upserts keyed by event or entity ID ([analytics](../services/analytics.md)). The stored offset plays the role of the inbox.

**Effects outside our database** cannot be made atomic with anything. Sending an email and recording "sent" are two systems again. Notification narrows the window and passes the delivery ID to the receiver as an idempotency key; the residual risk (a rare duplicate email after a crash) is accepted and documented ([ADR-0013](0013-notification-owns-delivery-state.md)).

**Realtime** is deliberately not idempotent and not durable: it forwards hints, and a duplicate hint only causes an extra refetch.

## Consequences

- Positive: duplicates and reordering are handled by a rule per flow, written down, with a database constraint behind each critical one.
- Positive: handler authors get deduplication for free and are handed a transaction they cannot forget to use.
- Negative: an extra insert per message and a table to prune per consuming service.
- Negative: handlers must do all their writes through the provided `manager`; a write through another connection escapes the guarantee.
- Harder later: none; this carries over to any at-least-once broker.

## Revisit when

- Inbox inserts show up in profiles (thousands of messages per second): switch hot flows to natural idempotency only.
- A consumer needs to call an external API as part of handling: it needs its own outbox-style step, like notification's delivery jobs.

## Learning choice?

No. This is standard practice wherever at-least-once delivery is used.

## How to explain it in an interview

> "Brokers give you at-least-once delivery, so I assume every message can arrive twice or out of order. Exactly-once delivery isn't really possible because the effect and the ack are two separate operations and you can crash between them. What you can get is effectively-once processing. Each consumer inserts the message ID into a processed-messages table in the same transaction as its changes; if the insert conflicts, it's a duplicate and I just ack. For ordering I carry a version number per aggregate and ignore anything older than what I've seen. And the critical invariants also have unique constraints, like one open incident per monitor, so even a bug in the logic can't produce a duplicate page."
