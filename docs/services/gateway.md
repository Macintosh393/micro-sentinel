# gateway

**Purpose:** the single public entry point: route each request to the right service, reject unauthenticated traffic early, and limit request rates. **Bounded context:** none. It owns no domain model.

Decision record: [ADR-0020](../adr/0020-nestjs-api-gateway.md). Concept: [API gateway and BFF](../learning/concepts/api-gateway-and-bff.md).

## Responsibilities

- Route by path prefix to an upstream service; strip the `/api` prefix.
- Verify the access token on protected routes ([ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md)) and forward the `Authorization` header unchanged.
- Rate-limit per IP and per user, with counters in Redis.
- Generate or accept `X-Request-Id`, forward it, return it.
- Enforce the upstream timeout and translate upstream failures into problem+json.
- Origin check for the two cookie-authenticated endpoints ([ADR-0016](../adr/0016-browser-session.md)).
- Add security response headers (CSP and friends) ([security](../cross-cutting/security.md)).
- Serve the web app by proxying to `web`.

## Explicitly not responsible for

- **Authorization by role** or any tenancy rule. Services do that.
- **Business logic, response aggregation or transformation.** It is not a BFF.
- **Retrying upstream calls.** A retried `POST` could act twice.
- **Caching** API responses.
- **Holding SSE connections' state.** It pipes the stream through; `realtime` owns the connections.
- **TLS.** Plain HTTP locally; a real deployment terminates TLS in front.

## Owned data

Redis keys for rate-limit counters only (`throttle:<scope>:<key>`), each with a TTL equal to its window. Losing Redis loses nothing of value.

## Routing table

| Public path | Upstream | Auth at gateway |
|-------------|----------|-----------------|
| `POST /api/v1/auth/register`, `/login`, `/refresh`, `/logout`, `/token` | `http://auth:3000/v1/auth/…` | Public |
| `GET /.well-known/jwks.json` | `http://auth:3000/.well-known/jwks.json` | Public |
| `/api/v1/users/**`, `/api/v1/orgs/**`, `/api/v1/api-keys/**` | auth | JWT |
| `/api/v1/monitors/**` | `http://monitor:3000` | JWT |
| `/api/v1/incidents/**`, `/api/v1/schedules/**`, `/api/v1/escalation-policy` | `http://incident:3000` | JWT |
| `/api/v1/contact-methods/**`, `/api/v1/channels/**`, `/api/v1/deliveries/**` | `http://notification:3000` | JWT |
| `/api/v1/analytics/**` | `http://analytics:3000` | JWT |
| `/api/v1/realtime/**` | `http://realtime:3000` (no timeout, no buffering) | JWT |
| `/health/live`, `/health/ready` | answered by the gateway itself | Public |
| Any other `/api/**` | — | `404` problem+json |
| Everything else | `http://web:80` (Vite dev server on 5173 in dev) | Public |

`/internal/**` paths of any service are **never** routed. Contracts: [OpenAPI specs](../contracts/openapi/).

## Sync APIs

**Exposed:** the routing table above; it has no API of its own beyond health.
**Consumed:** every upstream (HTTP, 10 s timeout); `GET http://auth:3000/.well-known/jwks.json` (cached 10 min); Redis.

## Events

None published, none consumed. The gateway has no broker connection.

## Internal components

| Component | Job |
|-----------|-----|
| `RequestIdMiddleware` | Request ID in, out and into the log context |
| `IpRateLimitMiddleware` | 600/min per IP; stricter buckets for login (10/min) and register (5/hour) |
| `JwtMiddleware` | Calls the shared `JwtVerifier`; `401` on failure; skips public routes |
| `UserRateLimitMiddleware` | 300/min per `sub` |
| `OriginCheckMiddleware` | For `/api/v1/auth/refresh` and `/logout` |
| `ProxyMiddleware` | One `http-proxy-middleware` instance per upstream; error handler maps `ECONNREFUSED` → `502`, timeout → `504` |
| `SecurityHeadersMiddleware` | CSP, `X-Content-Type-Options`, `Referrer-Policy` |
| `HealthController` | Liveness; readiness = JWKS loaded |

Order matters and is fixed: request ID → IP limit → JWT → user limit → origin check → proxy.

## Scaling model

Stateless. Any number of replicas behind a load balancer; counters are shared through Redis so limits hold across replicas. Locally: one.

## Failure modes

| Dependency down | Effect on the gateway | What the user sees |
|-----------------|-----------------------|--------------------|
| One upstream service | Requests to its paths fail fast | `502` (refused) or `504` (timeout) for that feature only; the rest of the app works |
| auth | Already-loaded JWKS keeps verifying tokens | Logged-in users continue for up to 15 min per token; login and refresh fail |
| Redis | Limiter **fails open**: requests pass, error counted in `gateway.ratelimit.errors` | Nothing; protection is temporarily off ([resilience](../cross-cutting/resilience.md)) |
| realtime | SSE requests get `502` | UI shows "live updates paused" and falls back to polling |
| web | `502` for page loads | App cannot load; API still works |

**When the gateway is down:** nothing is reachable from outside. Checks, incidents, pages and analytics continue, because none of them pass through it. It is the single point of failure for *users*, not for the *system*.

## Observability

| Kind | Items |
|------|-------|
| Metrics | `http.server.request.duration` by route group and status; `gateway.upstream.errors` by upstream and kind; `gateway.ratelimit.rejected` by bucket; `gateway.ratelimit.errors`; open SSE proxy connections |
| Spans | One server span per request (root of every user-initiated trace) and one client span for the upstream call |
| Log fields | `request_id`, `trace_id`, `route`, `upstream`, `status`, `duration_ms`, `user_id`, `org_id`, `client_ip`. Never the `Authorization` header or cookies |

## Patterns applied

[API gateway and BFF](../learning/concepts/api-gateway-and-bff.md) · [JWT and JWKS](../learning/concepts/jwt-and-jwks.md) · [service discovery](../learning/concepts/service-discovery.md) (Compose DNS) · [backpressure](../learning/concepts/backpressure.md) (rate limiting) · [health checks](../learning/concepts/health-checks.md)

## Framework and Nest structure

**NestJS 12**, Express adapter. Close to the line where plain Express would do; Nest is kept for the shared modules ([ADR-0004](../adr/0004-nestjs-everywhere-express-for-fixtures.md)).

| Nest element | Use |
|--------------|-----|
| Modules | `AppModule`, `ConfigModule`, `ProxyModule`, `RateLimitModule`, `HealthModule`, shared `AuthGuardModule` and `ObservabilityModule` |
| Middleware | All request handling (listed above). Applied in `AppModule.configure()` in explicit order |
| Guards / pipes / interceptors | Not used on proxied routes: raw middleware bypasses them. `JwtAuthGuard` is used only indirectly, through its `JwtVerifier` provider |
| Exception filter | `ProblemDetailsFilter` for errors raised by the gateway itself |
| Bootstrap | `bodyParser: false`, so request bodies stream to the upstream untouched |

## Phase introduced

Phase 0 (routing to one service and the web app) → Phase 1 (JWT, rate limiting, all routes) → Phase 2 (SSE pass-through).

## Related

Flows: [register-monitor](../flows/register-monitor.md). Frontend: [auth in the browser](../frontend/auth-in-the-browser.md).
