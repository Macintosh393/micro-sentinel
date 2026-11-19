# Phase 0: Foundations

**Effort:** 15–20 h (about 2 weeks). Overview: [phases](phases.md).

## Goal

A repository where every later sitting starts from working tooling: one command brings up a skeleton stack, one command runs all checks, and a browser request travels browser → gateway → service and back.

## Scope

**In:**

- pnpm workspace with `apps/`, `packages/`, `tools/` ([repo layout](../cross-cutting/repo-layout.md)).
- Shared TypeScript, lint (oxlint), format (Prettier), Vitest configs; `.gitattributes` for LF.
- Packages: `observability` (pino logger module, request-ID middleware, problem+json filter), `testing` (empty but wired).
- Apps: `gateway` (routing only), `auth` (empty Nest app: health + one `GET /v1/ping`), `web` (Vite + React + Router + Mantine shell).
- `compose.yaml` with `postgres`, `gateway`, `auth`, `web`; `compose.dev.yaml` with Compose Watch.
- Multi-stage Dockerfile template; health checks; `--wait` working.
- `pnpm ci:local`, `scripts/check-deps.mjs`, secret scan.
- `.env.example`; Postgres init script with all five databases and roles (cheap now, avoids migrations later).

**Deferred:** auth logic, JWT (Phase 1); Redis (Phase 1); broker (Phase 2); OTel SDK (Phase 4).

**Frontend slice:** app shell (header, nav placeholders), `/` → `/dashboard` page that calls `GET /api/v1/ping` and shows the response or the error state; `*` not-found page.

## Tasks (one per sitting)

1. Read [overview](../architecture/overview.md), [ADR-0002](../adr/0002-service-boundaries.md), [ADR-0003](../adr/0003-monorepo-pnpm-workspaces.md). Decide the repository location (WSL2 filesystem recommended, [open questions](../open-questions.md)). Install Node 26, enable Corepack/pnpm, configure `.wslconfig`.
2. Create the workspace: root `package.json`, `pnpm-workspace.yaml` with a catalog, `tsconfig.base.json`, `.npmrc`, `.gitattributes`, oxlint, Prettier.
3. Generate `apps/auth` with the Nest CLI as an **ESM** project; adapt to the workspace (catalog versions, base tsconfig, SWC builder, Vitest with `unplugin-swc`). Add `GET /v1/ping` and one unit test.
4. Build `packages/observability`: `LoggerModule` (nestjs-pino), `RequestIdMiddleware`, `ProblemDetailsFilter` (RFC 9457 with `code`). Use it in auth.
5. Health: `@nestjs/terminus` with `/health/live` and `/health/ready` (ready = DB ping). `healthcheck.mjs` for the container.
6. Dockerfile (multi-stage, `pnpm fetch` → build → `pnpm deploy --prod` → slim runtime, non-root, `node` as entrypoint). Build and run auth alone.
7. `compose.yaml` with postgres (+ init script for 5 DBs/roles), auth; `docker compose up -d --wait` works.
8. `apps/gateway`: Nest app, `bodyParser: false`, `http-proxy-middleware` routing `/api/v1/ping` → auth and `/` → web, 10 s timeout, `502`/`504` as problem+json. Add to Compose.
9. `apps/web`: Vite + React + Router + TanStack Query + Mantine; shell; dashboard calling `/api/v1/ping` through a minimal API client; Vite dev proxy for `/api`. nginx runtime image.
10. `compose.dev.yaml`: dev targets with `develop.watch` (sync `src/`, rebuild on `package.json`); verify a code change in auth reloads within a few seconds on Windows.
11. `pnpm ci:local`: format, lint, typecheck, `check-deps`, secret scan, unit tests, build. Optional `.github/workflows/ci.yml` calling it.
12. Measure memory; write the first numbers into [deployment-local](../architecture/deployment-local.md#memory-budget). Record the demo.

## Definition of Done

| # | Check | Expected |
|---|-------|----------|
| 1 | `pnpm install --frozen-lockfile && pnpm ci:local` | Exit 0 |
| 2 | `docker compose up -d --wait` | Exit 0 within 120 s; `docker compose ps` shows `healthy` for postgres, auth, gateway |
| 3 | `curl -s localhost:8080/api/v1/ping` | `{"pong":true,…}` with an `X-Request-Id` response header |
| 4 | `curl -s localhost:8080/api/v1/nope` | `404`, `Content-Type: application/problem+json`, body has `code` |
| 5 | `docker compose stop auth && curl -si localhost:8080/api/v1/ping` | `502` problem+json; `docker compose start auth` restores it |
| 6 | Open `http://localhost:8080` | Dashboard shows the ping result; with auth stopped it shows the error state with a request ID and a Retry button |
| 7 | `docker compose logs auth --no-log-prefix \| head -1` | One JSON line containing `service`, `request_id` |
| 8 | With `compose.dev.yaml --watch`, edit the ping message | New response within ~5 s, no manual restart |
| 9 | `node scripts/check-deps.mjs` after adding `"@microsentinel/gateway"` to auth's dependencies | Fails with a clear message (then revert) |
| 10 | `docker stats --no-stream` | Total < 1 GB |
| 11 | `docker image ls` | auth runtime image < 250 MB |

## Demo script (2 minutes)

1. `docker compose up -d --wait`, show all healthy.
2. Browser: dashboard shows "pong" from a service behind the gateway.
3. Stop auth, refresh: error state with request ID. Find that ID in `docker compose logs gateway`.
4. Show the folder structure and the dependency rule failing.
5. "Every service is reachable only through the gateway; containers find each other by DNS name."

## Concepts learned

[Microservices vs modular monolith](../learning/concepts/microservices-vs-modular-monolith.md) · [bounded contexts](../learning/concepts/bounded-contexts-and-ddd.md) · [service discovery](../learning/concepts/service-discovery.md) · [health checks](../learning/concepts/health-checks.md) · [API gateway](../learning/concepts/api-gateway-and-bff.md) · [NestJS building blocks](../learning/concepts/nestjs-building-blocks.md) (modules, middleware, filters)

## Interview talking points unlocked

- Monorepo vs polyrepo, and how you stop a monorepo coupling services.
- Multi-stage Docker builds and why the runtime image is small; why `node` and not `npm start` as PID 1's child.
- Liveness vs readiness.
- What happens to a request when an upstream is down (`502` vs `504`).
- Hot reload in containers on Windows and why file watching breaks across filesystems.

## Shortcuts taken

| Shortcut | Paid off in |
|----------|-------------|
| Gateway does no auth | Phase 1 |
| No Redis, no rate limiting | Phase 1 |
| Logs only; no tracing | Phase 4 |
| One shared Postgres superuser password in `.env` for init | Kept (local only); production note in [security](../cross-cutting/security.md) |
