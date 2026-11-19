# Vision and scope

## The problem

Websites and APIs go down. Someone has to notice, tell the right person quickly, and keep a record of how long it took to fix. Commercial tools split this into two products: uptime monitoring (UptimeRobot, Pingdom) and incident response (PagerDuty, Opsgenie). **MicroSentinel** is a small, self-hosted version of both:

1. Users register URLs as **monitors**.
2. Workers **check** each monitor on a schedule.
3. When a monitor is confirmed down, the system opens an **incident**.
4. It works out who is **on call**, **pages** them (email, SMS, webhooks, chat webhooks), and **escalates** if nobody acknowledges.
5. A React web app shows monitors and incidents live, lets users acknowledge and resolve incidents, and shows uptime, MTTA and MTTR.

All names in bold are defined in the [glossary](03-glossary.md).

## Why it is built as microservices (and why that is partly artificial)

MicroSentinel exists so that its author can learn distributed-systems patterns well enough to explain and defend them in interviews for junior full-stack Node.js roles (NestJS + React, TypeScript).

Be clear about what that means: **at the real size of this system, a modular monolith would be the better engineering choice.** One developer, one deploy, a few hundred monitors: nothing here needs independent scaling or independent teams. The split into services is a **learning choice**, recorded in [ADR-0002](adr/0002-service-boundaries.md), and the [monolith comparison guide](learning/concepts/microservices-vs-modular-monolith.md) explains what a pragmatic team would build instead. Saying this out loud in an interview is a strength, not a weakness.

What makes the domain a good teaching vehicle anyway:

- It has a natural **high-volume stream** (check results) next to a **low-volume, high-value stream** (incidents), which forces real decisions about what to publish.
- A duplicate is sometimes harmless (pinging a URL twice) and sometimes not (paging a person twice at 3 a.m.), which makes idempotency concrete.
- Failure is the product's subject matter, so breaking the system on purpose is on-topic.

## Goals

| # | Goal | How we will know |
|---|------|------------------|
| G1 | A working system: monitor → incident → page → acknowledge → resolve → metrics | The demo script in each [phase](roadmap/phases.md) runs end to end |
| G2 | Every architectural decision is written down with its alternatives | Every design tension has an [ADR](adr/README.md) |
| G3 | Every pattern used is understood, not just copied | Each has a [concept guide](learning/README.md) with interview questions |
| G4 | The system survives the failures it claims to survive | The [break-it lab](roadmap/break-it-lab.md) experiments pass |
| G5 | The whole stack runs offline from one `docker compose up` inside ~8 GB RAM | See [deployment-local](architecture/deployment-local.md) |
| G6 | The project strengthens the fundamentals junior interviews actually test | See the table below |

## Non-goals

- **Production readiness.** No high availability, no backups, no multi-region. Single-node everything.
- **Feature parity with PagerDuty.** No mobile app, no phone calls, no on-call overrides, no maintenance windows, no status pages, no billing.
- **Checks other than HTTP(S).** No TCP, ICMP, DNS, or browser checks.
- **Multi-location checking** in the core phases (it is a Phase 5 stretch).
- **Cloud anything.** No SaaS, no managed services, no accounts, no API keys ([ADR-0028](adr/0028-local-stand-ins.md)).
- **Kubernetes** in the core phases (Phase 5 stretch).
- **A design system.** The UI must be clear and correct, not beautiful.

## Learning objectives mapped to interview topics

The first group is what junior interviews test directly. The second group is the differentiator.

### Fundamentals (expected at junior level)

| Objective | Where the project exercises it | Typical interview question |
|-----------|-------------------------------|----------------------------|
| Node.js event loop and async I/O | The check executor in [Monitor](services/monitor.md) | "What happens if you run a CPU-heavy loop in a request handler?" |
| NestJS modules, DI, guards, pipes, interceptors, filters | Every service; see [NestJS building blocks](learning/concepts/nestjs-building-blocks.md) | "What is the request lifecycle in Nest?" |
| REST API design | [OpenAPI specs](contracts/openapi/), status codes, pagination, idempotent commands | "PUT vs PATCH? When do you return 409?" |
| SQL and PostgreSQL | [Schemas](data/), indexes, transactions, `SKIP LOCKED`, keyset pagination, partitioning | "What is an index? What does a transaction guarantee?" |
| TypeORM | Migrations, repositories vs query builder vs raw SQL ([ADR-0005](adr/0005-typeorm-and-class-validator-usage.md)) | "Why not `synchronize: true`?" |
| React | [Web app](frontend/web-app.md): routing, server state, forms, live updates | "Where do you keep server state? What causes a re-render?" |
| Authentication | [JWT/JWKS](learning/concepts/jwt-and-jwks.md), [browser auth](learning/concepts/browser-authentication.md) | "Where do you store a JWT in the browser and why?" |
| Testing | [Testing strategy](cross-cutting/testing-strategy.md) | "Unit vs integration vs e2e: what goes where?" |
| Docker | [Deployment](architecture/deployment-local.md): multi-stage images, Compose, health checks | "What is the difference between an image and a container?" |

### Distributed systems (bonus depth)

| Objective | Where | Typical interview question |
|-----------|-------|----------------------------|
| Sync vs async communication | [communication.md](architecture/communication.md) | "When would you use a queue instead of an HTTP call?" |
| Transactional outbox | [ADR-0008](adr/0008-transactional-outbox.md) | "How do you update a DB and publish an event atomically?" |
| Idempotency, at-least-once | [ADR-0012](adr/0012-idempotent-consumers.md) | "What if the same message arrives twice?" |
| Retries, backoff, DLQ | [ADR-0023](adr/0023-retry-topology-ttl-dlx.md) | "What is a dead-letter queue?" |
| Distributed coordination | [ADR-0010](adr/0010-postgres-leases-for-scheduling.md) | "How do you stop two workers doing the same job?" |
| CQRS and read models | [ADR-0021](adr/0021-postgres-analytics-store.md), [Analytics](services/analytics.md) | "What is CQRS and when is it overkill?" |
| Data ownership, eventual consistency | [data-ownership.md](architecture/data-ownership.md) | "How do services share data without sharing a database?" |
| Distributed tracing | [observability.md](cross-cutting/observability.md) | "How do you debug a request that crosses five services?" |
| Knowing when *not* to use any of this | Every ADR's "Learning choice?" section | "Would you use microservices for a new startup?" |

## Scope by phase (summary)

| Phase | Theme | Backend | Frontend slice |
|-------|-------|---------|----------------|
| 0 | Foundations | Monorepo, Compose, one empty service behind the gateway, local CI | App shell served through the gateway |
| 1 | Walking skeleton | Auth, Gateway, Monitor, Incident over REST | Login, monitor CRUD, incident list (polling), ack/resolve |
| 2 | Broker | RabbitMQ, outbox, idempotent consumers, Notification, Realtime | Live incident updates (SSE), incident timeline, channels |
| 3 | Resilience and scale | Worker replicas with leases, retries + DLQ, escalation, on-call schedules | On-call and escalation screens, delivery log with redrive |
| 4 | Observability and CQRS | OpenTelemetry everywhere, Analytics read model, contract and e2e tests | Uptime and SLA dashboards |
| 5 | Stretch | kind + Helm, k6 load tests, multi-location checks, ClickHouse swap | Optional |

Full detail: [roadmap/phases.md](roadmap/phases.md).

## Success criteria for the author

By the end of Phase 4 you can, without notes:

1. Draw the container diagram from memory and explain every arrow as sync or async, and why.
2. Explain what happens, step by step, when a monitored site goes down while the Notification service is offline.
3. Answer "why not Kafka?", "why not Prisma?", and "why not a monolith?" with the trade-offs, including where the alternative would have been better.
4. Point at a trace in Grafana and narrate it.
5. Name three things you would do differently next time ([interview prep](learning/interview-prep.md)).
