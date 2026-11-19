# Prompt: MicroSentinel planning & learning docs

> Paste everything below the line into a Fable 5.2 session opened in this repo.
> Edit the **My preferences** block first. Anything you leave as `decide`, Fable chooses and justifies in an ADR.

---

You are a staff-level distributed-systems architect, an experienced NestJS and React engineer, and a patient teacher. Your job in this session is to write the complete set of **planning, architecture, and learning documents** for a project called **MicroSentinel**, before any implementation code exists. The repository contains only this prompt, in `prompts/`.

## Why this project exists

I'm building MicroSentinel to actually understand microservices before interviewing for **junior full-stack Node.js roles (NestJS + React, TypeScript)**. At junior level, microservices knowledge sets a candidate apart but isn't expected, so the project also has to make me stronger on the fundamentals those interviews do test: Node.js, NestJS, REST API design, SQL, React, testing, and Docker. The finished code matters less than understanding it well enough to explain and defend every decision. The docs therefore have two jobs, and both matter equally:

1. **Engineering plan.** Specific enough that I (or a later coding session) can implement each phase without reopening design questions.
2. **Curriculum.** Each architectural decision should teach the concept behind it: what problem it solves, what it costs, when you shouldn't use it, and how to talk about it in an interview.

If a choice is good engineering but teaches nothing, say so. If a choice exists mainly for learning (and a real team might skip it), label it **"learning choice"** and say what a pragmatic team would do instead. Being honest about where microservices are overkill is part of the lesson, and interviewers look for it.

Write for my level. I'm comfortable in TypeScript and focused on the Nest + React stack, but I haven't built a distributed system before. Explain from first principles, define terms the first time they appear, and go deep on the core patterns rather than giving a shallow tour of everything.

## My preferences

```yaml
language: TypeScript everywhere, backend and frontend
backend_framework: NestJS by default; Express only where it is clearly more suitable, justified per service in an ADR
frontend: React # build tool, routing, data fetching, UI library: decide
message_broker: RabbitMQ # fixed; ADR still compares Kafka (or Redpanda), NATS JetStream, Redis Streams
analytics_store: decide # weigh ClickHouse, TimescaleDB, plain Postgres
api_gateway: decide # weigh Traefik, Kong (OSS), Envoy, a NestJS/Express gateway
orm_and_validation: TypeORM + class-validator DTOs # fixed; ADR still compares Prisma, Drizzle, MikroORM, and zod
monorepo_tooling: decide # e.g. Nx, pnpm workspaces, Nest CLI monorepo mode
hosting: self-hosted only # every component runs as an open-source Docker image; no SaaS, cloud APIs, accounts, or API keys
target_roles: junior full-stack Node.js developer (NestJS + React)
dev_machine: Windows 11 + Docker Desktop (WSL2 backend)
compose_ram_budget: ~8 GB for the whole stack at once
time_budget: ~10 hours/week
```

Values marked **fixed** are my decisions, so don't reopen them. They still get a full ADR with status Accepted: the real alternatives, an honest comparison, and where an alternative would have been the better call. An interviewer will ask "why not Kafka?" or "why not Prisma?", and I need a good answer. Design around each fixed tool's actual strengths and weaknesses rather than pretending it has none.

## Project brief (starting point, not a spec)

Users register URLs to monitor. Workers ping those URLs on a schedule. When a site goes down, the system opens an incident, works out who is on call, sends alerts (email, webhooks, Discord-style chat webhooks, SMS), and tracks resolution metrics. A React web app lets users manage monitors, watch incidents appear live, acknowledge and resolve them, and view uptime. Think of it as a small PagerDuty + UptimeRobot.

| Service         | Responsibility                                                                               | Storage                           | Main pattern to learn                                                                         |
| --------------- | -------------------------------------------------------------------------------------------- | --------------------------------- | --------------------------------------------------------------------------------------------- |
| API Gateway     | Single entry point, rate limiting, JWT validation, routing                                   | Redis (rate limits)               | Edge routing, auth offloading                                                                 |
| Auth & Org      | Users, organizations, API keys, issuing JWTs                                                 | PostgreSQL                        | Identity provider, stateless token verification                                               |
| Monitor         | CRUD for monitored endpoints + the ping scheduler/workers                                    | PostgreSQL + Redis                | Distributed coordination (no double pings across replicas)                                    |
| Incident        | Consumes "site down" signals, manages incident lifecycle (OPEN → ACK → RESOLVED)             | PostgreSQL                        | State machines, idempotency, possibly event sourcing                                          |
| Notification    | Consumes incident updates, sends email, webhooks, chat, and SMS (local stand-ins by default) | Stateless + DLQ                   | Pub/sub, retries with backoff, dead-letter queues                                             |
| Analytics / SLA | Consumes ping + incident events, computes uptime %, MTTA/MTTR                                | ClickHouse / Timescale / Postgres | CQRS read models                                                                              |
| Web app (React) | Dashboard: monitor CRUD, live incident list, acknowledge/resolve, uptime and SLA views       | Browser only                      | A frontend in a microservices system: one entry point, auth in the browser, real-time updates |

Cross-cutting themes: sync vs async communication, database-per-service, resilience (kill a service and its messages wait in the broker), distributed tracing (OpenTelemetry, with trace context carried over HTTP _and_ broker messages), and everything self-hosted, running locally from one `docker compose up` with health checks and service discovery by DNS name.

Proposed phases:

1. **Walking skeleton.** Gateway + Monitor + Incident over REST, in Docker Compose.
2. **Introduce the broker.** Decouple Monitor → Incident with events. Add Notification.
3. **Resilience & scale.** Multiple Monitor replicas with coordination, DLQ for failed notifications.
4. **Observability & CQRS.** OpenTelemetry across all services; Analytics service with uptime dashboards.

## Design tensions you must resolve explicitly

The brief has gaps and a few choices that sound good but don't survive scrutiny. Don't just accept it. Each item below gets its own ADR (or a clearly reasoned section) that compares the options and reaches a decision:

1. **The dual-write problem.** Monitor updates its DB _and_ publishes an event. What happens if one succeeds and the other fails? Evaluate the transactional outbox (polling publisher vs. CDC such as Debezium) against the alternatives. The brief never mentions this, and it's one of the most common interview questions.
2. **What Monitor actually publishes.** Raw ping results for every check (high volume, which Analytics needs) or state transitions such as UP→DOWN after N consecutive failures (low volume, which Incident needs)? Possibly both, on separate streams. Where does flap detection and failure confirmation live? Should a failure be confirmed by a second worker before an incident opens?
3. **Avoiding double pings across replicas.** Redis locks (and the Redlock correctness debate: lock expiry, GC pauses, fencing tokens) vs. lease-based scheduling in Postgres (`SELECT … FOR UPDATE SKIP LOCKED`) vs. static partitioning or consistent hashing of monitors across workers vs. leader election. Pick the simplest option that's correct, and explain why a double ping is mostly harmless here while a double incident or double page is not.
4. **Event sourcing in Incident.** Real event sourcing, or a state machine plus an append-only incident timeline? Be honest about the cost. If you recommend against full event sourcing, the concept guide should still teach it properly.
5. **Idempotency and "exactly once".** Brokers deliver at least once. Define idempotency keys, dedupe storage, and how each consumer stays correct under redelivery and reordering. Explain why "exactly-once delivery" is a myth and what "effectively once processing" means.
6. **"Stateless" Notification.** Avoiding duplicate pages for one incident needs _some_ state (dedupe keys, delivery attempts, escalation timers). Decide where that state lives.
7. **The on-call lookup.** The brief has Incident make a synchronous gRPC call to Auth/Org ("who is on call for org 42?"). That creates temporal coupling: if Auth is down, incidents can't route. Compare it against Incident keeping a local replica of on-call data built from events. Also decide which bounded context owns schedules and escalation policies. It may not be Auth.
8. **Where JWTs are validated.** At the gateway only, or also in each service (defense in depth / zero trust)? RS256 with a JWKS endpoint vs. a shared secret. How service-to-service calls authenticate. On the browser side: where the React app keeps its session (httpOnly cookie vs. in-memory access token plus refresh token), and what that means for CORS and CSRF.
9. **Ordering and replay.** Which flows need per-key ordering (per monitor, per incident)? Can Analytics rebuild its read model from scratch by replaying history? With RabbitMQ that isn't free: classic queues delete messages once they're acked. Compare RabbitMQ Streams, keeping an event archive (e.g. retained outbox rows or an archive consumer), and rebuilding from the source services. Explain how ordering works with RabbitMQ (per-queue ordering, what competing consumers and requeues do to it, single-active-consumer or consistent-hash exchanges for per-key ordering). Make this the main "why not Kafka?" material in the broker ADR.
10. **Fitting the RAM budget.** Kafka + ClickHouse + Jaeger/Tempo + Postgres×N + Redis is heavy. Use Compose profiles per phase, lighter alternatives (Redpanda, one Postgres server with a database per service), and say which shortcuts break "database per service" in spirit and which don't.
11. **Live updates in the browser.** The React app should show incidents opening and being acknowledged without a refresh. Compare polling, Server-Sent Events, and WebSockets (Nest gateways, socket.io). Decide which component pushes to the browser and how it learns about changes, without accidentally turning a domain service into a frontend server. A small BFF or realtime service fed by events is one option.
12. **How much the framework hides.** `@nestjs/microservices` gives one abstraction over many transports, but it can hide exactly what I'm trying to learn: acks, redelivery, dead-lettering, partitions, consumer groups. It also has its own message format, which matters if a non-Nest component ever produces or consumes. Compare its RabbitMQ transport with using `amqplib` directly inside Nest providers, or a thin Nest wrapper such as `@golevelup/nestjs-rabbitmq`. Cover exchanges and routing keys, manual acks, prefetch, dead-letter exchanges, and retry with delay (TTL + DLX queues vs. the delayed-message plugin). List any feature the design needs that the abstraction doesn't expose. Verify against current Nest docs rather than memory.
13. **Stack choices within TypeScript.** Nest vs. Express per service: Nest is the default, and any Express service needs a stated reason, such as a tiny worker or gateway where Nest adds nothing. TypeORM and class-validator are fixed, so the decisions here are how to use them well. Use migrations only, never `synchronize` outside throwaway tests. Choose between repositories and query builders, and say where raw SQL is the better tool (e.g. the outbox poller or `SKIP LOCKED` leases). Decide how transactions work across a domain write and its outbox row, and whether the same class-validator DTOs also validate consumed event payloads. Also decide monorepo tooling, and whether event types live in a shared `contracts` package, plus how to stop that package from coupling services into a distributed monolith.

If you find other tensions, add them.

## Deliverables

Create the following structure. Adjust file names or split and merge files if it reads better, but keep the overall coverage and explain any deviation in the README.

```
README.md                         # what MicroSentinel is, doc map, suggested reading order
docs/
  00-vision-and-scope.md          # problem, goals, non-goals, learning objectives → interview topics
  01-requirements.md              # user stories, functional reqs, NFRs with concrete numbers, assumptions
  02-domain-model.md              # bounded contexts, context map (upstream/downstream, ACLs), aggregates, invariants
  03-glossary.md                  # ubiquitous language; the single source of truth for names
  architecture/
    overview.md                   # the big picture in one page + the 2-minute verbal explanation
    c4-context.md                 # C4 level 1 (Mermaid)
    c4-containers.md              # C4 level 2: every service, DB, broker, gateway, observability stack
    communication.md              # which interactions are sync vs async and why, per edge
    data-ownership.md             # who owns what data, how others get a copy, consistency guarantees
    deployment-local.md           # Compose layout, networks, profiles per phase, health checks, ports, RAM estimate,
                                  # dev images with hot reload (incl. Windows/WSL2 file-watching gotchas), multi-stage prod images
    tech-stack.md                 # every tool and library: pinned version, why chosen, which Docker image runs it
  frontend/
    web-app.md                    # pages and routes, data fetching and caching, state, loading/error/empty states, phase per screen
    auth-in-the-browser.md        # login flow, session/token storage, refresh, CORS, CSRF
    realtime.md                   # how live incident updates reach the browser, reconnection, missed-event catch-up
  services/
    gateway.md
    auth-org.md
    monitor.md
    incident.md
    notification.md
    analytics.md
  contracts/
    event-catalog.md              # every event: name, producer, consumers, key, ordering, payload example, version
    asyncapi.yaml                 # AsyncAPI 3.x description of the event channels
    openapi/                      # one OpenAPI 3.1 spec per service with a public or internal HTTP API
    proto/                        # .proto files, if any gRPC survives the ADRs
    versioning.md                 # schema evolution rules, compatibility, envelope format, trace-context headers
  data/
    <service>-schema.md           # tables (DDL), indexes, retention, migration approach; one per stateful service
  flows/                          # Mermaid sequence diagrams: happy path + failure variants + expected trace tree
    register-monitor.md
    outage-to-resolution.md       # ping fails → confirmed down → incident → page → ack → recover → resolve
    flapping-and-duplicates.md
    notification-retry-and-dlq.md
    service-down-recovery.md      # e.g. Notification offline during an outage, then catches up
    analytics-rebuild.md          # rebuilding a read model from the event history
  adr/
    README.md                     # index table: number, title, status, one-line decision
    0001-record-architecture-decisions.md
    00NN-*.md                     # one per significant decision, including every design tension above
  cross-cutting/
    observability.md              # logs/metrics/traces, RED/USE, trace propagation over HTTP and the broker, dashboards
    resilience.md                 # timeouts, retries + backoff + jitter, circuit breakers, bulkheads, backpressure, budgets
    security.md                   # authn/z, JWT/JWKS, API keys, service-to-service trust, secrets in local dev, SSRF risk of pinging user URLs
    testing-strategy.md           # unit (Jest or Vitest), API (Supertest), integration (Testcontainers),
                                  # consumer-driven contracts (e.g. Pact JS), React components (Testing Library), browser e2e (Playwright), chaos
    configuration.md              # 12-factor config, env vars, per-service settings
    repo-layout.md                # monorepo vs polyrepo decision, folder structure, shared libs policy (and their coupling risk)
  roadmap/
    phases.md                     # Phase 0 (foundations) through Phase 5 (stretch), see below
    break-it-lab.md               # chaos experiments, see below
  learning/
    README.md                     # concept map: which concept, which doc, which phase
    concepts/                     # one guide per concept, see list and template below
    interview-prep.md             # question bank, system-design walkthrough, trade-off talking points
  open-questions.md               # things that really need my input, plus risks and their mitigations
```

### Concept guides (`docs/learning/concepts/`)

At minimum: microservices vs. modular monolith (including when the monolith wins); bounded contexts and DDD basics; database-per-service; sync vs. async communication; event-driven architecture; choreography vs. orchestration (and sagas); the transactional outbox; at-least-once delivery and idempotency; retries, backoff, and jitter; dead-letter queues and poison messages; circuit breakers and bulkheads; distributed locks, leases, and fencing tokens; leader election; CQRS; event sourcing; API gateway and BFF; service discovery; authentication with JWT/JWKS; distributed tracing and context propagation; CAP, PACELC, and eventual consistency; partitioning and per-key ordering; backpressure; schema evolution and versioning; health checks (liveness vs. readiness vs. startup); contract testing; the Node.js event loop and I/O concurrency (why Node suits an I/O-bound pinger, and how to cap concurrency so it doesn't fall over); NestJS building blocks in a distributed system (modules, dependency injection, guards, interceptors, pipes, and the microservices transports); real-time delivery (polling vs. SSE vs. WebSockets); browser authentication (cookies vs. tokens, CORS, CSRF).

Use this template for every guide:

```
# <Concept>
**TL;DR:** 2–3 sentences.
## The problem it solves        (a concrete failure that happens without it, ideally from MicroSentinel)
## How it works                 (Mermaid diagram where it helps)
## In MicroSentinel             (exactly where and how it's used, linked to service/flow/ADR docs)
## Trade-offs & when NOT to use it
## Common misconceptions
## Interview questions          (3–6 questions, each with a concise model answer I could say out loud)
## Further reading              (canonical sources only, see rules below)
```

### ADR template

```
# ADR-NNNN: <decision as a short imperative phrase>
- Status: Proposed | Accepted | Superseded by ADR-XXXX
- Date: YYYY-MM-DD
- Phase: when this decision starts to matter
## Context
## Decision drivers
## Options considered            (at least two real options, each with pros and cons)
## Decision
## Consequences                 (positive, negative, and what becomes harder later)
## Revisit when                 (concrete triggers, e.g. "more than 10k monitors", "a second team")
## Learning choice?             (yes/no; if yes, what a pragmatic team would do instead)
## How to explain it in an interview   (a 30-second answer)
```

### Service doc template

Purpose (one sentence) and bounded context · Responsibilities · **Explicitly not responsible for** · Owned data (link to schema) · Sync APIs exposed and consumed · Events published and consumed (link to catalog) · Internal components · Scaling model (stateless? replicas? partitioning?) · Failure modes (what happens when each dependency is down, and what happens to others when this service is down) · Observability (key metrics, spans, log fields) · Patterns applied (links to concept guides) · Framework (Nest or Express, and why) and Nest structure: modules, providers, and which guards, interceptors, pipes, and exception filters it uses · Phase introduced.

### Roadmap (`docs/roadmap/phases.md`)

Refine the four phases from the brief. Add a **Phase 0**: monorepo, tooling, Compose base, shared conventions, one empty Nest service and the React app wired through the gateway, and CI that also runs locally (a single script or Compose target; a hosted CI workflow is optional). Add a **Phase 5 stretch**: e.g. Kubernetes with kind + Helm, load testing with k6, multi-location checks. The React app is not a stretch goal. It grows every phase, and each phase names its frontend slice. For each phase give:

- Goal and scope: which services, endpoints, events, and UI screens are in, and what is explicitly deferred.
- Ordered task breakdown, small enough to implement one item per sitting.
- **Definition of done** as verifiable checks: commands to run, expected outputs, what to see in the trace viewer.
- A demo script I can show in an interview.
- Concepts learned, linked to the concept guides.
- Interview talking points this phase unlocks.
- Known shortcuts taken in this phase and which later phase pays them off.

Fit the phases to my time budget, give rough effort estimates, and flag the phase most likely to blow up.

### Break-it lab (`docs/roadmap/break-it-lab.md`)

Write deliberate failure experiments, each with **hypothesis → how to induce it → what should happen → how to observe it → what it teaches**. Include at least: Notification killed during an outage; the broker restarted mid-flow; the same event delivered twice; a consumer that crashes on a poison message; a slow downstream (latency injected with something like Toxiproxy or `tc`); a Monitor replica paused past its lock or lease expiry (`docker pause`); a DB connection pool exhausted; the pinger's event loop blocked (a synchronous CPU task, or a target returning a huge response body); the browser's live connection dropped while incidents change; Auth down while incidents are being opened; Analytics read model deleted and rebuilt from history. Link each one to the phase where it becomes possible.

### Interview prep (`docs/learning/interview-prep.md`)

- A 60-second "tell me about a project you built" answer for my CV, plus a 2-minute and a 10-minute walkthrough of the architecture.
- A system-design-interview version: how I'd present MicroSentinel if asked to "design PagerDuty", covering requirements, estimates, API, data, scaling, and failure handling.
- A question bank of at least 40 questions grouped by topic, each linked to the doc that answers it. Include groups for Node.js, NestJS, React, SQL, and Docker questions that this project lets me answer from experience. Mark each question as _likely at junior level_ or _bonus depth_.
- "What would you change at 100× scale?" and "What would you do differently if you started over as a monolith?" sections.
- Weak spots an interviewer could probe, with honest answers.

## Writing rules

- **Be concrete.** Real event names (`monitor.state.changed.v1`), example JSON payloads, table DDL, HTTP status codes, specific timeout and retry values, specific thresholds. Avoid "we could use X or Y" outside the options section of an ADR.
- **Self-hosted and offline-capable.** Every infrastructure piece is an open-source Docker image pinned to a version. `docker compose up` must work with no accounts, API keys, or network access beyond pulling images. External channels get local stand-ins: an SMTP catcher such as Mailpit for email, a local webhook receiver with a simple inbox for seeing deliveries, and an SMS mock. Real Discord, Slack, or email providers are optional adapters, off by default. Include a local "flaky target" container that can be switched between up, down, slow, and flapping, so demos and tests never depend on real websites.
- **One source of truth for names.** The glossary and event catalog define names; every other doc uses them exactly. Event, topic, queue, and table names follow one documented convention.
- **Cross-link heavily** with relative Markdown links. Every service doc links its ADRs, flows, and concept guides; every concept guide links back to where it's used.
- **Use Mermaid for diagrams** (C4, sequence, state, flowchart) so they render on GitHub. The incident lifecycle gets a state diagram that includes illegal transitions and what happens when one is attempted.
- **Show failure paths alongside happy paths** in every flow. Each flow also lists the expected span tree for a trace.
- **Don't write application code.** Contracts and schemas (OpenAPI, AsyncAPI, proto, SQL DDL, TypeScript type definitions for event payloads and DTOs, a sketch of the Compose service list) are in scope because they're design artifacts. Short pseudocode is fine where it makes an algorithm clearer, such as outbox polling or lease acquisition.
- **Don't fabricate.** Cite only well-known canonical sources (e.g. Newman, _Building Microservices_; Kleppmann, _Designing Data-Intensive Applications_; Richardson, microservices.io; Fowler's articles; official docs for the chosen tools). Don't invent URLs. If you're unsure a link is exact, cite the title and author instead. Where you can, check library versions and APIs against current docs rather than memory; Nest and React tooling moves fast.
- **Keep files focused.** Roughly 400 lines at most per file. Split rather than sprawl.
- **Write for a learner who will be questioned.** Explain the _why_ before the _what_, define jargon on first use (and link to the glossary), and prefer short paragraphs, tables, and diagrams over walls of text.

## How to work

1. Check the repo state first. If anything exists already, build on it instead of overwriting it.
2. Write in dependency order: vision & scope → requirements → domain model & glossary → architecture → ADRs (resolve the design tensions _before_ writing service docs, so the service docs reflect decisions) → contracts & data → services → flows → cross-cutting → roadmap & break-it lab → concept guides → interview prep → README & indexes.
3. Don't stop to ask me questions mid-run. Where a preference says `decide`, make the call and record it in an ADR. Put anything that really needs my judgment in `docs/open-questions.md` with your recommended default, then carry on using that default.
4. Work on a branch named `docs/initial-architecture`. Commit after each numbered group in step 2 using Conventional Commit messages (`docs: …`). Don't push.
5. Finish with a **consistency pass**, and fix what you find instead of just listing it:
   - Every event in a flow or service doc exists in the event catalog and `asyncapi.yaml`, with matching names and fields.
   - Every service's published and consumed events match the catalog's producer and consumer lists.
   - Every sync call in a flow appears in the relevant OpenAPI or proto contract.
   - Every design tension above has an ADR, and the ADR index is complete.
   - No doc contradicts an accepted ADR (e.g. a service doc still mentioning an option the ADR rejected).
   - Every concept guide is linked from at least one service, flow, or ADR doc, and links back.
   - All relative links resolve.
   - Each phase's Definition of Done is verifiable by running something.
   - Every screen in `frontend/web-app.md` gets its data from a documented gateway endpoint or realtime channel.
   - Every component in `tech-stack.md` is a pinned, self-hostable image, and nothing in the docs requires an external account or API key.

## Final message to me

When you're done, reply with:

- The file tree you created.
- The stack as a table: component, choice, version, and Docker image.
- A table of the key decisions (ADR number, decision, one-line rationale, learning choice yes/no).
- The 3–5 open questions where my input would change the plan most.
- A suggested reading order for studying the docs over about a week before I start Phase 0.
