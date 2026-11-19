# ADR-0002: Split the system into services along bounded contexts

- Status: Accepted
- Date: 2026-11-15
- Phase: 0 (boundaries), 1–4 (services appear)

## Context

The brief lists six services plus a web app. Before accepting that, two questions need honest answers: should this be microservices at all, and if so, are those the right boundaries?

A **modular monolith** is one deployable application whose code is divided into modules with enforced boundaries (in Nest: one app, several modules, each with its own tables, talking through interfaces). A **microservices** architecture deploys those modules as separate processes that communicate over the network.

Facts about this project: one developer, ~500 monitors at most, one machine, no independent teams, no component with a different scaling or availability need that could not be met by running more copies of one app.

## Decision drivers

- The stated purpose is to learn distributed-systems patterns by having to solve their problems.
- Honesty about over-engineering is itself something interviewers look for.
- The boundaries should be ones a real team could defend, so the lessons transfer.
- Total effort must fit ~10 h/week.

## Options considered

### A. Modular monolith (one Nest app, one database, modules per context)
- Pro: the right engineering answer at this size. One deploy, one transaction across modules, no broker needed, trivially consistent, easy to debug, fits in 500 MB.
- Pro: boundaries can still be clean, and services can be extracted later if a real reason appears.
- Con: the dual-write problem, idempotent consumers, replay, tracing across a broker and distributed coordination either do not arise or can be bypassed with a function call. The project's learning goal is not met.

### B. Microservices exactly as in the brief (6 services; on-call lookup in auth; stateless notification)
- Pro: matches the starting point.
- Con: three boundaries do not survive scrutiny: on-call data in auth ([ADR-0014](0014-on-call-in-incident-and-local-replicas.md)), "stateless" notification ([ADR-0013](0013-notification-owns-delivery-state.md)), and no home for browser push ([ADR-0019](0019-sse-realtime-service.md)).

### C. Microservices along revised bounded contexts (chosen)
Five domain services (auth, monitor, incident, notification, analytics) plus two edge components that own no domain model (gateway, realtime).
- Pro: each service maps to one bounded context in the [domain model](../02-domain-model.md); no synchronous calls between them.
- Con: seven Node processes; more code and more memory than the brief.

### D. Finer split (separate scheduler, checker, on-call, escalation services)
- Pro: more "micro".
- Con: splits things that change together and share invariants (an incident and its escalation timer; a monitor and its lease). Creates chatty calls and distributed transactions for no benefit. This is the **nanoservices** anti-pattern.

## Decision

Option C. Boundaries follow these tests, in order:

1. **Does it own a distinct model with its own invariants?** If yes, it is a candidate service.
2. **Would splitting it force a distributed transaction or a synchronous call on the hot path?** If yes, keep it together. This is why escalation stays inside incident and the scheduler stays inside monitor.
3. **Does it have a different runtime profile?** If only this differs, use a separate *process type* of the same service, not a new service. This is why `monitor` and `monitor-worker` are one service.

A modular-monolith variant of the same design is described in the [monolith guide](../learning/concepts/microservices-vs-modular-monolith.md) so the comparison stays concrete.

## Consequences

- Positive: every pattern in the learning goals has a real reason to exist somewhere in the system.
- Positive: boundaries are defensible with DDD vocabulary.
- Negative: roughly 2–3× the code and effort of the monolith for the same features. Operational surface (seven services, broker, five databases) that a one-person product would never choose.
- Negative: cross-service features need events, replicas and eventual consistency where a join would have done.
- Harder later: merging services back is easy; changing a boundary means moving data and rewriting event contracts.

## Revisit when

- A boundary produces a synchronous call on a hot path or a need for a transaction across two services: that boundary is wrong.
- Two services always change and deploy together for three features in a row: merge them.

## Learning choice?

**Yes.** A pragmatic team would build option A: one Nest application with modules `auth`, `monitoring`, `incidents`, `notifications`, `analytics`, one PostgreSQL database with a schema per module, in-process events (or a simple job queue for notification retries), and one background worker process for checks. They would extract a service only when a concrete force appeared: a team boundary, a component needing to scale or fail independently, or a different technology.

## How to explain it in an interview

> "Honestly, at this scale a modular monolith is the right call, and I'd say so to a team. I split it into services on purpose because I wanted to hit the problems you only get with separate processes: dual writes, duplicate and out-of-order messages, coordination between replicas. I drew the boundaries along bounded contexts and used one test in particular: if splitting something would need a distributed transaction or a synchronous call on the hot path, it stays together. That's why escalation lives with incidents and the scheduler lives with monitors."
