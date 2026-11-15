# ADR-0006: Use RabbitMQ as the message broker

- Status: Accepted (fixed by the author; comparison recorded here)
- Date: 2026-11-15
- Phase: 2

## Context

Services communicate through events ([communication](../architecture/communication.md)). The broker is fixed as RabbitMQ. This ADR compares it honestly with the alternatives and states how the design uses RabbitMQ's strengths and works around its weaknesses. The detailed ordering and replay analysis, the heart of "why not Kafka?", is [ADR-0017](0017-ordering-and-replay.md).

Two models to keep apart:

- A **message queue** (RabbitMQ classic and quorum queues): the broker routes each message to queues; a message is **removed when a consumer acks it**. The broker tracks what each consumer has processed. Smart broker, simple consumers.
- A **log** (Kafka, Redpanda, RabbitMQ streams): messages are appended to an ordered, partitioned log and **kept** for a retention period. Each consumer tracks its own position (**offset**) and can rewind. Simple broker, smart consumers.

## Decision drivers

- What the workloads need: work queues with per-message retry and dead-lettering (notification); pub/sub fan-out (incident events); replay (analytics); per-key ordering (monitor state changes).
- Memory: the whole stack has ~8 GB.
- Learning value for the target roles.
- Node.js client quality.

## Options considered

| | RabbitMQ 4.x (fixed) | Kafka / Redpanda | NATS JetStream | Redis Streams |
|-|----------------------|------------------|----------------|---------------|
| Model | Queues + exchanges; streams as an extra queue type | Partitioned log | Subjects + persistent streams with consumers | Log per key with consumer groups |
| Routing | Rich: direct, topic, fanout, headers exchanges | By topic; filtering is the consumer's job | Subject wildcards | None |
| Per-message ack, retry, dead-letter | **Native**: nack, dead-letter exchanges, TTL, delivery limits | Not native: you build retry topics and a DLQ topic; a failed message blocks its partition unless skipped | Native ack/nak with redelivery and max deliveries | Pending-entries list; manual claim logic |
| Replay | Only with streams | **Native**, the core feature | Native | Native until trimmed |
| Per-key ordering at scale | Possible but manual: single active consumer, or consistent-hash exchange | **Native**: key → partition, one consumer per partition in a group | Per subject | Per stream |
| Consumer scaling | Competing consumers on a queue; trivial | Consumer groups; bounded by partition count; rebalancing | Pull consumers | Consumer groups |
| Throughput ceiling | Tens of thousands msg/s | Hundreds of thousands to millions | Very high | High, memory-bound |
| Memory (single node, idle) | ~150–400 MB | Kafka ~1 GB+ (JVM); Redpanda ~1 GB default, tunable lower | ~30–50 MB | Already running, but data is in RAM |
| Node.js clients | `amqplib`: mature, simple | `kafkajs` (maintenance slowed), `@confluentinc/kafka-javascript` | `nats` official | `ioredis` |
| Operations | Moderate | Heavy (Kafka), moderate (Redpanda) | Light | Light, but durability depends on Redis persistence settings |

### Where each alternative would have been the better call

- **Kafka or Redpanda** if the dominant need were the check-result stream: high-volume, replayable, ordered per monitor, feeding several analytical consumers. Replay and per-key ordering would be free instead of designed. The price here: more RAM, and notification's retry and dead-letter flow would have to be built from extra topics.
- **NATS JetStream** on simplicity and footprint alone: it covers work queues, pub/sub and replay in one small binary. It loses on ecosystem: fewer jobs, fewer interview questions, less tooling.
- **Redis Streams** only for a system that must not add a broker at all. Durability and dead-lettering are do-it-yourself.

### Why RabbitMQ fits this system anyway

- Two of the three hard flows (notification retries with delays and DLQ; fan-out of incident events to independent consumers) are what RabbitMQ is built for.
- The third (replayable history) is covered by **streams**, a log-style queue type inside RabbitMQ, which also makes the queue-versus-log distinction something you can see in one broker.
- At ~17 messages per second, throughput is irrelevant.
- AMQP concepts (exchange, binding, routing key, ack, prefetch, dead-letter exchange) are common interview material for Node back-end roles.

## Decision

RabbitMQ 4.3 (`rabbitmq:4.3.7-management-alpine`), single node, used as follows:

| Need | RabbitMQ feature | Detail |
|------|------------------|--------|
| Publish events | One durable **topic exchange** `ms.events`; routing key = event type | [communication](../architecture/communication.md) |
| Durable work queues | **Quorum queues** (Raft-replicated; one replica locally) with `x-delivery-limit` | [ADR-0023](0023-retry-topology-ttl-dlx.md) |
| No lost publishes | **Publisher confirms** from the outbox relay; persistent messages | [ADR-0008](0008-transactional-outbox.md) |
| No lost consumes | **Manual broker acks**, bounded **prefetch** | [ADR-0007](0007-amqplib-over-nest-microservices.md) |
| Delayed retry, DLQ | TTL wait queues + dead-letter exchanges | [ADR-0023](0023-retry-topology-ttl-dlx.md) |
| Per-monitor ordering | Version numbers in events; **single active consumer** on one queue | [ADR-0017](0017-ordering-and-replay.md) |
| Replay | **Streams** `ms.archive.checks`, `ms.archive.lifecycle` | [ADR-0017](0017-ordering-and-replay.md) |
| Browser fan-out | Exclusive auto-delete queue per realtime instance | [ADR-0019](0019-sse-realtime-service.md) |

Shared exchanges and the two streams are declared in `infra/rabbitmq/definitions.json`, loaded at broker start. Consumer queues are declared by their consumers.

## Consequences

- Positive: retries, delays and dead-lettering are configuration, not code.
- Positive: routing is flexible; a new consumer binds a queue without any producer change.
- Negative: replay and per-key ordering had to be designed (ADR-0017) instead of inherited.
- Negative: a single-node broker is a single point of failure, and a one-replica quorum queue gives no replication benefit. Accepted locally; stated in [deployment-local](../architecture/deployment-local.md).
- Negative: streams have no log compaction ("keep only the latest message per key"), so low-volume snapshot events are kept in full on a long-retention stream.
- Harder later: moving to Kafka changes the consumer model (offsets, partitions, consumer groups) and the retry design. The envelope, the outbox and the idempotent consumers carry over unchanged, which is the main reason they are broker-agnostic.

## Revisit when

- Check volume passes ~5 000 messages per second, or more than three independent consumers need replay: a log-first broker fits better.
- Per-key ordering is needed on a high-volume flow with many parallel consumers.

## Learning choice?

No. RabbitMQ is a reasonable production choice for this workload. Using streams for replay (instead of just rebuilding analytics from source APIs) leans toward learning; see ADR-0017.

## How to explain it in an interview

> "RabbitMQ, because most of what I needed was task-queue behaviour: per-message acks, retries with delay, dead-letter queues, and flexible routing so several services can react to one event. Kafka's strengths are replay and per-key ordering at very high volume. I needed a little of both, so I use version numbers in events for ordering and a RabbitMQ stream as a replayable archive for the analytics read model. If check volume were a thousand times higher, or many consumers needed replay, I'd pick Kafka or Redpanda and build the retry topics myself."
