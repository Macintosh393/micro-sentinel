# ADR-0027: Test with Vitest, Testcontainers, in-repo Pact message contracts and Playwright

- Status: Accepted
- Date: 2026-11-15
- Phase: 0 (runner, unit, local CI), 1 (API and integration), 2 (messaging integration), 4 (contracts, browser e2e)

## Context

A distributed system fails at its seams: a query that is wrong only against real PostgreSQL, a consumer that mishandles redelivery, a producer that renames a field. The test strategy has to cover those seams without making every test slow. Tool choices: the test runner for backend and frontend, how to test against real infrastructure, how to check event contracts between services, and how to test the UI.

## Decision drivers

- Fast feedback for logic; realistic checks for SQL and messaging.
- Works on Windows + Docker Desktop, offline, with no accounts.
- One runner if possible.
- Interview relevance: unit vs integration vs e2e, mocking, contract testing.

## Options considered

### Runner

| Option | Pros | Cons |
|--------|------|------|
| Jest (backend) + Vitest (frontend) | Jest is what most existing Nest codebases and interview questions assume | Two runners and configs. Per the Nest 12 migration guide, Jest can load Nest's ESM-only packages only on Node ≥ 24.9 |
| **Vitest everywhere** (chosen) | Nest 12 scaffolds Vitest for ESM projects; native ESM; same runner as the Vite frontend; Jest-compatible API (`describe`, `it`, `expect`, `vi.fn()`) | Decorator metadata needs SWC (`unplugin-swc`); some older Nest testing articles assume Jest |

### Integration against infrastructure

| Option | Pros | Cons |
|--------|------|------|
| Mocks of TypeORM repositories and of the AMQP channel | Fast | Proves the code calls the mock; says nothing about `SKIP LOCKED`, constraints, dead-lettering or redelivery, which are the behaviours that matter |
| In-memory substitutes (SQLite, a fake broker) | Fast | Different SQL dialect and semantics; false confidence |
| A long-running Compose stack that tests connect to | Realistic | Shared state between tests; order-dependent failures |
| **Testcontainers** (chosen) | Each suite starts real `postgres:18.7` / `rabbitmq:4.3.7` containers, runs migrations, and discards them; realistic and isolated | Slower (seconds to start); needs Docker running |

### Contracts between services

| Option | Pros | Cons |
|--------|------|------|
| Shared TypeScript types only | Free in a monorepo | Checks shape at compile time for code built together; says nothing about what a consumer *actually relies on*, or about old-consumer/new-producer combinations |
| Schema validation: every example in the event catalog validates against AsyncAPI/JSON Schema and the contracts classes | Cheap; keeps docs, schema and code in step | Producer-centred; does not record consumer expectations |
| **Consumer-driven contracts with Pact (message pacts)** (chosen, narrowly) | The consumer's test states which fields it needs and generates a pact file; the producer's test proves it can produce a message satisfying it. Catches "producer removed a field one consumer needs" | Another tool and concept; with a shared package in one repo the practical gain is small |
| Pact Broker for sharing pacts | The standard for many repos and teams | A server and a database; unnecessary when both sides are in one repository |

### UI

| Level | Tool | Why |
|-------|------|-----|
| Component / page | **Testing Library + MSW** | Tests behaviour the way a user sees it; MSW intercepts `fetch` so components run against realistic API responses |
| Browser end-to-end | **Playwright** | Drives a real browser against the full Compose stack; auto-waiting; traces on failure |

## Decision

| Layer | Tool | Scope | Runs in |
|-------|------|-------|---------|
| Unit | Vitest | Pure logic: monitor confirmation state machine, incident transition table, on-call rotation arithmetic, SSRF address classifier, retry tier selection, React hooks and utilities | milliseconds; every save |
| API (per service) | Vitest + Supertest + Testcontainers (PostgreSQL) | A Nest app booted with its real database: validation, auth guard, status codes, problem+json, tenancy isolation | seconds |
| Integration (messaging) | Vitest + Testcontainers (PostgreSQL + RabbitMQ) | Outbox relay publishes and marks; duplicate delivery is ignored; retry → wait queue → redelivery; poison message → DLQ; lease claim and fencing with two concurrent workers | seconds to tens of seconds |
| Contract (events) | Schema checks for all events; **Pact message pacts for two contracts**: `monitor.state.changed.v1` (consumer incident) and `incident.page.requested.v1` (consumer notification) | Pact files written to `pacts/` in the repo; producer verification reads them from disk. No Pact Broker | seconds |
| Contract (HTTP) | Generated OpenAPI types in the web app; a test per service asserting that the served OpenAPI document matches the committed spec | Drift between spec and implementation | seconds |
| Component (web) | Vitest + Testing Library + MSW | Screens in loading, error, empty and data states; forms; optimistic updates | seconds |
| End-to-end | Playwright against `docker compose up` | Five journeys: register + login; create monitor; outage → incident appears live → acknowledge → recovery resolves; failed delivery → redrive; uptime chart shows data | minutes; before a phase is declared done |
| Chaos | Manual, scripted in the [break-it lab](../roadmap/break-it-lab.md) | Failure behaviour | on demand |

Rules:

- **Mock at process boundaries you do not own** (SMTP, external webhooks → use Mailpit and dev-stubs, which are real servers). **Do not mock what you own** (your database, your broker): use the real thing.
- Time-dependent logic takes a clock abstraction or uses fake timers; tests never `sleep`.
- **Local CI is one command**: `pnpm ci:local` = format check, lint, typecheck, unit, integration, contract, build. End-to-end is `pnpm e2e` because it needs the stack. An optional GitHub Actions workflow runs the same script; nothing depends on it.
- Rough shape: many unit tests, a solid layer of integration tests around the seams, few e2e tests. Detail and examples: [testing strategy](../cross-cutting/testing-strategy.md).

## Consequences

- Positive: the behaviours that distinguish this system (SKIP LOCKED, dedupe, DLQ, fencing) are tested against real PostgreSQL and RabbitMQ.
- Positive: one runner and one assertion style across backend and frontend.
- Positive: Pact is experienced on two real contracts without building it into everything.
- Negative: integration tests need Docker and take time; the full local CI will take a few minutes by Phase 3.
- Negative: no Jest practice, although many existing Nest projects use it (the APIs are nearly identical; say so if asked).
- Negative: Pact adds a concept whose main payoff (independent teams and deploys) does not exist here.
- Harder later: with separate repositories, pacts need a broker or shared storage, and producer verification must run in the producer's pipeline.

## Revisit when

- Local CI exceeds ~5 minutes: reuse containers across suites, or add task caching ([ADR-0003](0003-monorepo-pnpm-workspaces.md)).
- A target employer's stack is Jest-only: port one service's tests as an exercise.
- Services move to separate repositories: add a Pact Broker (it is open source and self-hostable).

## Learning choice?

Partly. Vitest, Testcontainers, Testing Library and Playwright are pragmatic. **Pact in a single-repo, single-developer project is a learning choice**: a pragmatic team in this position would rely on the shared contracts package plus schema checks, and adopt consumer-driven contracts when a second team or repository appeared.

## How to explain it in an interview

> "Unit tests cover the pure logic: the state machines, the rotation maths, the SSRF address checks. The interesting behaviour is at the seams, so integration tests use Testcontainers to run real Postgres and RabbitMQ: I test that the outbox relay publishes, that a duplicate message is ignored, that a failing message goes through the retry queues to the dead-letter queue, and that two workers can't both apply a result. I don't mock my own database or broker, because mocks would only prove I called the mock. For contracts I validate event examples against the schema and I use Pact on two events to try consumer-driven contracts. The UI has component tests with Testing Library and MSW, plus a handful of Playwright journeys against the whole stack."
