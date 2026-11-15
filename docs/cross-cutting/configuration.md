# Configuration

How services are configured: twelve-factor style environment variables, validated at startup, with one documented default per setting. Background: The Twelve-Factor App (Heroku, "III. Config").

## Rules

1. **Configuration comes from the environment**, never from code branches like `if (env === 'production')`. The same image runs everywhere; only variables change.
2. **Validated at startup.** Each service declares a schema; a missing or malformed variable stops the process with a clear message before it opens any port. Failing fast at boot is better than failing at 3 a.m. on first use.
3. **Typed access.** Code reads a typed config object (`config.get('database.url')` returns a string, `checkConcurrency` a number), never `process.env` directly. Only the config module touches `process.env`.
4. **Defaults are safe for local development**, and the few that are unsafe elsewhere are named so (`ALLOW_FAST_INTERVALS`, `COOKIE_SECURE=false`, `SSRF_ALLOW_HOSTS`).
5. **Secrets are configuration, but not in the repository** ([security](security.md#secrets-in-local-development)).
6. **Names are service-agnostic** where the meaning is the same (`DATABASE_URL`, not `INCIDENT_DATABASE_URL`). Compose sets different values per service.

## Mechanism

`@nestjs/config` with a schema. Nest 12's config module validates through Standard Schema, so the schema is written with **zod** here (the one backend use of zod; class-validator remains the choice for DTOs and events, [ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)). Each service exports `configSchema` from `src/config.ts`; shared fragments (database, broker, OTel, JWT) come from the shared packages.

```ts
// sketch — apps/incident/src/config.ts
export const configSchema = z.object({
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.string().url(),
  DB_POOL_MAX: z.coerce.number().int().min(1).max(50).default(10),
  AMQP_URL: z.string().url(),
  JWKS_URL: z.string().url().default('http://auth:3000/.well-known/jwks.json'),
  JWT_ISSUER: z.string().default('microsentinel-auth'),
  JWT_AUDIENCE: z.string().default('microsentinel-api'),
  ESCALATION_POLL_MS: z.coerce.number().int().min(500).default(5000),
  // …
});
```

## Shared variables

| Variable | Default | Used by | Meaning |
|----------|---------|---------|---------|
| `PORT` | `3000` | all Node services | HTTP port inside the container |
| `LOG_LEVEL` | `info` | all | pino level |
| `DATABASE_URL` | — (required) | auth, monitor, incident, notification, analytics | `postgres://<svc>_svc:<pw>@postgres:5432/<svc>_db` |
| `DB_POOL_MAX` | `10` (`5` for monitor-worker) | same | Pool size ([ADR-0018](../adr/0018-single-postgres-and-ram-budget.md)) |
| `DB_STATEMENT_TIMEOUT_MS` | `5000` | same | Also set on the role |
| `AMQP_URL` | — (required from Phase 2) | all but gateway | `amqp://ms:ms-dev@rabbitmq:5672` |
| `AMQP_HEARTBEAT_S` | `30` | same | |
| `JWKS_URL`, `JWT_ISSUER`, `JWT_AUDIENCE` | as above | all but auth | Token verification |
| `JWKS_CACHE_S` | `600` | same | |
| `OUTBOX_POLL_MS` | `500` | publishers | Relay interval |
| `OUTBOX_BATCH` | `100` | publishers | |
| `OTEL_SERVICE_NAME` | service name | all | |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://lgtm:4318` | all | |
| `OTEL_SDK_DISABLED` | `true` unless the `obs` profile is on | all | |
| `RETRY_TIERS_SCALE` | `1` | consumers | **Dev/test only.** Multiplies all wait-queue delays (`0.1` → 21 min schedule runs in ~2 min). Because queue TTLs are immutable, a scaled tier is a differently named queue (`…retry.1m@0.1`) |
| `CONSUMER_ACK_DELAY_MS` | `0` | consumers | **Lab only.** Waits this long between commit and broker-ack so a `docker kill` can land in the gap ([break-it lab](../roadmap/break-it-lab.md)). Refused at startup if `NODE_ENV=production` |

## Per service

### gateway

| Variable | Default | Meaning |
|----------|---------|---------|
| `UPSTREAM_AUTH` … `UPSTREAM_ANALYTICS`, `UPSTREAM_REALTIME`, `UPSTREAM_WEB` | `http://<name>:3000`, web `http://web:80` | Routing table targets |
| `UPSTREAM_TIMEOUT_MS` | `10000` | |
| `REDIS_URL` | `redis://redis:6379` | |
| `RATE_LIMIT_IP_PER_MIN`, `RATE_LIMIT_USER_PER_MIN` | `600`, `300` | |
| `RATE_LIMIT_FAIL_OPEN` | `true` | |
| `APP_ORIGIN` | `http://localhost:8080` | For the Origin check |

### auth

| Variable | Default | Meaning |
|----------|---------|---------|
| `ACCESS_TOKEN_TTL_S` | `900` | |
| `REFRESH_TOKEN_TTL_S` | `604800` | |
| `REFRESH_REUSE_GRACE_S` | `10` | Concurrent-tab grace |
| `COOKIE_SECURE` | `false` | `true` behind HTTPS |
| `ARGON2_MEMORY_KIB`, `ARGON2_ITERATIONS` | `19456`, `2` | |

### monitor (`MONITOR_ROLE=api` or `worker`)

| Variable | Default | Meaning |
|----------|---------|---------|
| `MONITOR_ROLE` | — (required) | Which modules boot |
| `WORKER_ID` | hostname | Lease owner |
| `CHECK_CONCURRENCY` | `50` | In-flight checks per worker |
| `CHECK_PER_HOST_CONCURRENCY` | `5` | |
| `LEASE_SECONDS` | `60` | Must exceed the max timeout (30 s) |
| `CLAIM_INTERVAL_MS` | `1000` | |
| `RETRY_INTERVAL_S` | `10` | Interval while a change is suspected |
| `MIN_INTERVAL_S` | `30` | |
| `ALLOW_FAST_INTERVALS` | `false` | Lowers the minimum to 10 s (lab, e2e) |
| `SSRF_ALLOW_HOSTS` | empty; `.env`: `flaky-target,dev-stubs` | [ADR-0026](../adr/0026-pinger-safety.md) |
| `EVENTLOOP_DELAY_NOT_READY_MS` | `200` | |
| `INCIDENT_SIGNAL_URL` | `http://incident:3000/internal/v1/monitor-signals` | **Phase 1 only** |

### incident

| Variable | Default | Meaning |
|----------|---------|---------|
| `ESCALATION_POLL_MS` | `5000` | |
| `DEFAULT_ESCALATION_DELAY_S` | `300` | For the default policy |

### notification

| Variable | Default | Meaning |
|----------|---------|---------|
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | `mailpit`, `1025`, empty, empty, `alerts@microsentinel.local` | |
| `SMS_GATEWAY_URL` | `http://dev-stubs:3000/sms/messages` | |
| `SEND_TIMEOUT_MS` | `5000` | |
| `DELIVERY_JOBS_PREFETCH` | `20` | Bulkhead size |
| `BREAKER_FAILURES`, `BREAKER_RESET_MS` | `5`, `30000` | |
| `SSRF_ALLOW_HOSTS` | as monitor | For webhook URLs |

### analytics

| Variable | Default | Meaning |
|----------|---------|---------|
| `CHECKS_STREAM_PREFETCH`, `LIFECYCLE_STREAM_PREFETCH` | `500`, `100` | |
| `BATCH_MAX_WAIT_MS` | `250` | |
| `ROLLUP_INTERVAL_MS` | `60000` | |
| `RAW_RETENTION_DAYS` | `30` | Must be < the checks stream's `x-max-age` (35 d) |

### realtime

| Variable | Default | Meaning |
|----------|---------|---------|
| `HEARTBEAT_S` | `15` | |
| `MAX_CONNECTIONS`, `MAX_CONNECTIONS_PER_USER` | `500`, `5` | |
| `SLOW_CLIENT_BUFFER_BYTES` | `65536` | |

### web

Build-time: none. The app uses relative `/api` paths ([web app](../frontend/web-app.md#build-and-serve)).

## Ports (dev override only)

`compose.dev.yaml` publishes each service's port 3000 to the host for direct debugging:

| Service | Host port |
|---------|-----------|
| auth | 3001 |
| monitor | 3002 |
| incident | 3003 |
| notification | 3004 |
| realtime | 3005 |
| analytics | 3006 |
| postgres | 5432 |
| rabbitmq (AMQP) | 5672 |
| Node inspector | 9229 + n (one per service, `--inspect=0.0.0.0:9229`) |

`monitor-worker` replicas publish no port (several replicas cannot share one host port).

## Files

| File | Purpose | Committed |
|------|---------|-----------|
| `.env.example` | Every variable with its local value and a comment | Yes |
| `.env` | Copy of the example, edited locally | No (git-ignored) |
| `compose.yaml` | Per-service `environment:` for values that differ by service (`DATABASE_URL`, `MONITOR_ROLE`, `OTEL_SERVICE_NAME`) | Yes |
| `infra/rabbitmq/definitions.json` | Broker user, vhost, shared exchanges, archive streams | Yes (dev credentials) |
| `infra/postgres/init/01-databases.sql` | Roles, databases, `statement_timeout` per role | Yes (dev passwords from env via `psql` variables) |

## Feature flags

None, deliberately. Phase-specific behaviour (for example the Phase 1 synchronous signal) is removed in code when the phase ends rather than kept behind a flag. Flags that outlive their purpose are a known source of complexity; this project does not need them.
