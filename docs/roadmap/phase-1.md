# Phase 1: Walking skeleton

**Effort:** 40–50 h (4–5 weeks). Overview: [phases](phases.md).

## Goal

The product loop works without a broker: a user registers, creates monitors, workers check them, a confirmed outage opens an incident, the user acknowledges and resolves it in the browser. Every service verifies JWTs.

## Scope

**In:**

| Area | Content |
|------|---------|
| auth | Register, login, refresh (rotation + reuse detection), logout, `GET`/`PATCH /v1/users/me`, `GET /v1/orgs/current`, JWKS, signing keys ([auth-org](../services/auth-org.md)) |
| `auth-guard` package | `JwtAuthGuard`, `RolesGuard`, `@Roles`, `@CurrentUser`, `@Public`, JWKS cache |
| gateway | Full routing table, JWT middleware, Redis rate limits, origin check, security headers ([gateway](../services/gateway.md)) |
| monitor | CRUD, pause/resume, recent checks; one worker; **leases with fencing**; confirmation policy; basic SSRF guard (scheme, port, resolved-IP check, allow-list); concurrency cap ([monitor](../services/monitor.md)) |
| incident | Incidents table and state machine; list (keyset), detail, acknowledge, resolve; `POST /internal/v1/monitor-signals` ([incident](../services/incident.md)) |
| `flaky-target` | Modes `up`, `down`, `timeout`, `slow`, `flapping` |
| Redis | For the gateway |

Endpoints: everything in [auth.yaml](../contracts/openapi/auth.yaml) except members and API keys; all of [monitor.yaml](../contracts/openapi/monitor.yaml); incidents part of [incident.yaml](../contracts/openapi/incident.yaml).

Events: none. The worker calls incident synchronously (the deliberate shortcut).

**Deferred:** broker, outbox, timeline, notification, realtime (Phase 2); escalation, schedules, members, API keys, several workers (Phase 3); analytics, tracing (Phase 4).

**Frontend slice:** `/login`, `/register`, session handling (in-memory token, startup refresh, single-flight refresh, replay on `401`); `/monitors` list, `/monitors/new`, `/monitors/:id`, `/monitors/:id/edit`; `/incidents` list with 5 s polling and status filter; `/incidents/:id` with acknowledge/resolve (optimistic); `/dashboard` counts; `/settings/profile`. Four states on every screen.

## Tasks (one per sitting)

1. auth: entities and migrations (`organizations`, `users`, `memberships`, `refresh_tokens`, `signing_keys`); `auth-migrate` job.
2. auth: register + login with Argon2id; uniform `401`; API tests.
3. auth: signing keys, JWT issuing, JWKS endpoint.
4. auth: refresh rotation, reuse detection, 10 s grace, logout; cookie attributes; API tests for each path.
5. `auth-guard` package: guard, decorators, JWKS cache with unknown-`kid` refetch; unit tests with a test key pair.
6. gateway: full routing table; JWT middleware; Redis + throttler; origin check; security headers.
7. web: API client (token in memory, single-flight refresh, replay once), `SessionProvider`, protected layout, login and register pages.
8. monitor: entity, migration, CRUD with explicit-column updates; API tests incl. tenancy.
9. monitor: `ConfirmationPolicy` as a pure function + exhaustive unit tests.
10. monitor: `safe-fetch` package v1 (timeouts, no redirects, no body, resolved-IP check, allow-list) + classifier tests.
11. monitor-worker: claim query, `p-limit` cap, complete with fencing; integration tests for two concurrent claimers and fencing.
12. incident: entity, migration (with the unique partial index), `IncidentStateMachine` + tests, `applyTransition` with optimistic locking.
13. incident: `/internal/v1/monitor-signals` (idempotent by `eventId` and `stateVersion` via `monitor_cursors`); worker calls it with 3 retries.
14. incident: list (keyset pagination), detail, acknowledge, resolve; API tests incl. `409`.
15. web: monitors list, form (React Hook Form + zod, server errors mapped), detail with recent checks.
16. web: incidents list (polling, filter in URL, load more), detail, acknowledge/resolve optimistic with rollback.
17. web: dashboard counts; profile; component tests (four states) for monitors and incidents.
18. `flaky-target` with control API and UI; Compose wiring; `.env` allow-list.
19. Seed script; run the demo; measure memory; fix rough edges.

## Definition of Done

| # | Check | Expected |
|---|-------|----------|
| 1 | `pnpm ci:local` | Exit 0; includes API tests for auth, monitor, incident and the lease/fencing integration tests |
| 2 | `docker compose up -d --wait` | All healthy (gateway, web, auth, monitor, monitor-worker, incident, postgres, redis, flaky-target) |
| 3 | `curl -s localhost:8080/.well-known/jwks.json \| jq '.keys[0].alg'` | `"RS256"` |
| 4 | Register via `curl` (`POST /api/v1/auth/register`) | `201` with `accessToken`; `Set-Cookie: ms_refresh=…; HttpOnly; SameSite=Strict` |
| 5 | `curl -s localhost:8080/api/v1/monitors` without a token | `401` problem+json |
| 6 | `curl -s http://localhost:3002/v1/monitors -H "authorization: Bearer $(node scripts/forge-token.mjs)"` **directly** on the dev port, with a well-formed token signed by a throwaway key | `401`: the service verifies, not just the gateway |
| 7 | Create a monitor on `flaky-target`, then `PUT localhost:8091/control/checkout {"mode":"down"}` | Within 90 s (30 s interval) `GET /api/v1/incidents?status=OPEN` returns one incident |
| 8 | `docker compose exec postgres psql -U postgres -d monitor_db -c "select count(*) from monitor_checks where checked_at > now()-interval '1 minute'"` | Roughly 60 / interval per monitor (no double checks with one worker) |
| 9 | Acknowledge twice with `curl` | Both `200`, same `version` |
| 10 | Set mode `up` | Incident `RESOLVED`, `resolution: RECOVERED`, within ~40 s |
| 11 | Create a monitor with URL `http://postgres:5432` | `422 BLOCKED_TARGET` |
| 12 | 15 rapid logins with a wrong password | `429` after 10 |
| 13 | Browser: log in, reload, still logged in; DevTools → Application → cookies shows `ms_refresh` as HttpOnly; `localStorage` is empty | ✓ |
| 14 | **The motivating failure:** `docker compose stop incident`, set a target `down`, wait 2 min, start incident | No incident is ever opened for that outage (the signal was lost after retries). Write down what you saw: it is the reason for Phase 2 |

## Demo script (3 minutes)

1. Register, create a monitor on `flaky-target`, watch it go `UP`.
2. Switch the target `down`; narrate the three failures on the monitor page; the incident appears within 5 s.
3. Acknowledge, then switch `up`: auto-resolved.
4. Show that a token from another organization gets `404` on this incident.
5. End with DoD #14: "this is the problem with synchronous calls between services, and why the next phase adds a broker".

## Concepts learned

[Database-per-service](../learning/concepts/database-per-service.md) · [JWT and JWKS](../learning/concepts/jwt-and-jwks.md) · [browser authentication](../learning/concepts/browser-authentication.md) · [sync vs async](../learning/concepts/sync-vs-async-communication.md) (feeling the sync cost) · [Node event loop](../learning/concepts/node-event-loop.md) · [distributed locks, leases, fencing](../learning/concepts/distributed-locks-leases-fencing.md) · [NestJS building blocks](../learning/concepts/nestjs-building-blocks.md) (guards, pipes, interceptors, filters) · [idempotency](../learning/concepts/idempotency-and-at-least-once.md) (idempotent commands)

## Interview talking points unlocked

- JWT with RS256 and JWKS; why every service verifies; where the browser keeps tokens; CSRF and CORS.
- `SELECT … FOR UPDATE SKIP LOCKED` and fencing tokens.
- Optimistic locking with a version column; idempotent `POST` commands; `409` vs `200`.
- Keyset vs offset pagination.
- Why `synchronize: true` is off; migrations as a one-shot job.
- Temporal coupling, demonstrated (DoD #14).

## Shortcuts taken

| Shortcut | Paid off in |
|----------|-------------|
| Synchronous worker → incident signal, lost when incident is down | Phase 2 (outbox + broker) |
| No incident timeline | Phase 2 |
| No paging at all | Phase 2 |
| Polling instead of push | Phase 2 |
| One worker only | Phase 3 |
| SSRF guard without per-host cap and full test list | Phase 3 |
| No members or API keys | Phase 3 |
| Monitor deletion does not resolve the incident (no event yet) | Phase 2 |
