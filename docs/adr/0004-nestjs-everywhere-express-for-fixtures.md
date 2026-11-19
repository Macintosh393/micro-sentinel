# ADR-0004: Use NestJS for every service and plain Express only for lab fixtures

- Status: Accepted
- Date: 2026-11-15
- Phase: 0

## Context

The preference is NestJS by default, with Express "only where it is clearly more suitable, justified per service". Each deployable needs a verdict. Nest runs on top of an HTTP adapter (Express by default, Fastify optionally), so "Nest vs Express" really means "Nest's structure on top of Express vs Express alone".

Nest 12 ships as ESM packages; new projects can be generated as CommonJS or ESM, and ESM projects use Vitest by default (Nest migration guide, v11 → v12).

## Decision drivers

- Target roles ask for NestJS; every hour inside Nest is interview preparation.
- A consistent structure across services lowers the cost of switching between them.
- Framework overhead should be justified where the component is tiny.

## Options considered

### Per component

| Component | Nest adds | Express alone would | Verdict |
|-----------|-----------|---------------------|---------|
| auth, monitor (API), incident, notification, analytics | DI, modules, validation pipe, guards, exception filters, TypeORM and config integration, testing module | Need all of that hand-assembled | **Nest** |
| monitor-worker | Same DI graph and config as the monitor API; lifecycle hooks for graceful shutdown; a health endpoint | Be a separate codebase sharing entities by import | **Nest** (same app, different role) |
| gateway | Guards for JWT, throttler module, config, health, the same logging and tracing setup | Be ~150 lines with `http-proxy-middleware` | **Nest**, narrowly. See below |
| realtime | Guard for JWT, lifecycle hooks, the shared messaging module | Be small, but need the same JWT and AMQP code | **Nest** |
| flaky-target | Nothing: three routes and an in-memory mode switch | Be ~60 lines | **Express** |
| dev-stubs | Nothing: a few routes storing requests in memory and an HTML list | Be ~150 lines | **Express** |

### The gateway is the closest call
- For Express: the gateway's core is middleware (proxying), which bypasses most of Nest (controllers, pipes, interceptors). The proxy middleware must be mounted before any body parser so request bodies stream through untouched.
- For Nest: authentication guard, rate limiting, config validation, health checks, logging and tracing are identical to the other services and come from shared packages built as Nest modules. Rewriting them for Express would duplicate code.
- Decided in detail in [ADR-0020](0020-nestjs-api-gateway.md).

### Fastify adapter instead of Express
- Pro: faster, schema-based serialization.
- Con: the performance difference is irrelevant at 17 requests per second; Express is what most Nest codebases and interviews assume; `http-proxy-middleware` targets Express-style middleware.

## Decision

- **NestJS 12 (Express adapter, ESM projects)** for gateway, auth, monitor, incident, notification, analytics and realtime.
- **Plain Express 5** for `flaky-target` and `dev-stubs`. They are test fixtures: no database, no auth, no DI, and they must stay trivially readable so that nobody wonders whether a failed test is the fixture's fault.
- Background work (outbox relay, lease loop, consumers, janitors) runs inside Nest providers using lifecycle hooks (`OnApplicationBootstrap`, `OnApplicationShutdown`), not in separate ad-hoc scripts.

Every service doc has a "Framework and Nest structure" section listing its modules, providers, guards, interceptors, pipes and filters.

## Consequences

- Positive: one mental model; shared Nest modules (`messaging`, `auth-guard`, `observability`) plug into every service.
- Positive: the Express fixtures give a small, honest point of comparison: you will have written the same kind of HTTP handler both ways.
- Negative: Nest's startup and memory cost (~150–200 MB per process in dev) is paid seven times.
- Negative: ESM + decorators + Vitest needs SWC for decorator metadata; this is a known setup step, done once in Phase 0.
- Harder later: nothing significant.

## Revisit when

- A service's hot path is dominated by framework overhead (not expected below thousands of requests per second): try the Fastify adapter.
- The gateway needs features Nest fights against (WebSocket proxying, HTTP/2): replace it with a dedicated proxy ([ADR-0020](0020-nestjs-api-gateway.md)).

## Learning choice?

Partly. Nest everywhere is what a Nest shop would do. Building the gateway in Nest at all is the learning choice, covered in ADR-0020.

## How to explain it in an interview

> "Everything that has business logic is NestJS, because I wanted consistent modules, dependency injection, guards and validation everywhere, and shared modules for messaging and auth. The only plain Express apps are two test fixtures: a fake target and a fake webhook receiver. They're a hundred lines with no dependencies, and Nest would add structure without adding value there."
