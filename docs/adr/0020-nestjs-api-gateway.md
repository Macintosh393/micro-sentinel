# ADR-0020: Build the API gateway as a small NestJS reverse proxy

- Status: Accepted
- Date: 2026-11-15
- Phase: 0 (routing), 1 (JWT, rate limiting)

## Context

Browsers and API clients need one address. Behind it are seven services. An **API gateway** is that single entry point: it routes by path, and usually takes over concerns every service would otherwise repeat, such as rejecting unauthenticated requests, rate limiting and request logging ("offloading").

The brief asks to weigh Traefik, Kong (OSS), Envoy and a NestJS/Express gateway.

## Decision drivers

- Required features: path-based routing, RS256 JWT verification against a JWKS, Redis-backed rate limiting per IP and per user, SSE pass-through, request IDs, uniform error bodies, health checks.
- Self-hosted, small memory footprint.
- Learning value for a junior NestJS + Node role.
- The gateway must stay thin: no business logic, no data aggregation.

## Options considered

| | Traefik | Kong (OSS) | Envoy | NestJS gateway |
|-|---------|------------|-------|----------------|
| Nature | Go reverse proxy; configuration from Docker labels | nginx/OpenResty + Lua plugins; declarative config | C++ proxy; the data plane of most service meshes | Our own Node process |
| Routing | Excellent; auto-discovers containers | Excellent | Excellent; verbose config | A route table we write |
| JWT + JWKS | Not in the open-source core: needs a plugin or a `forwardAuth` call to another service | JWT plugin (keys configured per consumer); OIDC/JWKS conveniences are in the enterprise edition | Built-in `jwt_authn` filter with remote JWKS | The shared guard used by every service |
| Rate limiting | Built in; per-instance memory by default | Plugin with Redis | Local filter, or global via a separate rate-limit service | `@nestjs/throttler` with Redis storage |
| Memory | ~50–100 MB | ~200–300 MB (DB-less) | ~50–100 MB | ~150–200 MB |
| Maturity as a proxy | Production-grade | Production-grade | Production-grade | Whatever we manage to write |
| What you learn | Label-driven infra config | Plugin configuration | Envoy's config model (valuable, steep) | HTTP proxying, streaming, middleware order, guards, rate-limit algorithms, timeouts: Node and Nest fundamentals |
| Risk | JWT story needs extra parts | Distribution of prebuilt OSS images changed in 2025; verify availability before adopting | Config complexity dominates Phase 0 | We own correctness: header handling, streaming, timeouts |

Where an alternative would be the better call: **Traefik** for any real deployment of this system (mature, tiny, and TLS handled), combined with JWT verification staying in the services; **Envoy** if a service mesh were on the roadmap; **Kong** where a plugin ecosystem and an admin API matter.

## Decision

A **NestJS 12 gateway** (`apps/gateway`), using `http-proxy-middleware` for proxying.

**Responsibilities**

| Concern | How |
|---------|-----|
| Routing | A static table: path prefix → upstream. `/api` is stripped; `/api/v1/monitors` becomes `http://monitor:3000/v1/monitors`. Full table in [gateway](../services/gateway.md) |
| Authentication offloading | `JwtAuthGuard` (same guard as services) on everything except the public list: `POST /api/v1/auth/register`, `/login`, `/refresh`, `/logout`, `/token`, `GET /.well-known/jwks.json`, static files, health. Invalid or missing token → `401` problem+json. The `Authorization` header is forwarded unchanged ([ADR-0015](0015-jwt-rs256-jwks-validate-everywhere.md)) |
| Rate limiting | `@nestjs/throttler` with Redis storage. Limits: 600 requests/min per IP; 300/min per `sub`; login 10/min per IP; register 5/hour per IP. Exceeded → `429` with `Retry-After`. If Redis is down the limiter **fails open** (requests pass, an error is logged and counted): availability over strictness for a local system; stated in [resilience](../cross-cutting/resilience.md) |
| Timeouts | 10 s upstream timeout → `504`; connection refused → `502`. Disabled for `/api/v1/realtime/stream` |
| Request ID | Accepts or generates `X-Request-Id`; forwards it; returns it |
| Headers | Sets `X-Forwarded-For/Proto/Host`; strips hop-by-hop headers; adds `Content-Security-Policy` and other security headers to responses |
| CSRF origin check | For `/api/v1/auth/refresh` and `/logout` ([ADR-0016](0016-browser-session.md)) |
| Static app | Proxies everything not under `/api` or `/.well-known` to `web` |
| Observability | Access log line and a server span per request; propagates `traceparent` |

**Explicitly not in the gateway:** request/response transformation, aggregation of several services into one response, authorization by role (services do that), caching, retries of upstream calls (a retried `POST` is not safe), any knowledge of domain objects.

**Implementation constraints** worth knowing before starting:

- The proxy middleware is mounted **before** any body parser. If Nest parses the JSON body first, the stream is consumed and the proxy forwards an empty body.
- Guards do not run for raw middleware, so the JWT and throttle checks are applied as middleware functions that reuse the guard's verification service, in a fixed order: request ID → rate limit (IP) → JWT → rate limit (user) → proxy.
- SSE responses must not be buffered or compressed.

## Consequences

- Positive: auth, rate limiting, logging and tracing reuse the shared Nest packages; one language everywhere.
- Positive: the author can explain what a gateway does from having written one.
- Negative: we own proxy edge cases that Traefik and Envoy solved years ago (large uploads, slow clients, header normalisation, HTTP/2). The scope is kept small to limit this.
- Negative: more memory than a Go or C++ proxy.
- Negative: verifying JWTs in the gateway *and* in services duplicates CPU work; chosen deliberately for defence in depth.
- Harder later: replacing it with Traefik is easy because services never depended on gateway-injected identity headers.

## Revisit when

- TLS termination, HTTP/2 or WebSocket proxying is needed: put Traefik in front or replace the gateway.
- The gateway accumulates business logic: stop; move it to a service or a BFF.
- Phase 5 (Kubernetes): use an Ingress controller for routing and keep only rate limiting, if anything, in Node.

## Learning choice?

**Yes.** A pragmatic team would run Traefik (or the cloud's load balancer) for routing and TLS, keep JWT verification in the services with the shared guard, and add rate limiting in the proxy or with a small middleware.

## How to explain it in an interview

> "The gateway is the single entry point: it routes by path prefix, rejects requests without a valid JWT, rate-limits with counters in Redis, and adds request IDs. I wrote it in NestJS with http-proxy-middleware because I wanted to understand what a gateway actually does: streaming bodies, timeouts, middleware order. In production I'd use Traefik or Envoy, because a hand-written proxy has edge cases they've already solved. I kept it deliberately thin: no business logic and no aggregation, and services still verify the token themselves, so swapping the gateway out is easy."
