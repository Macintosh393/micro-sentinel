# Roadmap: phases

Six phases, Phase 0 to Phase 5. Each phase ends with a system that runs, a demo you can give, and a set of concepts you can explain. The React app grows in **every** phase.

Each phase has its own file with: goal and scope, ordered tasks (one per sitting), Definition of Done as runnable checks, a demo script, concepts, interview talking points, and shortcuts taken.

| Phase | Theme | File | Effort (h) | Weeks at 10 h/week |
|-------|-------|------|-----------|---------------------|
| 0 | Foundations | [phase-0.md](phase-0.md) | 15–20 | 2 |
| 1 | Walking skeleton | [phase-1.md](phase-1.md) | 40–50 | 4–5 |
| 2 | Introduce the broker | [phase-2.md](phase-2.md) | 50–70 | 5–7 |
| 3 | Resilience and scale | [phase-3.md](phase-3.md) | 35–45 | 4 |
| 4 | Observability and CQRS | [phase-4.md](phase-4.md) | 35–45 | 4 |
| 5 | Stretch | [phase-5.md](phase-5.md) | open | — |
| | **Core total (0–4)** | | **175–230** | **19–22** (about 5 months) |

Estimates assume TypeScript is comfortable and Nest, RabbitMQ and OpenTelemetry are new. Each includes ~20 % for reading docs and debugging tooling. Treat them as a budget: if a phase runs 50 % over, stop and cut scope rather than push on.

## The phase most likely to blow up: Phase 2

Phase 2 introduces at once: RabbitMQ and AMQP concepts, the in-repo messaging package (connection recovery, topology, confirms, consumer wrapper), the outbox relay, the inbox, a new service (notification), a new edge component (realtime), SSE in the browser, and replicas. Each is small; together they are the steepest part of the project, and the messaging package is code everything after it depends on.

Mitigations, built into the Phase 2 task list:

1. **Spike first** (one sitting): publish and consume one message with raw amqplib, ack and nack by hand, watch it in the management UI. Then the `@nestjs/microservices` spike ([ADR-0007](../adr/0007-amqplib-over-nest-microservices.md)). Both thrown away.
2. **Messaging package before any service uses it**, with its integration tests (relay, dedupe, crash-before-ack).
3. **One flow end to end before widening**: monitor → incident over the broker, with the outbox and inbox, *before* notification or realtime exist.
4. **Cut list** if it overruns, in this order: org channels (keep only email pages) → realtime (keep polling one more phase) → replica bootstrap command (re-register users by hand).

Second riskiest: the gateway's proxying details in Phase 0–1 (body streaming, middleware order). Mitigation: the routing table and one route working on day one.

## Rules that apply to every phase

- **Definition of Done is a set of commands with expected outputs.** If it cannot be checked by running something, it is not done.
- **Shortcuts are written down** in the phase file with the phase that pays them off.
- **`pnpm ci:local` stays green** at the end of every sitting that touches code.
- **Measure memory** with `docker stats --no-stream` at the end of each phase and update the estimate in [deployment-local](../architecture/deployment-local.md#memory-budget).
- **Update docs when reality differs.** A changed decision gets a new ADR ([ADR-0001](../adr/0001-record-architecture-decisions.md)).
- **Record the demo** (a short screen recording) at the end of each phase. It is interview material.

## How phases map to the brief

| Brief | Here |
|-------|------|
| — | Phase 0 added: monorepo, tooling, Compose base, conventions, CI that runs locally, one empty service and the React shell through the gateway |
| 1. Walking skeleton (gateway + monitor + incident over REST) | Phase 1, plus auth (needed for a real gateway) and the first frontend slice |
| 2. Introduce the broker; add notification | Phase 2, plus the outbox, inbox, realtime and SSE |
| 3. Resilience and scale (replicas, coordination, DLQ) | Phase 3, plus delayed retries, escalation, on-call schedules. (Leases already exist from Phase 1; Phase 3 runs several workers and proves them) |
| 4. Observability and CQRS | Phase 4, plus contract tests and Playwright |
| — | Phase 5 stretch |

## Concept → phase

| Phase | Concepts introduced ([learning map](../learning/README.md)) |
|-------|------------|
| 0 | Microservices vs modular monolith · bounded contexts · service discovery · health checks · API gateway |
| 1 | Database-per-service · JWT/JWKS · browser authentication · sync communication · Node event loop · distributed locks and leases · NestJS building blocks |
| 2 | Async communication · event-driven architecture · transactional outbox · at-least-once and idempotency · choreography · ordering · schema evolution · real-time delivery · eventual consistency |
| 3 | Retries, backoff, jitter · DLQ and poison messages · circuit breakers and bulkheads · fencing tokens (proven) · backpressure · leader election (avoided) |
| 4 | Distributed tracing · CQRS · event sourcing (replay) · contract testing · CAP/PACELC |

## Interview readiness by phase

You can credibly put MicroSentinel on a CV after **Phase 1** ("NestJS services behind a gateway, JWT, PostgreSQL, React"). After **Phase 2** it becomes a distributed-systems story. Phases 3 and 4 add depth that is mostly bonus at junior level. If time runs short, a polished Phase 2 with a clear demo beats a half-finished Phase 4.
