# MicroSentinel

A small, self-hosted **uptime monitor and incident-response system**: a learning-sized UptimeRobot + PagerDuty. Users register URLs; workers check them; when one is confirmed down the system opens an incident, pages whoever is on call (email, SMS, webhooks, chat webhooks), escalates if nobody acknowledges, and tracks uptime, MTTA and MTTR. A React dashboard shows everything live.

It is built as **NestJS microservices in TypeScript** around RabbitMQ and PostgreSQL, and runs entirely locally with one `docker compose up`: no accounts, no API keys, no cloud services.

**Why it exists:** to learn distributed systems well enough to explain and defend every decision in interviews for junior full-stack Node.js roles (NestJS + React). The documentation is both an engineering plan and a curriculum. It says plainly where microservices are overkill for a system this size ([ADR-0002](docs/adr/0002-service-boundaries.md)).

**Status:** design and planning documents only. No application code yet. Implementation starts with [Phase 0](docs/roadmap/phase-0.md).

## Start here

| If you want… | Read |
|--------------|------|
| The one-page picture and the 2-minute explanation | [Architecture overview](docs/architecture/overview.md) |
| Why each decision was made | [ADR index](docs/adr/README.md) |
| What to build next, step by step | [Roadmap](docs/roadmap/phases.md) |
| To learn a concept | [Learning map](docs/learning/README.md) |
| To prepare for an interview | [Interview prep](docs/learning/interview-prep.md), [question bank](docs/learning/interview-question-bank.md) |
| To break things on purpose | [Break-it lab](docs/roadmap/break-it-lab.md) |
| What is still undecided | [Open questions](docs/open-questions.md) |

## Suggested reading order

1. [Vision and scope](docs/00-vision-and-scope.md) → [requirements](docs/01-requirements.md) → [domain model](docs/02-domain-model.md) → [glossary](docs/03-glossary.md)
2. [Architecture overview](docs/architecture/overview.md) → [C4 context](docs/architecture/c4-context.md) → [C4 containers](docs/architecture/c4-containers.md) → [communication](docs/architecture/communication.md) → [data ownership](docs/architecture/data-ownership.md)
3. The ADRs for the design tensions, in index order ([ADR index](docs/adr/README.md))
4. [Outage to resolution](docs/flows/outage-to-resolution.md), then the other [flows](docs/flows/)
5. The [service docs](docs/services/) and [frontend docs](docs/frontend/web-app.md)
6. [Roadmap](docs/roadmap/phases.md) and [Phase 0](docs/roadmap/phase-0.md)
7. Concept guides as you reach each phase ([learning map](docs/learning/README.md))

### One week before starting Phase 0 (about 10 hours)

| Day | Time | Read | Self-check |
|-----|------|------|------------|
| 1 | 1.5 h | Vision, requirements, glossary; [monolith guide](docs/learning/concepts/microservices-vs-modular-monolith.md); [ADR-0002](docs/adr/0002-service-boundaries.md) | Explain in 60 s why this is microservices and why a monolith would be better |
| 2 | 1.5 h | Domain model; overview; C4 containers; communication; [sync vs async](docs/learning/concepts/sync-vs-async-communication.md); [ADR-0014](docs/adr/0014-on-call-in-incident-and-local-replicas.md) | Draw the container diagram from memory; label each arrow sync or async |
| 3 | 1.5 h | Data ownership; [database per service](docs/learning/concepts/database-per-service.md); [outbox](docs/learning/concepts/transactional-outbox.md); [ADR-0008](docs/adr/0008-transactional-outbox.md); [messaging tables](docs/data/messaging-tables.md) | Explain the dual-write problem and the relay query |
| 4 | 1.5 h | [Idempotency](docs/learning/concepts/idempotency-and-at-least-once.md); [ADR-0012](docs/adr/0012-idempotent-consumers.md); [ordering](docs/learning/concepts/partitioning-and-ordering.md); [ADR-0017](docs/adr/0017-ordering-and-replay.md); [ADR-0006](docs/adr/0006-rabbitmq-as-broker.md) | Answer "why not Kafka?" and "what if a message arrives twice?" |
| 5 | 1.5 h | [Outage flow](docs/flows/outage-to-resolution.md); [locks and fencing](docs/learning/concepts/distributed-locks-leases-fencing.md); [ADR-0010](docs/adr/0010-postgres-leases-for-scheduling.md); [Node event loop](docs/learning/concepts/node-event-loop.md); [monitor](docs/services/monitor.md) | Walk the outage flow step by step; explain fencing |
| 6 | 1.5 h | [JWT](docs/learning/concepts/jwt-and-jwks.md); [browser auth](docs/learning/concepts/browser-authentication.md); [ADR-0015](docs/adr/0015-jwt-rs256-jwks-validate-everywhere.md); [ADR-0016](docs/adr/0016-browser-session.md); [web app](docs/frontend/web-app.md) | Explain where tokens live and why there is no CORS |
| 7 | 1 h | [NestJS building blocks](docs/learning/concepts/nestjs-building-blocks.md); [repo layout](docs/cross-cutting/repo-layout.md); [deployment](docs/architecture/deployment-local.md); [Phase 0](docs/roadmap/phase-0.md); [open questions](docs/open-questions.md) Q1–Q5 | Decide the open questions; start Phase 0 task 1 |

## Documentation map

```
README.md                              this file
docs/
  00-vision-and-scope.md               problem, goals, non-goals, learning objectives → interview topics
  01-requirements.md                   user stories, functional requirements, NFRs with numbers, assumptions
  02-domain-model.md                   bounded contexts, context map, aggregates, invariants, incident state machine
  03-glossary.md                       ubiquitous language and naming conventions (single source of truth)
  open-questions.md                    decisions needing input, with applied defaults; risks
  architecture/
    overview.md                        one-page picture + 2-minute verbal explanation
    c4-context.md                      C4 level 1
    c4-containers.md                   C4 level 2: every container
    communication.md                   every edge: sync or async, why, timeouts; broker topology
    data-ownership.md                  who owns what, replicas, consistency guarantees, retention
    deployment-local.md                Compose layout, profiles, ports, health, RAM, images, Windows/WSL2
    tech-stack.md                      every tool: pinned version, reason, image
  frontend/
    web-app.md                         routes, screens → endpoints, caching, state, four states, phases
    auth-in-the-browser.md             login, refresh rotation, CORS, CSRF, XSS
    realtime.md                        SSE, reconnection, missed-event catch-up
  services/
    gateway.md  auth-org.md  monitor.md  incident.md  notification.md  analytics.md  realtime.md
  contracts/
    event-catalog.md                   every event: producer, consumers, key, ordering, example
    asyncapi.yaml                      AsyncAPI 3.0 description of the channels
    versioning.md                      envelope, headers, compatibility rules, HTTP error format
    openapi/                           auth, monitor, incident, notification, analytics, realtime (OpenAPI 3.1)
    proto/README.md                    why there are no .proto files, plus a gRPC exercise
  data/
    messaging-tables.md                outbox and inbox DDL shared by all services; migration approach
    auth-schema.md  monitor-schema.md  incident-schema.md  notification-schema.md  analytics-schema.md
  flows/
    register-monitor.md  outage-to-resolution.md  flapping-and-duplicates.md
    notification-retry-and-dlq.md  service-down-recovery.md  analytics-rebuild.md
  adr/
    README.md                          index + design tension → ADR table
    0001 … 0028                        one decision each
  cross-cutting/
    observability.md  resilience.md  security.md  testing-strategy.md  configuration.md  repo-layout.md
  roadmap/
    phases.md                          overview, estimates, risk, concept → phase
    phase-0.md … phase-5.md            one file per phase
    break-it-lab.md                    12 failure experiments
  learning/
    README.md                          concept map and study order
    concepts/                          29 concept guides
    interview-prep.md                  60 s / 2 min / 10 min stories, system design, 100×, monolith, weak spots
    interview-question-bank.md         80 questions by topic, junior vs bonus, linked
prompts/
  01-planning-docs.md                  the brief these documents were written from
```

### Deviations from the requested structure

| Requested | Here | Why |
|-----------|------|-----|
| `roadmap/phases.md` containing all phases | `phases.md` (overview) + `phase-0.md` … `phase-5.md` | Each phase has tasks, a runnable Definition of Done, a demo, concepts, talking points and shortcuts; one file would exceed ~400 lines |
| `learning/interview-prep.md` containing the question bank | `interview-prep.md` + `interview-question-bank.md` | Same size limit; the bank is used separately for drills |
| Six service docs | Seven: added `services/realtime.md` | [ADR-0019](docs/adr/0019-sse-realtime-service.md) introduced a realtime service that pushes to browsers so no domain service has to |
| `data/<service>-schema.md` | Plus `data/messaging-tables.md` | Outbox and inbox DDL is identical in four databases; defined once |
| `contracts/proto/` with `.proto` files | `contracts/proto/README.md` only | [ADR-0014](docs/adr/0014-on-call-in-incident-and-local-replicas.md) removed the only gRPC call; no synchronous service-to-service call remains |
| `contracts/openapi/` per service with an HTTP API | Six specs, including realtime (SSE) | Gateway has no API of its own beyond routing; its routes are documented in [gateway](docs/services/gateway.md) |

## The stack in one table

Full detail, versions verified on 2026-11-15: [tech-stack](docs/architecture/tech-stack.md).

| Component | Choice | Version | Runs as |
|-----------|--------|---------|---------|
| Runtime / language | Node.js / TypeScript | 26.13.1 / 6.0.3 | `node:26.13.1-bookworm-slim` |
| Backend framework | NestJS (ESM projects) | 12.5.0 | built images |
| ORM / validation | TypeORM / class-validator | 1.1.3 / 0.15.1 | — |
| Broker | RabbitMQ (quorum queues, streams) | 4.3.7 | `rabbitmq:4.3.7-management-alpine` |
| Database | PostgreSQL (one server, database per service) | 18.7 | `postgres:18.7` |
| Cache (rate limits) | Redis | 8.8.4 | `redis:8.8.4-alpine` |
| Frontend | React + Vite + React Router + TanStack Query + Mantine | 19.3.1 / 8.3.8 / 8.5.1 / 5.107.1 / 9.8.1 | `nginx:1.30.6-alpine` |
| Observability | OpenTelemetry → Grafana LGTM | SDK 0.301.2 | `grafana/otel-lgtm:0.41.0` |
| Email stand-in | Mailpit | 1.33.1 | `axllent/mailpit:v1.33.1` |
| Chaos | Toxiproxy | 2.13.0 | `ghcr.io/shopify/toxiproxy:2.13.0` |
| Monorepo | pnpm workspaces | 12.21.0 | — |
| Tests | Vitest, Supertest, Testcontainers, Pact, Testing Library, MSW, Playwright | see tech-stack | — |

## Conventions

- **Names** come from the [glossary](docs/03-glossary.md) and the [event catalog](docs/contracts/event-catalog.md). Events: `<service>.<entity>.<past-tense-verb>.v<major>`. Queues: `<consumer>.<purpose>`. Tables: `snake_case`. JSON: `camelCase`.
- **Diagrams** are Mermaid, so they render on GitHub.
- **Learning choices** (decisions made mainly to learn a pattern) are labelled in every ADR with what a pragmatic team would do instead.
- **No application code** in these docs: contracts, DDL, type sketches and short pseudocode only.
- Commit messages follow Conventional Commits (`docs: …`, later `feat(incident): …`).

## Consistency rules

Rerun these at the end of every phase. They were checked when the documents were first written.

1. Every event named in a flow or service doc exists in the [event catalog](docs/contracts/event-catalog.md) and [asyncapi.yaml](docs/contracts/asyncapi.yaml) with the same name and fields.
2. Every service's published and consumed events match the catalog's producer and consumer lists.
3. Every synchronous call in a flow appears in an [OpenAPI spec](docs/contracts/openapi/).
4. Every design tension has an ADR; the [ADR index](docs/adr/README.md) is complete.
5. No document contradicts an accepted ADR.
6. Every concept guide is linked from at least one service, flow or ADR doc, and links back.
7. All relative links resolve.
8. Each phase's Definition of Done is checkable by running something.
9. Every screen in [web-app](docs/frontend/web-app.md) gets its data from a documented gateway endpoint or the realtime channel.
10. Every component in [tech-stack](docs/architecture/tech-stack.md) is a pinned, self-hostable image or package; nothing requires an external account or API key.
