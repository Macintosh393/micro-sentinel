# Local deployment

Everything runs from Docker Compose on one machine, offline once images are pulled. No accounts, no API keys ([ADR-0028](../adr/0028-local-stand-ins.md)).

## Files

```
compose.yaml              # the whole system with production-style images
compose.dev.yaml          # override: dev build target, file sync, debug ports
.env.example              # copied to .env; development defaults only
infra/
  postgres/init/01-databases.sql   # creates the five databases and roles
  rabbitmq/rabbitmq.conf           # memory watermark, load_definitions
  rabbitmq/definitions.json        # shared exchanges, archive streams, dev user
  lgtm/                            # Grafana dashboards, optional collector override
  toxiproxy/toxiproxy.json         # proxies used by the break-it lab
```

## Commands

| Goal | Command |
|------|---------|
| Run the core stack, production-style images | `docker compose up -d --wait` |
| Develop with hot reload | `docker compose -f compose.yaml -f compose.dev.yaml up --watch` |
| Add observability | `docker compose --profile obs up -d --wait` |
| Add analytics | `docker compose --profile analytics up -d --wait` |
| Add chaos tools | `docker compose --profile chaos up -d` |
| Everything | `COMPOSE_PROFILES=obs,analytics,chaos docker compose up -d --wait` |
| Three workers | `docker compose up -d --scale monitor-worker=3` |
| Full reset, including data | `docker compose down -v` |

`--wait` returns only when every container with a health check is healthy, which makes it usable in scripts and the Definition of Done checks.

## Profiles

A Compose **profile** marks a service as optional. Services without a profile always start.

| Profile | Containers | Why optional |
|---------|-----------|--------------|
| *(none: core)* | gateway, web, auth, monitor, monitor-worker, incident, notification, realtime, postgres, redis, rabbitmq, mailpit, dev-stubs, flaky-target, the `*-migrate` jobs | The product |
| `analytics` | analytics, analytics-migrate | The core loop works without it; while it is off the archive streams simply accumulate, which is itself a demonstration |
| `obs` | lgtm | The largest single container |
| `chaos` | toxiproxy | Only for [break-it lab](../roadmap/break-it-lab.md) experiments |

Profiles are for **optional** components. They are not "one per phase": once Phase 2 code exists, the services need RabbitMQ to become ready, so it cannot be optional. The phase in which each container first appears is in [C4 containers](c4-containers.md).

## Compose sketch

A sketch of the service list, not the final file. Health checks, limits and environment are abbreviated.

```yaml
name: microsentinel

x-node-service: &node-service
  restart: unless-stopped
  networks: [internal]
  env_file: .env
  healthcheck:
    test: ["CMD", "node", "/app/healthcheck.mjs"]   # GETs /health/ready, exit 0 or 1
    interval: 10s
    timeout: 3s
    retries: 5
    start_period: 20s
    start_interval: 2s

services:
  gateway:
    <<: *node-service
    build: { context: ., dockerfile: apps/gateway/Dockerfile, target: runtime }
    networks: [public, internal]
    ports: ["8080:3000"]
    depends_on: { redis: { condition: service_healthy }, auth: { condition: service_healthy } }
    mem_limit: 256m

  web:
    build: { context: ., dockerfile: apps/web/Dockerfile, target: runtime }   # nginx:1.30.6-alpine
    networks: [internal]
    mem_limit: 64m

  auth:
    <<: *node-service
    build: { context: ., dockerfile: apps/auth/Dockerfile, target: runtime }
    depends_on:
      auth-migrate: { condition: service_completed_successfully }
      rabbitmq: { condition: service_healthy }
    mem_limit: 256m
  auth-migrate:
    build: { context: ., dockerfile: apps/auth/Dockerfile, target: runtime }
    command: ["node", "dist/migrate.js"]
    depends_on: { postgres: { condition: service_healthy } }
    restart: "no"

  monitor:        { <<: *node-service, environment: { MONITOR_ROLE: api } }      # + monitor-migrate
  monitor-worker: { <<: *node-service, environment: { MONITOR_ROLE: worker }, deploy: { replicas: 2 } }
  incident:       { <<: *node-service }                                          # + incident-migrate
  notification:   { <<: *node-service }                                          # + notification-migrate
  realtime:       { <<: *node-service }
  analytics:      { <<: *node-service, profiles: [analytics] }                   # + analytics-migrate

  postgres:
    image: postgres:18.7
    volumes: [pgdata:/var/lib/postgresql, ./infra/postgres/init:/docker-entrypoint-initdb.d:ro]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s, retries: 10 }
    mem_limit: 512m
  redis:
    image: redis:8.8.4-alpine
    command: ["redis-server", "--save", "", "--appendonly", "no", "--maxmemory", "64mb"]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s }
    mem_limit: 96m
  rabbitmq:
    image: rabbitmq:4.3.7-management-alpine
    volumes: [mqdata:/var/lib/rabbitmq, ./infra/rabbitmq:/etc/rabbitmq/conf.d:ro]
    ports: ["15672:15672"]
    healthcheck: { test: ["CMD", "rabbitmq-diagnostics", "-q", "check_port_connectivity"], interval: 10s, retries: 12, start_period: 30s }
    mem_limit: 640m
  mailpit:      { image: axllent/mailpit:v1.33.1, ports: ["8025:8025"], mem_limit: 64m }
  dev-stubs:    { build: tools/dev-stubs, ports: ["8090:3000"], mem_limit: 96m }
  flaky-target: { build: tools/flaky-target, ports: ["8091:3000"], mem_limit: 96m }
  lgtm:         { image: grafana/otel-lgtm:0.41.0, profiles: [obs], ports: ["3030:3000"], mem_limit: 1536m }
  toxiproxy:    { image: ghcr.io/shopify/toxiproxy:2.13.0, profiles: [chaos], ports: ["8474:8474"], mem_limit: 64m }

networks: { public: {}, internal: {} }
volumes: { pgdata: {}, mqdata: {} }
```

## Networks and ports

Two networks. `public` holds only the gateway. `internal` holds everything. Containers find each other by service name through Docker's built-in DNS ([service discovery](../learning/concepts/service-discovery.md)).

| Host port | Container | For |
|-----------|-----------|-----|
| **8080** | gateway | The application: `http://localhost:8080` |
| 15672 | rabbitmq | Management UI (user `ms`, password `ms-dev`) |
| 8025 | mailpit | Email inbox UI |
| 8090 | dev-stubs | Webhook inbox, SMS and chat inbox UI, failure switches |
| 8091 | flaky-target | Control API: switch mode up / down / slow / flapping |
| 3030 | lgtm | Grafana (`obs` profile) |
| 8474 | toxiproxy | Toxiproxy API (`chaos` profile) |
| 5432, 5672, 3001–3007 | postgres, rabbitmq, each service | Published **only** by `compose.dev.yaml`, for a DB client and for debugging a service directly |

Inside the network every Node service listens on port 3000.

## Startup order and health checks

Compose starts containers in dependency order using `depends_on` with conditions:

1. `postgres`, `redis`, `rabbitmq` start and become **healthy**.
2. Each `*-migrate` job runs TypeORM migrations and exits 0 (`service_completed_successfully`). This is the Compose equivalent of a Kubernetes init container. Migrations never run inside the service process, so scaling to several replicas cannot race.
3. Services start. Each one declares its queues and bindings, starts its outbox relay and consumers, and only then reports **ready**.
4. `gateway` starts last.

| Endpoint | Meaning | Checks | Used by |
|----------|---------|--------|---------|
| `GET /health/live` | The process is running and its event loop responds | Nothing external | A restart decision (Kubernetes liveness in Phase 5) |
| `GET /health/ready` | The service can do useful work now | Database ping, broker connection, JWKS loaded | Compose `healthcheck`, `depends_on` |

Compose has one health check per container and no separate startup probe; `start_period` plus `start_interval` gives the same effect. Why the two endpoints differ, and why liveness must not check dependencies: [health checks](../learning/concepts/health-checks.md).

## Memory budget

Estimates for dev mode, to be replaced with measurements (`docker stats`) at the end of Phase 0 and each later phase. `mem_limit` is set a little above each estimate so that a leak kills one container instead of the machine.

| Container | Estimate (MB) | | Container | Estimate (MB) |
|-----------|---------------|-|-----------|---------------|
| postgres | 350 | | notification | 200 |
| rabbitmq | 400 | | realtime | 160 |
| redis | 40 | | web (nginx / Vite dev) | 20 / 300 |
| gateway | 180 | | mailpit | 40 |
| auth | 200 | | dev-stubs | 70 |
| monitor | 200 | | flaky-target | 60 |
| monitor-worker × 2 | 360 | | **Core total** | **≈ 2.5–2.8 GB** |
| incident | 200 | | | |

| Optional | Estimate (MB) |
|----------|---------------|
| analytics | 200 |
| lgtm | 900–1300 |
| toxiproxy | 30 |
| **Everything** | **≈ 4–4.5 GB** |

That leaves headroom inside 8 GB for Docker Desktop itself, the editor and a browser. Cap WSL2 so it cannot starve Windows: in `%UserProfile%\.wslconfig` set `[wsl2]` `memory=8GB`. What was traded away to fit the budget is in [ADR-0018](../adr/0018-single-postgres-and-ram-budget.md).

## Images

### Production-style image (multi-stage)

A **multi-stage build** uses several `FROM` stages so that compilers and dev dependencies never reach the final image.

| Stage | Base | Does |
|-------|------|------|
| `base` | `node:26.13.1-bookworm-slim` | Enables pnpm through Corepack |
| `deps` | base | `pnpm fetch` using only the lockfile, so this layer is cached until the lockfile changes |
| `build` | deps | Copies sources, `pnpm install --offline`, builds the one app and the packages it uses |
| `prune` | build | `pnpm deploy --prod` to produce a folder with only that app and its production dependencies |
| `runtime` | `node:26.13.1-bookworm-slim` | Copies the pruned folder, runs as the non-root `node` user, `CMD ["node", "dist/main.js"]` |
| `dev` | deps | Used only by `compose.dev.yaml`: runs `nest start --watch` |

Debian slim rather than Alpine: `argon2` and other native modules ship prebuilt binaries for glibc, and musl differences are a distraction this project does not need. `web` builds with Node and copies `dist/` into `nginx:1.30.6-alpine`.

Run `node` directly, not `pnpm start`: the process must be PID 1's direct child so that `SIGTERM` from `docker stop` reaches it and Nest's shutdown hooks run (drain HTTP, stop consuming, finish in-flight messages, close connections).

### Dev images and hot reload on Windows

The problem: with the repository on the Windows filesystem (`C:\…`), files are shared into Linux containers through a translation layer. Linux file-change notifications (inotify) do not fire for changes made on the Windows side, and file access is slow. Watchers either miss changes or fall back to polling the whole tree.

| Option | How | Verdict |
|--------|-----|---------|
| **A. Repository inside WSL2** (`\\wsl$\Ubuntu\home\<you>\micro-sentinel`), edited with VS Code's WSL extension | Bind mounts are native Linux mounts; inotify works; fast | **Recommended** |
| **B. Compose Watch** (`develop.watch` with `action: sync`) | Docker watches host files and copies changes into the container; no bind mount | **Default in `compose.dev.yaml`**; works from either filesystem |
| C. Polling (`CHOKIDAR_USEPOLLING=true`, Vite `server.watch.usePolling`) | Watcher re-scans on a timer | Fallback only: high CPU, slow |
| D. Infrastructure in Docker, services on the host with `pnpm dev` | `docker compose up postgres rabbitmq redis mailpit …` then run Node on Windows | Fine for working on one service; services then use `localhost` ports |

`compose.dev.yaml` uses Compose Watch: `sync` for `src/` (the Nest watcher inside the container sees a normal Linux file change and recompiles with SWC), and `rebuild` when `package.json` or the lockfile changes. The repository is currently on `C:\`; moving it into WSL2 is recommended in [open questions](../open-questions.md).

Other Windows notes:

- Set `git config core.autocrlf input` and add a `.gitattributes` with `* text=auto eol=lf`. Shell scripts with CRLF line endings fail inside Linux containers with confusing errors.
- `node_modules` must never be bind-mounted from Windows into a Linux container (native modules differ). Compose Watch avoids this; with bind mounts, use a named volume for `node_modules`.

## What is different from production

| Local | A real deployment would |
|-------|-------------------------|
| One Postgres server | Use a managed database or at least one server per service group, with backups |
| Single-node RabbitMQ; quorum queues with one replica | Run three nodes so quorum queues survive a node loss |
| Plain HTTP on localhost | Terminate TLS in front of the gateway |
| Secrets in `.env` | Use a secret manager ([configuration](../cross-cutting/configuration.md)) |
| `lgtm` all-in-one | Run separate, persistent telemetry backends |
| Compose | Kubernetes or similar (Phase 5 stretch) |
