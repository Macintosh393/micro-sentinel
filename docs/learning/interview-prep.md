# Interview prep

Stories to tell, a system-design version, scale and hindsight answers, and the weak spots an interviewer will find. The 40+ question bank is in [interview-question-bank.md](interview-question-bank.md).

Practise every answer **out loud**, timed. Replace anything you did differently with what you actually did: an interviewer will follow up on the details, and "I planned X but built Y because…" is a strong answer.

## 60 seconds: "Tell me about a project you built"

> I built MicroSentinel, a small self-hosted version of UptimeRobot plus PagerDuty. You register URLs, workers check them, and when one is confirmed down it opens an incident, pages whoever is on call by email, SMS or webhook, escalates if nobody acknowledges, and tracks uptime and time-to-resolve. There's a React dashboard that updates live.
>
> The backend is NestJS services in TypeScript behind a gateway, each with its own Postgres database, talking through RabbitMQ. I built it to learn distributed systems properly, so I focused on the hard parts: a transactional outbox so events are never lost, idempotent consumers so duplicates don't page anyone twice, and Postgres leases with fencing tokens so several workers never apply the same check. Everything runs locally with one `docker compose up`, with OpenTelemetry tracing across services.
>
> I'm also honest that at this scale a modular monolith would be the better engineering choice, and I can explain exactly what would get simpler.

## 2 minutes: architecture walkthrough

Use the verbal explanation in [architecture overview](../architecture/overview.md#the-2-minute-verbal-explanation). Draw while you talk: browser → gateway → services; services → RabbitMQ; each service → its own database. Point at one arrow and say whether it is sync or async and why.

## 10 minutes: deep walkthrough

| Minutes | Topic | Say | Show (if screen-sharing) |
|---------|-------|-----|------|
| 0–1 | Problem and honesty | Uptime + paging; learning project; monolith would win at this size ([ADR-0002](../adr/0002-service-boundaries.md)) | C4 container diagram |
| 1–3 | Boundaries and data | Five contexts; database per service on one server; replicas instead of calls; on-call moved to incident ([ADR-0014](../adr/0014-on-call-in-incident-and-local-replicas.md)) | [Context map](../02-domain-model.md#context-map) |
| 3–5 | The outage flow | Confirmation with hysteresis → state change via outbox → incident opens in one transaction → page request → notification plan/send → realtime hint ([flow](../flows/outage-to-resolution.md)) | The trace in Grafana |
| 5–7 | Reliability | Dual write → outbox; at-least-once → inbox + versions + constraints; retries via TTL wait queues → DLQ; what's still not covered (duplicate email after a crash) | RabbitMQ UI, a DLQ message |
| 7–8 | Coordination | `SKIP LOCKED` leases, fencing token, why a double ping is harmless and a double page isn't ([ADR-0010](../adr/0010-postgres-leases-for-scheduling.md)) | `docker pause` experiment |
| 8–9 | Read side and frontend | Analytics as CQRS read model from streams, rebuildable; React with TanStack Query, SSE hints, token in memory + httpOnly refresh cookie | Dashboard, DevTools cookies |
| 9–10 | Trade-offs and next steps | RabbitMQ vs Kafka; what changes at 100×; what I'd do as a monolith | — |

## System-design version: "Design PagerDuty" (or "design an uptime monitor")

Run it in this order; ask clarifying questions first and write numbers down.

### 1. Requirements (2–3 min)

- Functional: register HTTP checks; detect outages; open incidents; page on-call with escalation; acknowledge/resolve; history and SLA metrics; live dashboard.
- Non-functional: **never miss a page** (durability > latency); avoid duplicate pages; detection within ~1–2 min; multi-tenant; pages delivered within seconds of an incident.
- Out of scope (say it): phone calls, status pages, multi-region, mobile apps.

### 2. Estimates (2 min)

| Quantity | MicroSentinel design | "Real product" scale (state assumptions) |
|----------|---------------------|------------------------------------------|
| Monitors | 500 | 1 M |
| Average interval | 30 s | 60 s |
| Checks/s | ~17 | ~17 000 |
| Check results/day | 1.4 M | 1.4 B |
| Raw storage/day at ~100 B/row | ~150 MB | ~150 GB (needs columnar + compression) |
| Incidents/day | tens | ~10 000s |
| Pages/day | tens | ~100 000 |

The point to make: checks are high-volume and cheap individually; incidents and pages are low-volume and expensive to get wrong. **Design them as two different pipelines.**

### 3. API (2 min)

`POST/GET/PATCH/DELETE /v1/monitors`, `GET /v1/incidents?status=&cursor=`, `POST /v1/incidents/{id}/acknowledge` (idempotent), `PUT /v1/escalation-policy`, `GET /v1/analytics/…`, an SSE stream for live updates. Keyset pagination, problem+json errors. Specs: [openapi](../contracts/openapi/).

### 4. Data (3 min)

- Monitors + schedule/lease (relational).
- Check results: time series, append-only, partitioned by time, short raw retention + rollups.
- Incidents: state machine row + timeline; unique constraint on one open incident per monitor.
- Deliveries: dedupe key unique.
- Who owns what, and why no cross-service joins ([data-ownership](../architecture/data-ownership.md)).

### 5. High-level design (5 min)

Draw: checkers (scaled horizontally, claim work from the scheduler store) → **check-result stream** → analytics; checkers → **state-change events** → incident service (with on-call data local) → **page requests** → notification workers (per channel, retries, DLQ) → providers. Gateway + realtime for the UI.

### 6. Deep dives (pick what the interviewer wants)

| Topic | Key points |
|-------|-----------|
| Scheduling at scale | `SKIP LOCKED` leases work to ~10 k checks/s on one Postgres; beyond: shard monitors by hash into N scheduler partitions, each owned by a worker group (Kafka partitions or consistent hashing); time-wheel scheduling in memory per partition |
| False positives | Consecutive-failure thresholds with fast retries; multi-region quorum ("down in 2 of 3 locations") |
| Never missing a page | Outbox on every hop; durable queues; idempotent consumers; escalation as a durable timer; monitoring the monitor (dead man's switch: alert if checks/min drops to 0) |
| Duplicate pages | Inbox, versions, unique constraints, dedupe keys; residual risk at the provider boundary |
| Notification providers | Per-provider circuit breakers, rate limits, retries with backoff, failover to a second provider, delivery receipts |
| Storage at scale | Columnar store (ClickHouse) for checks; rollups; tiered retention |
| Multi-tenancy | Org ID in every key; per-tenant rate limits; noisy-neighbour isolation (per-tenant check quotas) |

### 7. Failure handling (2 min)

Walk two: "notification is down during an outage" ([service-down-recovery](../flows/service-down-recovery.md)) and "a checker pauses past its lease" ([break-it lab #6](../roadmap/break-it-lab.md)).

## What would you change at 100× scale?

At 50 000 monitors (~1 700 checks/s, 140 M results/day):

| Area | Change | Why |
|------|--------|-----|
| Check results stream | **Kafka or Redpanda** for check results (keep RabbitMQ for work queues, or move everything) | Partitioned, replayable, high-throughput log; consumer groups for analytics |
| Analytics store | **ClickHouse** with materialized views for rollups | 140 M rows/day is beyond comfortable row-store aggregation |
| Scheduling | Partition monitors across scheduler shards; consider in-memory timing wheels per shard with leases per shard instead of per monitor | One claim query per second per worker against one table stops scaling |
| Outbox | CDC (Debezium) instead of polling for monitor | Polling latency and load |
| Databases | Separate PostgreSQL servers per service, with replicas and backups | Failure and performance isolation |
| Broker | Three-node RabbitMQ cluster, quorum queues replicated | Single node is a SPOF |
| Multi-region | Checkers in several regions with quorum confirmation | False positives from one network path |
| Gateway | Traefik/Envoy, TLS, HTTP/2 | Mature edge |
| Notifications | Provider adapters with failover and per-provider rate limiting | Real providers have quotas |
| Observability | Tail sampling, persistent telemetry backends, SLO-based alerting | Volume |
| Kubernetes | Deployments with autoscaling on schedule lag and queue depth | Elasticity |

What would **not** change: the outbox pattern, idempotent consumers, versioned events, fencing, the incident state machine. That's the point of having picked patterns that are broker- and scale-independent.

## What would you do differently as a monolith?

> One NestJS application with five modules (identity, monitoring, incidents, notifications, analytics), each owning its own Postgres schema with no cross-module entity imports, enforced by a lint rule. A separate worker process type for checks, same codebase. Modules talk through an in-process event bus; anything that must survive a crash goes through one outbox table and a job table processed with `SKIP LOCKED`, which also does notification retries. Analytics is a few rollup tables and SQL queries in the same database. No broker, no gateway, no JWKS distribution, no replicas; incident reads users from the identity module's service interface.
>
> I'd keep: leases with fencing for checks, the confirmation logic, the incident state machine, idempotent commands, the SSRF guard, the React app unchanged.
>
> I'd extract a service only when something forced it: the checkers needing to run in several regions would be the first, because that's a genuinely different deployment.

The concrete comparison is Phase 5 stretch [S10](../roadmap/phase-5.md). Background: [monolith guide](concepts/microservices-vs-modular-monolith.md).

## Weak spots and honest answers

| Probe | Honest answer |
|-------|---------------|
| "Isn't this massively over-engineered?" | "Yes, for the problem size, deliberately. I can explain what a monolith version would look like and what it would save. The patterns themselves are the same ones a real system at scale needs." |
| "Your Postgres is a single point of failure." | "Yes. Database-per-service is logical here, not operational. Separate servers is a connection-string change; I'd also need replicas and backups." |
| "Single-node RabbitMQ?" | "Also a SPOF; a quorum queue with one replica gives no redundancy. Production needs three nodes." |
| "Can a page be sent twice?" | "Yes, in one narrow case: the provider accepted it and the process died before recording it. I narrow the window and send an idempotency key; I accept the residual risk because twice is better than never." |
| "Can a page be missed?" | "Not by the pipeline under any single crash, because every hop is durable. It can be *late*: if incident or notification is down, pages wait. And if the checker itself is down, nothing detects outages, which is why 'zero checks in two minutes' is the most important alert." |
| "Your gateway is hand-written." | "As a learning exercise; I'd use Traefik or Envoy in production. Services verify tokens themselves, so swapping it is easy." |
| "A removed member keeps access?" | "For up to 15 minutes, the access-token lifetime. Their refresh token is revoked immediately. A token denylist would close the gap." |
| "Flapping sites page repeatedly." | "Slow flapping does; fast flapping is absorbed by the thresholds. Flap suppression is designed but not built." |
| "Why not Kafka?" | Use the answer in [ADR-0006](../adr/0006-rabbitmq-as-broker.md) and [ADR-0017](../adr/0017-ordering-and-replay.md): work-queue semantics fit; streams cover replay; versions cover ordering; Kafka wins at volume and for per-key ordering with parallelism. |
| "Why TypeORM and not Prisma?" | "It's what most Nest codebases use and what I'm applying for. Prisma or Drizzle would give better type safety; I use raw SQL where TypeORM gets in the way." ([ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)) |
| "Did you write tests?" | Name the integration tests: outbox under broker outage, duplicate delivery, fencing with two workers, DLQ path. Then the Pact and Playwright layers. Be exact about which exist. |
| "Is any of this in production?" | "No, it runs locally. The break-it lab is how I tested failure behaviour; I can show you the notes." |
| "What was hardest?" | Pick from your real experience. Likely: the messaging package (reconnects, confirms, ack timing) or getting trace context through the outbox. |

## Before an interview

1. Run the full demo once, end to end ([phase 4 demo](../roadmap/phase-4.md#demo-script-5-minutes)). Fix anything that rotted.
2. Reread the six ADRs in the [ADR index](../adr/README.md#reading-order-for-interviews) reading list.
3. Do 10 random questions from the [bank](interview-question-bank.md) out loud.
4. Re-read your break-it lab notes; pick two stories.
5. Have the C4 container diagram ready to draw from memory.
