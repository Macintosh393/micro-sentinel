# ADR-0007: Use amqplib through a thin in-repo Nest module instead of @nestjs/microservices

- Status: Accepted
- Date: 2026-11-15
- Phase: 2

## Context

NestJS offers `@nestjs/microservices`: one abstraction (`@MessagePattern`, `@EventPattern`, `ClientProxy`) over several transports, including RabbitMQ. The question is whether that abstraction exposes what this design needs and what the author wants to learn, or hides it.

Checked against the current Nest documentation ("RabbitMQ" microservices page, Nest 12) on 2026-11-15:

| Topic | What the Nest RMQ transport does |
|-------|----------------------------------|
| Topology | A server listens on **one queue** (`queue` option). Options `exchange`, `exchangeType`, `exchangeArguments`, `routingKey` and `wildcards: true` allow a topic exchange with `*`/`#` patterns |
| Queue arguments | Passed through `queueOptions` (so `x-queue-type`, `x-dead-letter-exchange` can be set on that one queue) |
| Acks | `noAck` defaults to **`true`** (automatic acks). Manual mode needs `noAck: false` and then `context.getChannelRef().ack(context.getMessage())` in each handler |
| Prefetch | `prefetchCount`, `isGlobalPrefetchCount` |
| Message format | Not specified on that docs page. From the transport's long-standing behaviour: Nest's own JSON wrapper around the payload, with the handler pattern inside the message body next to the data; request/response adds a correlation ID and reply queue. **Confirm on the wire during the Phase 2 spike** |
| Headers and properties | Set with `RmqRecordBuilder`; read through `RmqContext.getMessage()` |
| Driver access | `unwrap()` returns the underlying connection manager |
| Dependencies | `amqplib` and `amqp-connection-manager` |

## Decision drivers

- The design needs: several queues per service with different settings; wait queues and a DLQ per queue; publisher confirms awaited by the outbox relay; single active consumer; stream consumption with `x-stream-offset`; exclusive auto-delete queues; a plain, documented message format.
- The learning goal is acks, redelivery, dead-lettering, prefetch and routing themselves.
- A non-Nest component (a script, the Express fixtures, a future service in another language) must be able to publish and consume.

## Options considered

### A. `@nestjs/microservices` RMQ transport
- Pro: idiomatic Nest; handlers are decorated methods; guards, pipes and interceptors apply; switching transport is a config change.
- Con, each one a feature this design needs that the abstraction does not expose or makes awkward:
  1. **One queue per microservice instance.** incident needs three queues with different prefetch and consumer settings; notification needs three. That means several `connectMicroservice()` calls, each with its own handler namespace.
  2. **Retry topology.** Wait queues and DLQ bindings are not declared by the transport; they must be created out of band or through the unwrapped channel.
  3. **Publisher confirms.** The documented options contain nothing for awaiting the broker's confirm of a batch of published messages, which the outbox relay must do before it marks rows as published.
  4. **Consumer arguments.** Stream consumption needs the `x-stream-offset` consumer argument; there is no option for consumer arguments.
  5. **Message format.** The pattern is inside the body. A consumer in another stack must know Nest's wrapper, and a non-Nest producer must imitate it. The design wants a neutral envelope with the event type in the routing key and AMQP `type` property ([versioning](../contracts/versioning.md)).
  6. **Defaults.** Automatic acks by default is the opposite of what an at-least-once consumer needs; it is one forgotten option away from silent message loss.
- Con: the uniform abstraction is the lowest common denominator across Redis, NATS, MQTT, Kafka and RabbitMQ; it models "send a pattern and data", not exchanges and bindings.

### B. `@golevelup/nestjs-rabbitmq` (9.x)
- Pro: a Nest module designed around RabbitMQ's real model: `@RabbitSubscribe({ exchange, routingKey, queue, queueOptions })`, several queues, per-handler error behaviour, plain JSON bodies, access to the raw message.
- Con: still an abstraction to learn and trust for reconnects, error handling and ack timing; streams and custom confirm handling go through its escape hatches.

### C. `amqplib` directly, wrapped in a small in-repo Nest module (chosen)
- Pro: nothing is hidden. Declaring an exchange, binding a queue, acking and nacking are explicit lines of code.
- Pro: full control over topology, confirms, consumer arguments and message format.
- Pro: `amqplib` 2.x has bundled TypeScript types and built-in connection recovery with backoff and a setup callback, which removes most of the reconnect code that used to justify `amqp-connection-manager`.
- Con: we own the code: recovery edge cases, channel errors, graceful shutdown. Roughly 300–400 lines plus tests.
- Con: handlers are not Nest controllers, so guards/pipes/interceptors do not apply automatically; validation and tracing are called explicitly in the wrapper.

## Decision

Option C. A workspace package `@microsentinel/messaging` provides a Nest dynamic module with four small providers:

| Provider | Responsibility |
|----------|----------------|
| `AmqpConnection` | Connects with amqplib's recovery, re-runs topology setup after a reconnect, exposes a publish (confirm) channel and consume channels, reports health |
| `Topology` | Declarative description of a service's queues, bindings, wait queues and DLQ; asserted at startup |
| `EventPublisher` | `publish(envelope)` on a confirm channel; resolves only when the broker confirms. Used **only** by the outbox relay |
| `Consumer` | `subscribe(queue, handler, { prefetch, … })`; does envelope parsing, payload validation, trace-context extraction, inbox check, and maps the handler's outcome to ack, retry or dead-letter |

Wire format: UTF-8 JSON [envelope](../contracts/versioning.md) as the body; `contentType: application/json`; `messageId` = envelope `id`; `type` = event type; `persistent: true`; W3C `traceparent` in headers.

One sitting in Phase 2 is reserved for a **spike with `@nestjs/microservices`**: build one consumer with the built-in transport, look at the message on the wire in the RabbitMQ management UI, and see the default-ack behaviour first-hand. The spike is thrown away; the knowledge is the deliverable, because "have you used Nest microservices?" is a likely interview question ([NestJS building blocks](../learning/concepts/nestjs-building-blocks.md)).

## Consequences

- Positive: every broker concept in the learning goals is visible in code the author wrote.
- Positive: the wire format is framework-neutral.
- Negative: more code to write, test and get wrong, concentrated in Phase 2, which is already the riskiest phase ([phases](../roadmap/phases.md)).
- Negative: no free Nest pipeline on message handlers.
- Harder later: adopting `@golevelup/nestjs-rabbitmq` is straightforward because the wire format and topology are already standard AMQP.

## Revisit when

- The wrapper exceeds ~500 lines or needs a third reconnect bug fix: switch to `@golevelup/nestjs-rabbitmq`.
- A second transport is really needed: the Nest abstraction's portability starts to matter.

## Learning choice?

**Yes.** A pragmatic team would use `@golevelup/nestjs-rabbitmq`, or `@nestjs/microservices` if their needs were simple request/response or fire-and-forget events on one queue.

## How to explain it in an interview

> "Nest's microservices package gives one abstraction over many transports, which is nice for portability, but with RabbitMQ it's built around one queue per server, defaults to automatic acks, and wraps messages in its own format. I needed several queues per service, dead-letter and retry queues, publisher confirms for my outbox, and stream consumption, and I wanted to actually see acks and redelivery. So I wrapped amqplib in a small Nest module. On a real team I'd probably reach for the golevelup RabbitMQ module instead of maintaining my own."
