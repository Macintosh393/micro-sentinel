# Interview question bank

Grouped by topic. **J** = likely at junior level; **B** = bonus depth. Each links to the document that answers it. Many guides also contain model answers under "Interview questions"; the aim here is coverage and quick self-testing.

How to use: cover the "Answer starts with" column, answer out loud with a MicroSentinel example, then check the linked doc.

## Node.js

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 1 | J | How does Node handle many concurrent requests on one thread? | Event loop + OS async I/O via libuv | [event loop](concepts/node-event-loop.md) |
| 2 | J | What happens if you run a CPU-heavy loop in a request handler? | Blocks every callback, timers fire late | [event loop](concepts/node-event-loop.md) |
| 3 | J | Promises vs callbacks vs async/await; what does `await` actually do? | Yields until the promise settles; continuation is a microtask | [event loop](concepts/node-event-loop.md) |
| 4 | J | `process.nextTick` vs `setImmediate` vs `setTimeout(0)`? | Microtask before the loop continues vs check phase vs timers phase | [event loop](concepts/node-event-loop.md) |
| 5 | B | How do you cap concurrency for outbound requests? | Semaphore (`p-limit`), claim only free slots | [ADR-0026](../adr/0026-pinger-safety.md) |
| 6 | B | What uses libuv's thread pool, and when does it matter? | DNS lookup, fs, crypto, zlib; Argon2 in auth | [event loop](concepts/node-event-loop.md) |
| 7 | J | How do you shut down a Node service gracefully? | SIGTERM → stop accepting → drain → close connections | [resilience](../cross-cutting/resilience.md#graceful-shutdown) |
| 8 | B | What is backpressure in Node streams? | `write()` returns false; wait for `drain` | [backpressure](concepts/backpressure.md) |

## NestJS

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 9 | J | Explain the Nest request lifecycle. | Middleware → guards → interceptors → pipes → handler → interceptors → filters | [Nest building blocks](concepts/nestjs-building-blocks.md) |
| 10 | J | Guard vs middleware vs interceptor? | Access decision with metadata vs raw pre-routing vs wrapping the handler | [Nest building blocks](concepts/nestjs-building-blocks.md) |
| 11 | J | What is dependency injection and why use it? | Declared constructor deps, swappable in tests | [Nest building blocks](concepts/nestjs-building-blocks.md) |
| 12 | J | How does `ValidationPipe` work; what do `whitelist` and `forbidNonWhitelisted` do? | class-validator DTOs; strip vs reject unknown fields | [ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md) |
| 13 | J | How do you return consistent error responses? | Global exception filter → problem+json with `code` | [versioning](../contracts/versioning.md#http-api-versioning) |
| 14 | B | Have you used `@nestjs/microservices`? Why not for RabbitMQ here? | One queue per server, auto-ack default, own wire format | [ADR-0007](../adr/0007-amqplib-over-nest-microservices.md) |
| 15 | B | How do you share code across Nest services without coupling them? | Technical modules only; dependency rules in CI | [ADR-0022](../adr/0022-contracts-package-and-shared-libs.md) |
| 16 | J | How do you test a Nest service? | Testing module, real DB via Testcontainers, Supertest | [testing strategy](../cross-cutting/testing-strategy.md) |

## REST API design

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 17 | J | PUT vs PATCH vs POST? | Replace vs partial update vs create/command | [OpenAPI specs](../contracts/openapi/) |
| 18 | J | When do you return 400, 401, 403, 404, 409, 422? | Validation, unauthenticated, forbidden, not found (incl. other tenant), conflict, semantically invalid | [incident.yaml](../contracts/openapi/incident.yaml) |
| 19 | J | Offset vs keyset (cursor) pagination? | Keyset is stable and fast on deep pages | [incident-schema](../data/incident-schema.md) |
| 20 | J | What makes an endpoint idempotent? Is your acknowledge endpoint? | Same effect when repeated; yes, `200` unchanged | [domain model](../02-domain-model.md#incident-lifecycle) |
| 21 | B | How do you version an API? | Path version, additive changes only, new major side by side | [versioning](../contracts/versioning.md) |

## SQL and PostgreSQL

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 22 | J | What does a transaction guarantee? | Atomicity, consistency, isolation, durability; outbox relies on atomicity | [outbox](concepts/transactional-outbox.md) |
| 23 | J | What is an index, and how did you choose yours? | Lookup structure; from query patterns, partial indexes | [monitor-schema](../data/monitor-schema.md) |
| 24 | J | What is a partial index? Give an example. | Index with `WHERE`; one unresolved incident per monitor | [incident-schema](../data/incident-schema.md) |
| 25 | B | Explain `SELECT … FOR UPDATE SKIP LOCKED`. | Lock rows, skip locked ones: a contention-free work queue | [locks guide](concepts/distributed-locks-leases-fencing.md) |
| 26 | J | Optimistic vs pessimistic locking? | Version check on write vs row lock while deciding | [incident-schema](../data/incident-schema.md) |
| 27 | B | Why partition a table? | Instant retention by dropping partitions, pruning | [analytics-schema](../data/analytics-schema.md) |
| 28 | J | What is an upsert and when is it dangerous? | `INSERT … ON CONFLICT`; blind upserts break ordering → version guard | [incident-schema](../data/incident-schema.md#replicas) |
| 29 | J | Why `synchronize: false` and migrations only? | Versioned, reviewed schema; no silent drops | [ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md) |
| 30 | J | ORM vs query builder vs raw SQL: when each? | CRUD vs dynamic filters vs locking/CTEs/bulk | [ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md) |
| 31 | B | How do you rename a column with zero downtime? | Expand → migrate → contract | [schema evolution](concepts/schema-evolution.md) |

## React

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 32 | J | Server state vs client state; where do you keep each? | TanStack Query vs context/local state | [web app](../frontend/web-app.md#state) |
| 33 | J | How do you handle loading, error and empty states? | Four states per screen, per query | [web app](../frontend/web-app.md#the-four-states-every-screen-implements) |
| 34 | J | What causes a re-render, and how do you avoid unnecessary ones? | State/props/context changes; RHF uncontrolled inputs, query selectors | [ADR-0024](../adr/0024-frontend-stack.md) |
| 35 | J | What is an optimistic update and how do you roll it back? | Update cache first, restore on error, reconcile by refetch | [web app](../frontend/web-app.md#mutations) |
| 36 | J | How do you validate forms, and does the server still validate? | zod in the browser for UX; server is the authority | [web app](../frontend/web-app.md#forms) |
| 37 | B | How do live updates reach the UI? | SSE hint → invalidate query → refetch | [frontend realtime](../frontend/realtime.md) |
| 38 | J | How do you test React components? | Testing Library by role, MSW for HTTP | [testing strategy](../cross-cutting/testing-strategy.md#component-tests-react) |

## Auth and security

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 39 | J | Where do you store a JWT in the browser? | Memory; refresh in httpOnly SameSite cookie | [browser auth](concepts/browser-authentication.md) |
| 40 | J | What is CSRF / XSS / CORS? | Auto-attached cookies / injected script / cross-origin read permission | [browser auth](concepts/browser-authentication.md) |
| 41 | J | How does a service verify a JWT without calling auth? | Signature with cached JWKS public key; check iss/aud/exp | [JWT guide](concepts/jwt-and-jwks.md) |
| 42 | B | RS256 vs HS256? How do you rotate keys? | Asymmetric; publish new kid, then sign, then retire | [ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md) |
| 43 | J | How do you store passwords? | Argon2id, salted, tuned cost | [security](../cross-cutting/security.md#passwords) |
| 44 | B | What is SSRF and how does a URL pinger defend against it? | Resolve, validate IP, connect to that IP, no redirects | [ADR-0026](../adr/0026-pinger-safety.md) |
| 45 | J | How do you keep tenants' data apart? | Org from the token in every query; 404 cross-tenant | [security](../cross-cutting/security.md#authorization) |

## Docker

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 46 | J | Image vs container? | Template vs running instance | [deployment](../architecture/deployment-local.md) |
| 47 | J | What is a multi-stage build and why use it? | Build tools stay out of the runtime image | [deployment](../architecture/deployment-local.md#images) |
| 48 | J | How do containers find each other in Compose? | Service names via embedded DNS | [service discovery](concepts/service-discovery.md) |
| 49 | J | Liveness vs readiness? | Restart me vs don't send me traffic | [health checks](concepts/health-checks.md) |
| 50 | B | Why does hot reload break in Docker on Windows, and what did you do? | inotify across filesystems; WSL2 checkout or Compose Watch | [deployment](../architecture/deployment-local.md#dev-images-and-hot-reload-on-windows) |
| 51 | J | Why run `node` directly instead of `npm start` in the image? | Signals reach the process; graceful shutdown | [deployment](../architecture/deployment-local.md#images) |

## Microservices and messaging

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 52 | J | Would you use microservices for a new product? | Usually a modular monolith first | [monolith guide](concepts/microservices-vs-modular-monolith.md) |
| 53 | J | When use a queue instead of an HTTP call? | Sender doesn't need the result; receiver may be down | [sync vs async](concepts/sync-vs-async-communication.md) |
| 54 | B | How do you update a DB and publish an event atomically? | Transactional outbox | [outbox](concepts/transactional-outbox.md) |
| 55 | J | What if a message is delivered twice? | Inbox table in the same transaction | [idempotency](concepts/idempotency-and-at-least-once.md) |
| 56 | B | Why is exactly-once delivery impossible? | Effect and ack are two operations | [idempotency](concepts/idempotency-and-at-least-once.md) |
| 57 | B | How do you handle out-of-order events? | Per-aggregate version; ignore stale | [ordering](concepts/partitioning-and-ordering.md) |
| 58 | J | What is a dead-letter queue? | Parking for unprocessable messages | [DLQ](concepts/dead-letter-queues.md) |
| 59 | J | How do you retry well? | Transient only, backoff, jitter, cap, one layer | [retries](concepts/retries-backoff-jitter.md) |
| 60 | B | Why RabbitMQ and not Kafka? | Work-queue semantics; Kafka wins on replay and per-key ordering at volume | [ADR-0006](../adr/0006-rabbitmq-as-broker.md) |
| 61 | B | Exchange, queue, binding, routing key? | Publish to exchange; bindings route to queues by key | [event-driven](concepts/event-driven-architecture.md) |
| 62 | B | How do you stop two workers doing the same job? | `SKIP LOCKED` leases + fencing token | [locks guide](concepts/distributed-locks-leases-fencing.md) |
| 63 | B | What is a fencing token? | Monotonic number checked by the resource | [locks guide](concepts/distributed-locks-leases-fencing.md) |
| 64 | B | Choreography vs orchestration? | Reactions to events vs a coordinator | [choreography](concepts/choreography-vs-orchestration.md) |
| 65 | B | What is CQRS and when is it overkill? | Separate read model; overkill when one DB answers fine | [CQRS](concepts/cqrs.md) |
| 66 | B | Event sourcing: did you use it? | Considered, rejected; replay in analytics | [event sourcing](concepts/event-sourcing.md) |
| 67 | B | Explain CAP / where is your system eventually consistent? | CP vs AP under partition; replicas, read model | [CAP](concepts/cap-pacelc-eventual-consistency.md) |
| 68 | B | How do you share data without sharing a database? | Events → local replicas; versioned snapshots | [data-ownership](../architecture/data-ownership.md) |
| 69 | B | How do you change an event schema safely? | Additive + tolerant reader; v2 side by side | [schema evolution](concepts/schema-evolution.md) |
| 70 | B | What is a circuit breaker; where didn't you use one? | Fail fast when a dependency fails; not on targets | [circuit breakers](concepts/circuit-breakers-and-bulkheads.md) |

## Observability and testing

| # | Level | Question | Answer starts with | Doc |
|---|-------|----------|--------------------|-----|
| 71 | J | How do you debug a request across several services? | Trace ID propagated; one trace | [tracing](concepts/distributed-tracing.md) |
| 72 | B | How does trace context cross a broker and an outbox? | Headers; context stored in the outbox row | [observability](../cross-cutting/observability.md#over-rabbitmq) |
| 73 | B | RED vs USE? Metric cardinality? | Services vs resources; no IDs as labels | [observability](../cross-cutting/observability.md#metrics) |
| 74 | J | Unit vs integration vs e2e: what goes where? | Pure logic vs seams with real infra vs few journeys | [testing strategy](../cross-cutting/testing-strategy.md) |
| 75 | J | Why not mock the database? | Mocks can't reproduce locking, constraints | [ADR-0027](../adr/0027-testing-stack.md) |
| 76 | B | What is consumer-driven contract testing? | Consumers record needs; providers verify | [contract testing](concepts/contract-testing.md) |
| 77 | B | How did you test failure behaviour? | Break-it lab: hypothesis, induce, observe | [break-it lab](../roadmap/break-it-lab.md) |

## System design

| # | Level | Question | Doc |
|---|-------|----------|-----|
| 78 | B | Design PagerDuty / an uptime monitor. | [interview prep](interview-prep.md#system-design-version-design-pagerduty-or-design-an-uptime-monitor) |
| 79 | B | What changes at 100× scale? | [interview prep](interview-prep.md#what-would-you-change-at-100-scale) |
| 80 | B | What would you do differently as a monolith? | [interview prep](interview-prep.md#what-would-you-do-differently-as-a-monolith) |
