# Tech stack

Every tool, its pinned version, why it was chosen, and which Docker image runs it. Versions were checked against the npm registry and Docker Hub on **2026-11-15**. Pin exact versions in `package.json` and image tags in Compose; upgrade deliberately, one at a time.

Everything listed is open source and runs locally. Nothing needs an account or an API key.

## Runtime and language

| Tool | Version | Why | Image |
|------|---------|-----|-------|
| Node.js | 26.13.1 (LTS "Lithium") | Active LTS. Nest 12's generators need ≥ 22.22.3, ≥ 24.15 or ≥ 26 | `node:26.13.1-bookworm-slim` |
| TypeScript | 6.0.3 | Nest 12 tooling requires ≥ 6. TypeScript 7 exists (7.0.2) but is not adopted until decorator-metadata support is confirmed for the Nest + TypeORM + class-validator combination ([open questions](../open-questions.md)) | — |
| pnpm | 12.21.0 | Workspaces, strict dependency resolution, `pnpm deploy` for small images ([ADR-0003](../adr/0003-monorepo-pnpm-workspaces.md)) | via Corepack |

The dev machine currently has Node 24.22.1, which can run and scaffold Nest 12. Match the images by installing Node 26 (or accept the difference, since services run in containers).

## Backend

| Library | Version | Used for | Decision |
|---------|---------|----------|----------|
| NestJS (`@nestjs/core`, `common`, `platform-express`) | 12.5.0 | Every service. Projects generated as **ESM** | [ADR-0004](../adr/0004-nestjs-everywhere-express-for-fixtures.md) |
| Express | 5.3.0 | Nest's HTTP adapter; standalone only in `flaky-target` and `dev-stubs` | same |
| TypeORM | 1.1.3 | ORM and migrations | [ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md) |
| `@nestjs/typeorm` | 12.0.3 | Nest integration | same |
| `pg` | 8.23.1 | PostgreSQL driver | same |
| class-validator / class-transformer | 0.15.1 / 0.5.1 | HTTP DTOs and consumed event payloads | same |
| amqplib | 2.3.1 | RabbitMQ client (types and connection recovery are built in since 1.x) | [ADR-0007](../adr/0007-amqplib-over-nest-microservices.md) |
| jose | 6.2.17 | JWT signing, verification, JWKS | [ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md) |
| argon2 | 0.45.2 | Password hashing (Argon2id) | [security](../cross-cutting/security.md) |
| `@nestjs/config` | 12.1.1 | Typed, validated configuration | [configuration](../cross-cutting/configuration.md) |
| `@nestjs/terminus` | 12.1.0 | Health endpoints | [health checks](../learning/concepts/health-checks.md) |
| `@nestjs/swagger` | 12.1.2 | Serves each service's OpenAPI document in dev | [testing strategy](../cross-cutting/testing-strategy.md) |
| `@nestjs/schedule` | 12.0.4 | Janitor jobs (retention cleanup) | [data ownership](data-ownership.md) |
| `@nestjs/throttler` + `@nest-lab/throttler-storage-redis` | 6.7.1 + 1.2.0 | Gateway rate limiting backed by Redis | [ADR-0020](../adr/0020-nestjs-api-gateway.md) |
| ioredis | 6.0.0 | Redis client | same |
| http-proxy-middleware | 4.2.1 | Gateway reverse proxy | same |
| undici | 8.12.4 | HTTP client for checks and webhooks, with a custom DNS lookup for the SSRF guard | [ADR-0026](../adr/0026-pinger-safety.md) |
| p-limit | 7.3.3 | Concurrency cap for in-flight checks | same |
| opossum | 10.0.0 | Circuit breaker around notification senders | [resilience](../cross-cutting/resilience.md) |
| nodemailer | 10.5.0 | SMTP client | [notification](../services/notification.md) |
| nestjs-pino / pino | 5.3.1 / 10.4.0 | Structured JSON logs | [observability](../cross-cutting/observability.md) |
| `@opentelemetry/api` | 1.10.0 | Tracing and metrics API | [ADR-0025](../adr/0025-observability-stack.md) |
| `@opentelemetry/sdk-node` | 0.301.2 | SDK and OTLP export | same |
| `@opentelemetry/auto-instrumentations-node` | 0.83.1 | HTTP, Express, pg, amqplib, ioredis, pino instrumentation | same |

**Not used, on purpose:** `@nestjs/microservices` (see [ADR-0007](../adr/0007-amqplib-over-nest-microservices.md)), `amqp-connection-manager` (amqplib now recovers connections itself), Passport (a 30-line guard with `jose` is clearer than a strategy), and `@nestjs/observe`, Nest 12's own observability SDK (OpenTelemetry is the vendor-neutral standard this project sets out to learn; see [ADR-0025](../adr/0025-observability-stack.md)).

## Frontend

| Library | Version | Used for | Decision |
|---------|---------|----------|----------|
| React / React DOM | 19.3.1 | UI | fixed by the brief |
| Vite | 8.3.8 | Dev server and build | [ADR-0024](../adr/0024-frontend-stack.md) |
| React Router | 8.5.1 | Client-side routing (library use, no server rendering) | same |
| TanStack Query | 5.107.1 | Server state: fetching, caching, invalidation | same |
| Mantine (`@mantine/core`, `hooks`, `notifications`) | 9.8.1 | Component library | same |
| React Hook Form | 7.92.0 | Forms | same |
| zod | 4.6.7 | Form validation in the browser only | same |
| Recharts | 3.10.1 | Uptime and latency charts | same |
| openapi-typescript / openapi-fetch | 7.13.0 / 0.17.0 | Types and a typed client generated from the OpenAPI specs | same |
| eventsource-parser | 4.1.3 | Parsing the SSE stream read with `fetch` | [ADR-0019](../adr/0019-sse-realtime-service.md) |

## Testing and quality

| Tool | Version | Used for |
|------|---------|----------|
| Vitest | 5.1.0 | Unit and integration tests, backend and frontend. Nest 12 generates Vitest for ESM projects |
| unplugin-swc / `@swc/core` | 2.0.0 / 1.16.22 | Lets Vitest compile Nest's decorators with metadata |
| Supertest | 7.3.2 | HTTP API tests against a Nest app |
| Testcontainers | 12.2.1 | Real PostgreSQL and RabbitMQ in integration tests |
| Pact (`@pact-foundation/pact`) | 17.1.4 | Consumer-driven message contracts |
| Testing Library (React) | 16.3.3 | Component tests |
| MSW | 3.0.6 | Mocking HTTP in component tests |
| Playwright | 1.65.1 | Browser end-to-end tests |
| oxlint | 1.93.0 | Linting (Nest 12's default) |
| Prettier | 3.10.1 | Formatting |
| k6 | 1.8.3 (`grafana/k6:1.8.3`) | Load tests, Phase 5 only |

Decisions: [ADR-0027](../adr/0027-testing-stack.md), [testing strategy](../cross-cutting/testing-strategy.md).

## Infrastructure images

| Component | Image and tag | Licence | Why |
|-----------|---------------|---------|-----|
| PostgreSQL | `postgres:18.7` | PostgreSQL | Native `uuidv7()`, `SKIP LOCKED`, declarative partitioning. One server, five databases ([ADR-0018](../adr/0018-single-postgres-and-ram-budget.md)) |
| RabbitMQ | `rabbitmq:4.3.7-management-alpine` | MPL 2.0 | Fixed by the brief. 4.3.x rather than 4.4.0, which was three days old on the check date. Quorum queues and streams are built in ([ADR-0006](../adr/0006-rabbitmq-as-broker.md)) |
| Redis | `redis:8.8.4-alpine` | AGPLv3 (one of Redis 8's licence options) | Rate-limit counters. Valkey is a drop-in alternative if the licence matters |
| Mailpit | `axllent/mailpit:v1.33.1` | MIT | SMTP catcher with a web inbox and an API that tests can query |
| Grafana OTel LGTM | `grafana/otel-lgtm:0.41.0` | AGPLv3 / Apache 2.0 components | OTel Collector, Tempo, Prometheus, Loki and Grafana in one container; built for development ([ADR-0025](../adr/0025-observability-stack.md)) |
| Toxiproxy | `ghcr.io/shopify/toxiproxy:2.13.0` | MIT | Injects latency and connection failures for the break-it lab |
| nginx | `nginx:1.30.6-alpine` | BSD-2-Clause | Serves the built React app |

RabbitMQ plugins: `rabbitmq_management` and `rabbitmq_prometheus` (both shipped in the image). No third-party plugins: the delayed-message exchange is deliberately not used ([ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)).

## Built in this repository

| Container | Stack | Purpose |
|-----------|-------|---------|
| `flaky-target` | Express 5 | A target whose behaviour you control: `up`, `down`, `slow`, `flapping`, `huge-body`. Demos and tests never touch real websites |
| `dev-stubs` | Express 5 | Local stand-ins: generic webhook inbox, chat-webhook receiver (accepts the Discord-style `{ "content": … }` shape), SMS gateway mock. Each can be told to fail or respond slowly |

Specification of both: [ADR-0028](../adr/0028-local-stand-ins.md).

## Version policy

- **Exact pins** everywhere (`save-exact=true` in `.npmrc`; full image tags, never `latest`).
- One `pnpm.catalog` entry per shared dependency so all workspaces agree on a version.
- Upgrades are their own commits. Major upgrades of Nest, TypeORM, React Router or Vite get a line in the relevant ADR's "Revisit when".
- Several majors here are recent (Nest 12, TypeORM 1.x, React Router 8, Vite 8, Vitest 5, TypeScript 6). Where a doc describes an API, treat the **official docs for the pinned version** as the authority and this repository's docs as the intent.

## What was verified and how

| Claim | Source checked |
|-------|----------------|
| Package versions | `npm view <pkg> version` |
| Image tags exist | Docker Hub registry API |
| Nest 12 is ESM, scaffolds Vitest for ESM projects, requires TypeScript ≥ 6 and the Node versions above | Nest migration guide (v11 → v12) |
| Nest's RabbitMQ transport options and manual-ack behaviour | Nest docs, "RabbitMQ" microservices page |
| Streams can be consumed over AMQP 0.9.1 with `x-stream-offset`, need prefetch and acks | RabbitMQ docs, "Streams" |
| amqplib has built-in types and connection recovery | amqplib changelog |

Not independently verified, flagged where used: exact API details of React Router 8, Mantine 9 and TypeORM 1.x beyond the general model. Check their docs when implementing.
