# Resilience

How the system behaves when parts of it are slow or broken: timeouts, retries with backoff and jitter, circuit breakers, bulkheads, backpressure, and graceful shutdown. Concepts: [retries, backoff and jitter](../learning/concepts/retries-backoff-jitter.md), [circuit breakers and bulkheads](../learning/concepts/circuit-breakers-and-bulkheads.md), [backpressure](../learning/concepts/backpressure.md). Experiments: [break-it lab](../roadmap/break-it-lab.md).

## Principles

1. **Every remote call has a timeout.** A call without one can wait forever, and one stuck call holds a socket, a pool slot or a prefetch slot.
2. **Async by default between services** ([communication](../architecture/communication.md)). The broker is the shock absorber: a slow consumer creates a queue, not a cascade.
3. **Retry only what is safe and likely to succeed**: transient errors, idempotent operations. Never retry a `4xx`. Never auto-retry a non-idempotent HTTP call.
4. **Bound everything**: concurrency, queue prefetch, pool sizes, body sizes, retry counts. Unbounded is how a small problem becomes an outage.
5. **Fail the request, not the process.** Errors become problem+json or a routed message; only a broken invariant at startup crashes.

## Timeout budget

| Call | Timeout | Notes |
|------|---------|-------|
| Browser → gateway (UI) | 15 s client-side | UI shows the error state |
| Gateway → upstream | 10 s | `504` on expiry. SSE exempt |
| Service → PostgreSQL | `statement_timeout = 5 s`, connect 3 s, pool acquire 2 s | Set per role in the init script; pool acquire timeout makes pool exhaustion a fast `503`, not a hang |
| Gateway → Redis | 200 ms per command | Then fail open |
| Service → JWKS | 3 s | Cached keys used meanwhile |
| Worker → target | `timeoutMs` (1–30 s), connect 5 s | It is the measurement |
| Notification → receiver | 5 s total | |
| Relay → broker confirms | 5 s per batch | Rollback and retry next tick |
| Handler, per message | 30 s soft limit (logged), no hard kill | Long handlers indicate a bug |

The gateway's 10 s must exceed what any synchronous handler needs (DB 5 s + overhead). Nested timeouts must shrink inward, otherwise the outer one fires first and the inner work keeps running for nobody.

## Retries

| Where | Policy | Jitter |
|-------|--------|--------|
| Message handlers (event consumers) | 4 attempts: 5 s, 30 s, 2 min, via wait queues ([ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)) | Natural (different failure times) |
| Delivery jobs | 5 attempts: 10 s, 1 min, 5 min, 15 min | Natural |
| Broker reconnect (amqplib recovery) | Exponential from 1 s, cap 30 s, unlimited | Built-in jitter |
| JWKS fetch | 3 attempts: 0.5 s, 1 s, 2 s; then rely on cache | Full jitter |
| Outbox relay | Next tick (500 ms); after 5 consecutive failures back off to 5 s | — |
| Phase 1 worker → incident signal | 3 attempts: 1 s, 2 s, 4 s | Full jitter |
| Browser queries | 2 retries on network/`502`/`503`/`504` for `GET` only | TanStack default backoff |
| Browser SSE reconnect | Full jitter, `random(0, min(30 s, 1 s × 2^n))` | Yes ([realtime](../frontend/realtime.md)) |
| Gateway → upstream | **None** | — |

**Full jitter** means the delay is drawn uniformly between 0 and the exponential cap. Without jitter, clients that failed together retry together and knock the recovering service over again (the **thundering herd**).

**Retry budget.** Retries multiply load: 3 layers × 3 attempts = 27 calls for one failure. MicroSentinel has at most one retrying layer per path (the queue *or* the client, never both on the same hop), which is the practical form of a retry budget.

## Circuit breakers

A **circuit breaker** stops calling a dependency that is failing, for a while, so callers fail fast instead of each waiting for a timeout.

| Where | Key | Opens after | Half-open after | While open |
|-------|-----|-------------|-----------------|-----------|
| Notification senders (opossum) | channel type + destination host | 5 consecutive failures, or > 50 % of ≥ 10 calls in 30 s | 30 s, one trial | Fail immediately as retryable; attempt still counted |

Not used elsewhere, on purpose:

- **Between services:** there are no synchronous calls to protect.
- **Worker → targets:** a failing target *is the signal*. Fast-failing it would hide outages. Timeouts and per-host caps already limit the cost.
- **Database:** if the own database is down the service cannot do anything useful; readiness goes false instead.

## Bulkheads

A **bulkhead** isolates resources so one overloaded part cannot sink the rest (named after ship compartments).

| Bulkhead | Isolates |
|----------|----------|
| One process per service, with `mem_limit` | A leak in analytics cannot exhaust incident's memory |
| Separate database per service, separate pool per service | Pool exhaustion in one service does not starve another's connections (the server's `max_connections` is still shared: [ADR-0018](../adr/0018-single-postgres-and-ram-budget.md)) |
| Separate queues per consumer and purpose | A backlog of delivery jobs does not delay incident events |
| Prefetch per queue | Slow sends hold at most 20 slots |
| Per-host concurrency (5) in workers | One slow host cannot take all 50 check slots |
| Per-channel breakers | A dead SMS gateway does not slow email |
| `realtime` separated from domain services | Thousands of browser connections cannot affect incident |

## Backpressure

**Backpressure** is a slow consumer pushing back on a fast producer, so work waits upstream instead of piling up in memory.

| Point | Mechanism |
|-------|-----------|
| Workers | Claim only as many leases as free slots; due monitors wait in the database |
| Consumers | Prefetch: the broker sends no more until acks free credit |
| Streams | Prefetch acts as credit; acks advance the offset |
| Outbox | Relay batches of 100; backlog stays in the table |
| Gateway | Rate limits return `429` with `Retry-After` |
| RabbitMQ itself | Memory watermark (`vm_memory_high_watermark` 0.6 of its limit): above it, publishers are **blocked**; the relay sees it as confirm timeouts and the outbox absorbs it |
| realtime | Slow-client guard closes connections whose buffer exceeds 64 KB |

## Degraded modes

| Condition | Behaviour |
|-----------|-----------|
| Redis down | Rate limiting fails open, `gateway.ratelimit.errors` counted. Chosen: availability over protection for a local system. A public deployment should fail closed on login endpoints |
| auth down | Existing tokens work ≤ 15 min; login impossible ([service-down-recovery](../flows/service-down-recovery.md)) |
| realtime down | UI falls back to polling |
| analytics down | Dashboards show error states for analytics panels only |
| Broker down | Writes succeed; events wait in outboxes; incidents and pages are delayed |
| PostgreSQL down | Everything stops: shared single point of failure, stated in [ADR-0018](../adr/0018-single-postgres-and-ram-budget.md) |

## Health and readiness

| Endpoint | Checks | If failing |
|----------|--------|-----------|
| `/health/live` | Event loop responds | Container should be restarted |
| `/health/ready` | DB `SELECT 1` (≤ 1 s); broker connection up (services with messaging); JWKS loaded; outbox relay ticked in the last 10 s; worker event-loop delay p99 < 200 ms | Not ready: Compose `depends_on` waits; gateway would stop routing in a real load balancer |

Readiness checks **own** dependencies only. A service never reports not-ready because *another service* is down: that would turn one outage into many ([health checks](../learning/concepts/health-checks.md)).

## Graceful shutdown

On `SIGTERM` (`docker stop`, 10 s default grace; set `stop_grace_period: 40s` for workers):

1. Readiness → false.
2. Stop accepting HTTP (Express adapter drains in-flight requests).
3. Cancel broker consumers; let in-flight handlers finish; ack or nack them.
4. Workers: stop claiming; wait up to 35 s for in-flight checks; release leases of anything not started.
5. Relay: finish the current batch.
6. Close broker connection, DB pool, OTel exporter (flush).
7. Exit 0.

Nest's `enableShutdownHooks()` plus `OnApplicationShutdown` implement this. Without it, every deploy would look like a crash: leases expire slowly, unacked messages redeliver, and the last spans are lost.

## Idempotency as a resilience tool

Retries are only safe because effects are idempotent ([ADR-0012](../adr/0012-idempotent-consumers.md)). Resilience and idempotency are one design: you can retry aggressively exactly to the extent that duplicates are harmless.

## What is not covered

No multi-node broker or database, no automatic failover, no load shedding by priority, no chaos automation beyond the [break-it lab](../roadmap/break-it-lab.md). These are the first things to add for production and are listed in [interview prep](../learning/interview-prep.md#what-would-you-change-at-100-scale).
