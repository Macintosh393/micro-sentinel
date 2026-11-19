# Architecture Decision Records

One file per significant decision. Format and rules: [ADR-0001](0001-record-architecture-decisions.md). An accepted ADR is not edited; a changed decision gets a new ADR that supersedes it.

**Learning choice** = made mainly to learn the pattern; the ADR says what a pragmatic team would do instead.

## Index

| # | Title | Status | Decision in one line | Learning choice? |
|---|-------|--------|----------------------|------------------|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | Keep numbered ADRs in the repo, with phase, revisit triggers and an interview answer | No |
| [0002](0002-service-boundaries.md) | Split into services along bounded contexts | Accepted | Five domain services plus gateway and realtime; a modular monolith would be the pragmatic choice | **Yes** |
| [0003](0003-monorepo-pnpm-workspaces.md) | One repository with pnpm workspaces | Accepted | Monorepo, plain pnpm workspaces, no Nx/Turborepo yet | No |
| [0004](0004-nestjs-everywhere-express-for-fixtures.md) | NestJS everywhere, Express for fixtures | Accepted | Nest 12 (ESM) for all services; plain Express only for `flaky-target` and `dev-stubs` | Partly |
| [0005](0005-typeorm-and-class-validator-usage.md) | TypeORM and class-validator usage | Accepted (tools fixed) | Migrations only; raw SQL where it fits; explicit transactions; tolerant validation of events | No |
| [0006](0006-rabbitmq-as-broker.md) | RabbitMQ as the broker | Accepted (fixed) | Topic exchange, quorum queues, confirms, streams; Kafka would win on replay and per-key ordering at volume | No |
| [0007](0007-amqplib-over-nest-microservices.md) | amqplib over `@nestjs/microservices` | Accepted | A thin in-repo Nest module over amqplib so acks, topology and confirms stay visible | **Yes** |
| [0008](0008-transactional-outbox.md) | Transactional outbox, polling relay | Accepted | Events are rows written in the business transaction; a `SKIP LOCKED` relay publishes with confirms | No |
| [0009](0009-check-results-and-state-changes.md) | Publish checks and state changes; confirm in Monitor | Accepted | Two event types; consecutive-result thresholds with hysteresis live in Monitor | No |
| [0010](0010-postgres-leases-for-scheduling.md) | PostgreSQL leases with fencing | Accepted | `FOR UPDATE SKIP LOCKED` claims plus a lease token checked on write; no Redis locks | No |
| [0011](0011-incident-state-machine-not-event-sourcing.md) | State machine + timeline, not event sourcing | Accepted | Current-state row with optimistic locking and an append-only timeline in the same transaction | No |
| [0012](0012-idempotent-consumers.md) | Idempotent consumers | Accepted | Inbox table in the handler's transaction, per-aggregate versions, unique constraints | No |
| [0013](0013-notification-owns-delivery-state.md) | Notification owns delivery state | Accepted | Notification has a database: deliveries with a unique dedupe key; plan and send stages | No |
| [0014](0014-on-call-in-incident-and-local-replicas.md) | On-call in Incident; local replicas | Accepted | No synchronous calls between services; schedules and escalation in Incident; replicas from snapshot events | Partly |
| [0015](0015-jwt-rs256-jwks-validate-everywhere.md) | RS256 + JWKS, verified everywhere | Accepted | Auth signs; gateway and every service verify with cached public keys | Partly |
| [0016](0016-browser-session.md) | Browser session | Accepted | Access token in memory, rotating refresh token in an httpOnly SameSite=Strict cookie, single origin | No |
| [0017](0017-ordering-and-replay.md) | Ordering and replay | Accepted | Version numbers instead of relying on queue order; RabbitMQ streams as the replayable archive | Partly |
| [0018](0018-single-postgres-and-ram-budget.md) | One PostgreSQL server, RAM budget | Accepted | One server, a database and role per service; optional components behind Compose profiles | No |
| [0019](0019-sse-realtime-service.md) | SSE from a Realtime service | Accepted | Polling in Phase 1; then SSE hints from a small event-fed service; browser refetches | Partly |
| [0020](0020-nestjs-api-gateway.md) | NestJS API gateway | Accepted | A thin Nest reverse proxy with JWT check and Redis rate limiting; Traefik in real life | **Yes** |
| [0021](0021-postgres-analytics-store.md) | PostgreSQL for analytics | Accepted | Partitioned raw table plus recomputed hourly rollups; ClickHouse deferred | No |
| [0022](0022-contracts-package-and-shared-libs.md) | Contracts package and shared libraries | Accepted | Share event contracts and technical plumbing; never domain code; apps never import apps | No |
| [0023](0023-retry-topology-ttl-dlx.md) | Retry topology | Accepted | TTL wait queues + dead-letter exchanges + delivery limit; no delayed-message plugin | No |
| [0024](0024-frontend-stack.md) | Frontend stack | Accepted | Vite SPA, React Router, TanStack Query, Mantine, React Hook Form + zod, generated API types | No |
| [0025](0025-observability-stack.md) | Observability stack | Accepted | OpenTelemetry in every service; one `grafana/otel-lgtm` container; trace context through the outbox | No |
| [0026](0026-pinger-safety.md) | Pinger safety | Accepted | Resolve-validate-connect SSRF guard; concurrency cap tied to lease claims; never read bodies | No |
| [0027](0027-testing-stack.md) | Testing stack | Accepted | Vitest everywhere, Testcontainers for real PostgreSQL/RabbitMQ, in-repo Pact on two events, Playwright | Partly |
| [0028](0028-local-stand-ins.md) | Local stand-ins | Accepted | Mailpit, `dev-stubs` and `flaky-target` replace every external dependency | No |

## Design tensions from the brief → ADR

| # | Tension | ADR |
|---|---------|-----|
| 1 | The dual-write problem | [0008](0008-transactional-outbox.md) |
| 2 | What Monitor publishes; flap detection; confirmation | [0009](0009-check-results-and-state-changes.md) |
| 3 | Avoiding double pings across replicas | [0010](0010-postgres-leases-for-scheduling.md) |
| 4 | Event sourcing in Incident | [0011](0011-incident-state-machine-not-event-sourcing.md) |
| 5 | Idempotency and "exactly once" | [0012](0012-idempotent-consumers.md) |
| 6 | "Stateless" Notification | [0013](0013-notification-owns-delivery-state.md) |
| 7 | The on-call lookup; who owns schedules | [0014](0014-on-call-in-incident-and-local-replicas.md) |
| 8 | Where JWTs are validated; browser session, CORS, CSRF | [0015](0015-jwt-rs256-jwks-validate-everywhere.md), [0016](0016-browser-session.md) |
| 9 | Ordering and replay ("why not Kafka?") | [0017](0017-ordering-and-replay.md), [0006](0006-rabbitmq-as-broker.md) |
| 10 | Fitting the RAM budget | [0018](0018-single-postgres-and-ram-budget.md) |
| 11 | Live updates in the browser | [0019](0019-sse-realtime-service.md) |
| 12 | How much the framework hides | [0007](0007-amqplib-over-nest-microservices.md), [0023](0023-retry-topology-ttl-dlx.md) |
| 13 | Stack choices within TypeScript | [0004](0004-nestjs-everywhere-express-for-fixtures.md), [0005](0005-typeorm-and-class-validator-usage.md), [0003](0003-monorepo-pnpm-workspaces.md), [0022](0022-contracts-package-and-shared-libs.md) |

Tensions added during design:

| Tension | ADR |
|---------|-----|
| Microservices or a modular monolith at all? | [0002](0002-service-boundaries.md) |
| Which gateway? | [0020](0020-nestjs-api-gateway.md) |
| Which analytics store? | [0021](0021-postgres-analytics-store.md) |
| Checking user-supplied URLs safely (SSRF, overload) | [0026](0026-pinger-safety.md) |
| Trace context across an outbox | [0025](0025-observability-stack.md) |
| No external services at all | [0028](0028-local-stand-ins.md) |

## Reading order for interviews

If you only reread six before an interview: [0008](0008-transactional-outbox.md), [0012](0012-idempotent-consumers.md), [0010](0010-postgres-leases-for-scheduling.md), [0014](0014-on-call-in-incident-and-local-replicas.md), [0017](0017-ordering-and-replay.md), [0002](0002-service-boundaries.md).
