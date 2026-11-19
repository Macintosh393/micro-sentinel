# Phase 2: Introduce the broker

**Effort:** 50–70 h (5–7 weeks). **The phase most likely to overrun**; mitigations and cut list in [phases](phases.md#the-phase-most-likely-to-blow-up-phase-2).

## Goal

Replace the synchronous worker → incident call with events through RabbitMQ, published reliably (outbox) and consumed safely (inbox, versions). Add paging by email and webhooks (notification) and live updates in the browser (realtime + SSE). After this phase, killing any one service delays work but loses nothing.

## Scope

**In:**

| Area | Content |
|------|---------|
| RabbitMQ | `rabbitmq:4.3.7-management-alpine`; `definitions.json` with `ms.events`, `ms.retry`, `ms.dlx`, archive streams (declared now so history accumulates from day one) |
| `contracts` package | Envelope; payload classes for every event in the [catalog](../contracts/event-catalog.md); constants; the catalog/AsyncAPI/contracts check |
| `messaging` package | Connection with recovery, topology declaration, confirm-channel publisher, outbox service + relay, consumer wrapper with inbox, validation, permanent-vs-retryable outcome. Retryable errors in this phase go to a **single** 5 s wait queue; full tiers in Phase 3 |
| monitor | Outbox for all three events; Phase 1 signal call **removed** |
| incident | Consumers for `incident.monitor-events`, `incident.auth-events`, `incident.notification-events`; timeline; outbox; default escalation policy (one level: owner) from `auth.org.created.v1`; `responders` replica; pages at open (level 1 only) |
| auth | Outbox for `auth.*` events; `snapshots:republish` command |
| notification | New: planner + sender for `EMAIL` (Mailpit) and `WEBHOOK` (dev-stubs); deliveries with dedupe key; `recipients` replica; default email contact method; org channels CRUD; cancel-on-ack |
| realtime | New: SSE endpoint, exclusive queue, fan-out, heartbeat, token-expiry close |
| dev-stubs, Mailpit | New containers |
| Trace context | `traceparent` stored in outbox rows and carried in message headers (no SDK yet) |

Events: all in the catalog, except that `incident.escalated.v1` is defined but not yet emitted.

**Deferred:** multi-level escalation and schedules, SMS and chat, retry tiers, circuit breaker, delivery log UI, members, API keys (Phase 3); analytics, tracing SDK (Phase 4).

**Frontend slice:** `useRealtime()` with fetch-based SSE, reconnect with jitter, catch-up refetch, polling fallback, live indicator; toast on new incident; incident timeline on `/incidents/:id`; `/settings/channels` (list, create, edit, delete, test).

## Tasks (one per sitting)

1. **Spike A:** raw amqplib script: assert exchange and queue, publish, consume with manual ack, nack, watch in the management UI. Thrown away.
2. **Spike B:** `@nestjs/microservices` RMQ transport: one `@EventPattern` consumer; inspect the message on the wire and the default ack mode. Write five lines in [ADR-0007](../adr/0007-amqplib-over-nest-microservices.md)'s notes about what you saw. Thrown away.
3. RabbitMQ in Compose: config, definitions file, health check; verify streams exist.
4. `contracts`: envelope, payload classes, constants; the catalog-example validation test.
5. `messaging`: `AmqpConnection` with recovery and topology re-declaration; health indicator.
6. `messaging`: `OutboxService.add(manager, …)`, relay with `SKIP LOCKED` + confirms; integration tests (publish, broker outage, two relays).
7. `messaging`: `Consumer.subscribe` with inbox, validation, outcomes; integration tests (duplicate, crash-before-ack via `CONSUMER_ACK_DELAY_MS`, permanent → DLQ).
8. monitor: outbox rows in the worker transaction and in the API; remove the signal call.
9. incident: `incident.monitor-events` consumer (single active consumer, prefetch 1, version check); timeline; outbox for `incident.*`. **End-to-end checkpoint: an outage opens an incident through the broker.**
10. auth: outbox for `auth.*`; snapshot republish command. incident: `responders` replica and default policy.
11. incident: resolve page targets for level 1 and emit `incident.page.requested.v1`; `incident.notification-events` consumer.
12. notification: skeleton, migrations, `recipients` replica, default email contact method.
13. notification: planner (dedupe key) + sender + email adapter (Mailpit); success/failure events.
14. notification: webhook adapter (HMAC signature, idempotency key, SSRF guard); org channels CRUD and test endpoint; cancel-on-ack.
15. dev-stubs: webhook inbox, chat and SMS endpoints (SMS and chat used in Phase 3), control API, inbox page.
16. realtime: SSE controller, registry, fan-out from an exclusive queue, heartbeat, slow-client guard; gateway route without buffering.
17. web: `useRealtime()`, invalidation map, coalescing, indicator, polling fallback; component tests with a streamed SSE body.
18. web: timeline on incident detail; channels settings.
19. Break-it run: experiments 1, 2, 3, 9 from the [break-it lab](break-it-lab.md); fix what breaks. Measure memory. Record the demo.

## Definition of Done

| # | Check | Expected |
|---|-------|----------|
| 1 | `pnpm ci:local` | Exit 0, including messaging integration tests and the catalog/AsyncAPI/contracts check |
| 2 | `docker compose up -d --wait` | All healthy, including rabbitmq, notification, realtime, mailpit, dev-stubs |
| 3 | `curl -su ms:ms-dev localhost:15672/api/queues \| jq -r '.[].name' \| sort` | Includes `incident.monitor-events`, `notification.incident-events`, `notification.delivery-jobs`, `ms.archive.checks`, `ms.archive.lifecycle`, their `.dlq` queues, and a `realtime.events.<id>` |
| 4 | Outage on `flaky-target` | Incident opens; email in Mailpit (`curl -s localhost:8025/api/v1/messages \| jq '.total'` ≥ 1); webhook in dev-stubs; browser updates without reload |
| 5 | Incident detail timeline | `OPENED`, `PAGE_REQUESTED`, `NOTIFICATION_SENT` in order |
| 6 | **Phase 1's failing experiment again:** stop incident, outage for 2 min, start incident | Incident opens on restart; `opened_at` equals the monitor's `changedAt`. Nothing lost |
| 7 | Stop notification during an outage, acknowledge, start notification | Queue drains; pending pages cancelled (`select status, count(*) from deliveries group by 1` shows `CANCELLED`) ([service-down-recovery](../flows/service-down-recovery.md)) |
| 8 | Stop rabbitmq for 1 min during an outage, start it | `select count(*) from outbox where published_at is null` grows then returns to 0 in monitor_db and incident_db; incident opens late, once |
| 9 | Republish a `monitor.state.changed.v1` from the management UI with the same `message_id` | No second incident; `processed_messages` unchanged count for that ID |
| 10 | Publish a malformed body to `incident.monitor-events` | Lands in `incident.monitor-events.dlq` with `x-ms-last-error` |
| 11 | Two browsers on `/incidents`; acknowledge in one | Other updates in < 2 s |
| 12 | `docker compose stop realtime` | UI indicator "Live updates paused" within ~3 s; list still updates by polling; start realtime → green |
| 13 | `docker stats --no-stream` | Core stack < 3 GB |

## Demo script (4 minutes)

1. Outage → incident appears live → email in Mailpit → webhook in dev-stubs.
2. Open the RabbitMQ UI: show `ms.events`, the bindings, a queue's message rate.
3. Stop notification, cause an outage, show the queue filling; start it, show it drain.
4. Show the outbox table during a broker stop.
5. "A service being down delays work; it doesn't lose it. Each write and its event commit together, and every consumer tolerates duplicates."

## Concepts learned

[Sync vs async](../learning/concepts/sync-vs-async-communication.md) · [event-driven architecture](../learning/concepts/event-driven-architecture.md) · [transactional outbox](../learning/concepts/transactional-outbox.md) · [at-least-once and idempotency](../learning/concepts/idempotency-and-at-least-once.md) · [choreography vs orchestration](../learning/concepts/choreography-vs-orchestration.md) · [partitioning and per-key ordering](../learning/concepts/partitioning-and-ordering.md) · [schema evolution](../learning/concepts/schema-evolution.md) · [real-time delivery](../learning/concepts/realtime-delivery.md) · [CAP, PACELC, eventual consistency](../learning/concepts/cap-pacelc-eventual-consistency.md) · [dead-letter queues](../learning/concepts/dead-letter-queues.md) (basic) · [NestJS microservices transports](../learning/concepts/nestjs-building-blocks.md#microservices-transports)

## Interview talking points unlocked

- The dual-write problem and the outbox; polling vs CDC.
- Why exactly-once delivery is a myth; inbox + versions + constraints.
- Exchanges, routing keys, queues, prefetch, manual acks, publisher confirms.
- Why not Kafka (first version of the answer).
- Local replicas vs synchronous lookups; temporal coupling.
- SSE vs WebSockets vs polling; hints + refetch; reconnect with jitter.
- `@nestjs/microservices` vs amqplib, from having tried both.

## Shortcuts taken

| Shortcut | Paid off in |
|----------|-------------|
| One retry delay (5 s) for every queue | Phase 3 (tiers) |
| One escalation level, no timer | Phase 3 |
| Email and webhook only | Phase 3 (SMS, chat) |
| No circuit breaker | Phase 3 |
| Trace context carried but not exported | Phase 4 |
| Streams accumulate history with no consumer | Phase 4 (analytics) |
| One broker user `ms` for all services | Phase 5 (per-service users with topic permissions) |
