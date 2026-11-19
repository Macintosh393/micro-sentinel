# Open questions, risks and mitigations

Decisions that need your judgement. Each has a **recommended default**, which the rest of the documentation already assumes. If you choose differently, update the linked docs (and write an ADR if it changes an accepted decision).

## Questions that would change the plan most

| # | Question | Recommended default (applied) | What changes if you choose otherwise |
|---|----------|-------------------------------|--------------------------------------|
| Q1 | **Scope target.** Is the goal to finish Phase 4, or to reach a polished Phase 2 sooner and start interviewing? | Plan for Phases 0–4 (~5 months at 10 h/week), but treat a **polished Phase 2 plus demo** as the first "interview-ready" milestone (~3 months). Phases 3–4 continue while interviewing | Stopping at Phase 2: drop multi-worker proof, retry tiers, analytics and tracing from the CV story; the [interview prep](learning/interview-prep.md) stories shrink accordingly |
| Q2 | **Repository location on Windows.** The repo is on `C:\`; file watching in containers is slow or broken across the Windows/Linux boundary | Move the clone into the WSL2 filesystem (`~/micro-sentinel`) and edit with VS Code's WSL extension. Compose Watch is configured anyway and works from either location | Staying on `C:\`: rely on Compose Watch only; expect slower rebuilds; host-run services (option D in [deployment-local](architecture/deployment-local.md#dev-images-and-hot-reload-on-windows)) become more attractive |
| Q3 | **Learning choices vs time.** Four big learning choices cost time: hand-written gateway ([ADR-0020](adr/0020-nestjs-api-gateway.md)), in-repo amqplib wrapper ([ADR-0007](adr/0007-amqplib-over-nest-microservices.md)), Pact ([ADR-0027](adr/0027-testing-stack.md)), microservices at all ([ADR-0002](adr/0002-service-boundaries.md)) | Keep all four; they are the interview material | Use Traefik for the gateway (−4 h, lose a Node/Nest talking point) or `@golevelup/nestjs-rabbitmq` (−8–10 h, Phase 2 much less risky, lose the "acks and confirms by hand" story) |
| Q4 | **No synchronous service-to-service calls, so no gRPC.** Do target employers expect hands-on gRPC? | No gRPC in the core; optional exercise [S8](roadmap/phase-5.md) | If gRPC matters: do S8 in Phase 3 (6–8 h); architecture unchanged |
| Q5 | **Test runner.** Vitest everywhere, though many existing Nest codebases (and some job specs) use Jest | Vitest ([ADR-0027](adr/0027-testing-stack.md)); the APIs are nearly identical | If an employer you target is Jest-heavy: port one service's tests to Jest as a 2–3 h exercise |

## Other open questions

| # | Question | Recommended default (applied) | Where |
|---|----------|-------------------------------|-------|
| Q6 | TypeScript 6.0.3 or 7.x? | **6.0.3**: Nest 12 tooling requires ≥ 6, and decorator-metadata support across Nest + TypeORM + class-validator on TS 7 is unverified | [tech-stack](architecture/tech-stack.md) |
| Q7 | Node 26 (images) while the dev machine has 24.22.1? | Install Node 26 LTS locally; 24.22 also works for Nest 12 if you prefer to wait | [tech-stack](architecture/tech-stack.md) |
| Q8 | A user manually resolves an incident while the monitor is still `DOWN`. Open a new incident immediately, or wait until the monitor recovers and fails again? | **Wait** (no new incident until a fresh `DOWN`). Simple, avoids an open/resolve loop; the UI warns "monitor is still down" | [domain model](02-domain-model.md#incident-lifecycle) |
| Q9 | Incident `opened_at`/`resolved_at`: monitor's `changedAt` or processing time? | **Monitor's `changedAt`**, so outages of incident itself don't distort MTTA/MTTR; processing delay is a separate metric | [service-down-recovery](flows/service-down-recovery.md) |
| Q10 | Flap suppression (one incident for a flapping monitor) | Not built; slow flapping produces one incident per down period. Stretch [S7](roadmap/phase-5.md) | [ADR-0009](adr/0009-check-results-and-state-changes.md) |
| Q11 | Client idempotency keys on `POST /v1/monitors` (to make retried creates safe) | Not built; the unique name gives `409` on a retried create | [register-monitor](flows/register-monitor.md) |
| Q12 | PostgreSQL row-level security as a second tenancy layer | Not built; stretch [S6](roadmap/phase-5.md) | [security](cross-cutting/security.md) |
| Q13 | Persist the last JWKS to disk so a service can start while auth is down | Not built; stretch [S11](roadmap/phase-5.md) | [break-it lab #10](roadmap/break-it-lab.md) |
| Q14 | Invitations and password reset (need email to users outside incidents) | Not built; admins create members with a temporary password | [auth-org](services/auth-org.md) |
| Q15 | Keyword or JSON-body checks | Out of scope; would reopen the event-loop and memory risks | [ADR-0026](adr/0026-pinger-safety.md) |
| Q16 | Users in more than one organization | Schema allows it; v1 assumes one; tokens carry one `org` | [requirements](01-requirements.md#assumptions) |
| Q17 | On-call overrides, multiple escalation policies, per-monitor policies | Out of scope | [incident-schema](data/incident-schema.md) |
| Q18 | Rate limiter when Redis is down: fail open or closed? | **Open** locally; a public deployment should fail closed on login | [resilience](cross-cutting/resilience.md#degraded-modes) |
| Q19 | Redis 8 (AGPLv3 among its licence options) or Valkey | Redis 8.8.4; Valkey is a drop-in swap if the licence matters to you | [tech-stack](architecture/tech-stack.md) |
| Q20 | Hosted CI (GitHub Actions) | Optional; `pnpm ci:local` is authoritative | [testing strategy](cross-cutting/testing-strategy.md) |

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Phase 2 overruns (many new concepts at once) | High | Schedule slip, motivation | Spikes first, messaging package before services, one end-to-end flow before widening, explicit cut list ([phases](roadmap/phases.md#the-phase-most-likely-to-blow-up-phase-2)) |
| Very recent major versions (Nest 12, TypeORM 1.x, React Router 8, Vite 8, Vitest 5, TS 6) have rough edges and fewer answers online | Medium | Hours lost to tooling | Pin exact versions; read migration guides first; if a tool blocks for more than one sitting, drop one major version and note it in [tech-stack](architecture/tech-stack.md) |
| ESM + decorators + Vitest setup friction | Medium | Phase 0 slips | Solve once in Phase 0 task 3 with SWC; copy the working config to every app |
| The hand-written amqplib wrapper has reconnect or ack-timing bugs | Medium | Lost or stuck messages, confusing debugging | Integration tests from day one (broker restart, crash before ack); switch to `@golevelup/nestjs-rabbitmq` if a third reconnect bug appears ([ADR-0007](adr/0007-amqplib-over-nest-microservices.md)) |
| Memory budget exceeded on the dev machine | Low–medium | Swapping, slow machine | Measure each phase; profiles; one worker; prod-style web image; host-run services ([ADR-0018](adr/0018-single-postgres-and-ram-budget.md)) |
| Windows file watching in containers | High if repo stays on `C:\` | Slow feedback loop | Q2 |
| Docs drift from code | High over months | Interview answers contradict the repo | Update docs in the same commit as behaviour changes; a new ADR when a decision changes; rerun the [consistency rules](../README.md#consistency-rules) at the end of each phase |
| Over-investment in infrastructure, too little product | Medium | Weak demo | Every phase has a demo script and a frontend slice; record the demo before starting the next phase |
| Security-sensitive code (SSRF classifier, refresh rotation) has a bug | Medium | Wrong lesson; in a real deployment, a vulnerability | Exhaustive unit tests listed in [testing strategy](cross-cutting/testing-strategy.md#unit-tests); never expose the stack beyond localhost |
| Telemetry packages (`@opentelemetry/*`) version mismatches | Medium | Missing spans | Pin all OTel packages together; upgrade as a set |

## How to resolve a question

1. Decide.
2. If it changes an accepted ADR, write a new ADR that supersedes it.
3. Update the docs listed in the "Where" column.
4. Move the row to a "Resolved" table at the bottom of this file with the date and the decision.

## Resolved

*(none yet)*
