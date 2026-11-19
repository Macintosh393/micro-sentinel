# Health checks: liveness, readiness, startup

**TL;DR:** **Liveness** asks "is this process stuck and should it be restarted?". **Readiness** asks "can it do useful work right now, should it receive traffic?". **Startup** asks "has it finished booting?" so slow starters aren't killed early. Mixing them up turns one failing dependency into a fleet of restarting services.

## The problem it solves

A service whose database is briefly unreachable should stop receiving requests until it recovers, not be killed: restarting won't fix the database, and if every service restarts because the database blipped, recovery becomes a stampede. Conversely, a process whose event loop is deadlocked will never recover by waiting; it needs a restart. One endpoint can't express both.

## How it works

| Probe | Question | Should check | On failure | Must NOT check |
|-------|----------|--------------|-----------|----------------|
| **Liveness** | Is the process alive and responsive? | The event loop answers; maybe an internal watchdog | Restart the container | External dependencies |
| **Readiness** | Can it serve now? | Own DB reachable, broker connected, required config/keys loaded, background loops ticking | Stop routing traffic to it; don't restart | Other services' health |
| **Startup** | Has it finished starting? | Same as readiness, once | Keep waiting (up to a limit), then restart | — |

Rules of thumb:

- Liveness should almost never fail. If it checks the database, a database outage restarts every service at once.
- Readiness checks **own** dependencies only. A service that reports not-ready because *another service* is down spreads the outage.
- Health endpoints must be cheap and time-bounded (a check that hangs is a failure).

In Kubernetes these map to `livenessProbe`, `readinessProbe`, `startupProbe`. Docker Compose has a single `healthcheck` per container plus `start_period`/`start_interval`, used for `depends_on: condition: service_healthy` (startup ordering) and the `--wait` flag.

## In MicroSentinel

- Every service exposes `GET /health/live` and `GET /health/ready` via `@nestjs/terminus` ([deployment-local](../../architecture/deployment-local.md#startup-order-and-health-checks)).
- Readiness checks: DB `SELECT 1` (≤ 1 s), broker connected, JWKS loaded, outbox relay ticked in the last 10 s, and in workers event-loop delay p99 < 200 ms ([resilience](../../cross-cutting/resilience.md#health-and-readiness)).
- Compose `healthcheck` calls readiness through a tiny `healthcheck.mjs` (slim images have no `curl`).
- Startup order: infrastructure healthy → `*-migrate` jobs `service_completed_successfully` → services → gateway.
- No service's readiness depends on another service: incident is ready while notification is down, by design.
- Analytics' public API returning `503` during a rebuild is **not** a readiness failure; the process is fine.

## Trade-offs & when NOT to use it

- Readiness that is too strict (checking optional things) takes healthy instances out of rotation.
- Readiness that is too loose sends traffic to instances that will fail.
- Deep checks on every probe add load; cache results for a second or two if checks are expensive.

## Common misconceptions

- **"One `/health` endpoint is enough."** It can't distinguish "restart me" from "don't send me traffic".
- **"Liveness should check the database."** Then a database outage restarts everything.
- **"Healthy means correct."** A ready service can still have a bug; health is about the ability to serve.

## Interview questions

1. **Liveness vs readiness?**
   "Liveness says the process is alive, and failing it gets the container restarted, so it should only check things a restart would fix, like a stuck event loop. Readiness says it can serve right now: my services check their own database, broker connection and that JWT keys are loaded. Failing readiness stops traffic but doesn't restart."
2. **Should a service's readiness depend on another service?**
   "No. If incident reported not-ready because notification was down, one outage would cascade. Each service checks only its own dependencies; that's why messages wait in queues instead."
3. **What's a startup probe for?**
   "Slow-starting apps: it gives them time to boot before liveness kicks in, so they aren't killed in a restart loop. In Compose I approximate it with `start_period`."

## Further reading

- Kubernetes documentation: "Configure Liveness, Readiness and Startup Probes".
- Docker documentation: Compose file reference, `healthcheck` and `depends_on`.
- NestJS documentation: "Health checks (Terminus)".

Used in: [deployment-local](../../architecture/deployment-local.md), [resilience](../../cross-cutting/resilience.md), [gateway](../../services/gateway.md), [realtime](../../services/realtime.md), [phase 0](../../roadmap/phase-0.md).
