# ADR-0017: Order with per-aggregate versions and replay from RabbitMQ streams

- Status: Accepted
- Date: 2026-11-15
- Phase: 2 (ordering), 4 (replay)

## Context

Two questions the brief leaves open, and the two things Kafka gives for free that RabbitMQ does not:

1. **Ordering.** Which flows need events processed in order, and how is that achieved?
2. **Replay.** Can analytics throw away its tables and rebuild them from history? A classic or quorum queue **deletes a message once it is acked**, so by default there is no history.

### How ordering really works in RabbitMQ

- A single queue is FIFO: messages are delivered in the order they were enqueued.
- That order survives to processing only with **one consumer handling one message at a time**. Each of these breaks it:
  - **Competing consumers** (several consumers on one queue): messages 1 and 2 go to different consumers; 2 may finish first.
  - **Prefetch > 1 with concurrent handling**: one consumer holds several messages and processes them in parallel.
  - **Requeue or retry**: a nacked message is redelivered later, after messages that were published after it.
  - **Publishing order**: the outbox relay with several replicas does not guarantee publish order across batches ([ADR-0008](0008-transactional-outbox.md)).
- Tools RabbitMQ offers for per-key ordering:
  - **Single active consumer** (`x-single-active-consumer`): several consumers attach, the broker delivers to exactly one, and fails over if it dies. Ordered, but no parallelism.
  - **Consistent-hash exchange** (a plugin shipped with RabbitMQ): routes by a hash of the routing key or a header to one of N queues, so all messages for one key land in the same queue; one consumer per queue. This is hand-built partitioning.
  - **Super streams**: partitioned streams with the same idea.

### How Kafka does it

A topic is split into **partitions**; a message's **key** picks the partition; within a partition order is total; a **consumer group** assigns each partition to exactly one consumer. Per-key ordering with parallelism across keys is the default behaviour, and every partition is a replayable log. This is the real "why Kafka" answer.

## Decision drivers

- Correctness must not depend on a broker configuration detail that someone can change later.
- Volume on the one order-sensitive flow is tiny.
- Analytics rebuild is a stated requirement (FR-A4).
- Memory budget; no second broker.

## Options considered

### Ordering

| Option | Pros | Cons |
|--------|------|------|
| A. Rely on queue order (one consumer, prefetch 1, never retry) | No extra code | Fragile: breaks the moment a second replica, a retry or a parallel relay appears. A blocked message blocks everything behind it |
| B. Consistent-hash exchange → N queues, one consumer each | Ordered per key with parallelism | Plugin to enable; fixed N; rebalancing when N changes; overkill for a few messages per minute |
| C. **Make consumers order-tolerant with a version per aggregate** (chosen) | Correct regardless of delivery order, duplicates or topology; survives a broker change | Each producer maintains a version; each consumer stores "last seen" |
| D. C plus single active consumer and prefetch 1 as a first line of defence (chosen for one queue) | Keeps the common case in order so version checks rarely discard anything | That queue is processed serially |

### Replay

| Option | Pros | Cons |
|--------|------|------|
| E. No replay; fix the read model with ad-hoc scripts | Nothing to build | Violates FR-A4; a bug in a projection is permanent |
| F. Rebuild from source services (analytics calls monitor and incident APIs and pages through history) | No broker feature needed; sources are authoritative | Monitor does not keep check history (that is analytics' data), so checks cannot be rebuilt; adds synchronous dependencies and bulk-export endpoints to every source |
| G. Retain outbox rows forever and re-publish on request | Simple idea | Producers become archives; replay floods *all* consumers unless routed specially; outbox tables grow without bound |
| H. Archive consumer: a service that writes every event to its own table | Queryable history in SQL | One more service and database; replay still needs a re-publish path |
| I. **RabbitMQ streams bound to the events exchange** (chosen) | Built into the broker; non-destructive reads; any consumer can start from any offset; time- and size-based retention; live consumption and replay are the same code path | No log compaction; retention is a hard horizon; one more concept; single consumer per stream in our simple usage |
| J. Add Kafka/Redpanda only for the replayable flows | Purpose-built | Two brokers; ~1 GB more RAM |

## Decision

### Ordering: C everywhere, D on one queue

| Flow | Needs order? | Mechanism |
|------|--------------|-----------|
| `monitor.state.changed.v1` → incident | **Yes, per monitor** (a stale `DOWN` after an `UP` would open a false incident) | `stateVersion` in the event; incident stores `last_state_version` per monitor and ignores anything ≤ it. Queue `incident.monitor-events` uses single active consumer and prefetch 1 |
| Snapshot events → replicas | Latest wins | `version` field; upsert only if newer |
| `incident.*` → notification | No | Each event is acted on independently. When `incident.acknowledged.v1` or `incident.resolved.v1` arrives, still-pending page deliveries for that incident are cancelled; if a page request arrives after them (rare reordering), one late page is sent, which is accepted |
| `incident.*` → analytics | No | Commutative upserts: each event sets its own timestamp column |
| `monitor.check.completed.v1` → analytics | No | Independent facts keyed by check ID |
| `incident.*`, `monitor.state.changed.v1` → realtime | No | Hints; the browser refetches current state |

The envelope carries the version in a common field, `sequence`, so tooling can see it without parsing `data` ([versioning](../contracts/versioning.md)).

### Replay: I

Two streams, declared in the broker definitions file and bound to `ms.events`:

| Stream | Bound routing keys | Retention | Why separate |
|--------|--------------------|-----------|--------------|
| `ms.archive.checks` | `monitor.check.completed.v1` | `x-max-age=35D`, `x-max-length-bytes=2 GB` | High volume, short-lived value |
| `ms.archive.lifecycle` | `monitor.state.changed.v1`, `monitor.definition.changed.v1`, `incident.#`, `auth.#`, `notification.delivery.#` | `x-max-age=1825D` (5 years), `x-max-length-bytes=1 GB` | Low volume; holds snapshots that must not age out, because streams cannot compact to "latest per key" |

- Streams are consumed over ordinary AMQP 0.9.1 with `amqplib`: the consumer sets a prefetch, passes the consumer argument `x-stream-offset` (`first`, or a number), and acks to receive more. Each delivery carries its offset.
- **Analytics reads both streams as its only input**, live and during rebuild. It stores the last processed offset per stream in `analytics_db` in the same transaction as the rows it writes.
- **Rebuild:** stop the consumer, truncate the read-model tables, set stored offsets to "first", start. Procedure and limits: [analytics-rebuild](../flows/analytics-rebuild.md).
- **Bootstrap of replicas** in other services uses `ms.archive.lifecycle` the same way.
- Limits, stated plainly: nothing older than the retention horizon can be replayed; raw checks older than 35 days exist only as rollups in analytics. Retention on raw data (30 days) is shorter than on the stream (35 days) so a rebuild never loses rows that were still visible.

## Consequences

- Positive: no consumer's correctness depends on delivery order.
- Positive: replay needs no new infrastructure, and the queue-versus-log difference is visible inside one broker.
- Positive: a new read model later is "a new consumer starting at offset `first`".
- Negative: version bookkeeping in producers and consumers.
- Negative: no compaction; the lifecycle stream grows for years (kilobytes per day at this scale).
- Negative: this usage reads each stream with one consumer; partitioned parallel reads would need super streams.
- Harder later: at high volume the manual partitioning story (B, super streams) is real work that Kafka would have done for us.

## Revisit when

- An order-sensitive flow exceeds what one serial consumer can handle (~hundreds of messages per second): consistent-hash exchange or super streams, or Kafka.
- A third independent replay consumer appears, or compaction is needed: a log-first broker is the better tool.

## Learning choice?

Partly. Version checks are standard. Using streams as the replay source is a reasonable choice inside a RabbitMQ shop; a pragmatic team with only one small read model might choose option F for incident data and simply not offer rebuild for raw checks.

## How to explain it in an interview

> "RabbitMQ queues are FIFO, but that only gives you ordered processing with a single consumer handling one message at a time, and retries break it anyway. So I don't rely on broker ordering: state-change events carry a version per monitor and the consumer ignores anything older than what it has seen. For replay, normal queues delete messages on ack, so I bind RabbitMQ streams to the same exchange. Streams are append-only logs with offsets, like a Kafka partition. Analytics consumes from them and stores its offset with its data, so I can truncate the read model and replay from the start. That's the honest 'why not Kafka': Kafka gives per-key ordering and replay by default, and I'd switch if those became the dominant need or the volume grew a lot."
