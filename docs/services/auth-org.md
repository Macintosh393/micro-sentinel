# auth (Auth & Org)

**Purpose:** know who users are, which organization and role they have, and issue tokens that prove it. **Bounded context:** Identity & Access ([domain model](../02-domain-model.md)).

Decision records: [ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md) (tokens), [ADR-0016](../adr/0016-browser-session.md) (browser session), [ADR-0014](../adr/0014-on-call-in-incident-and-local-replicas.md) (what auth does *not* own).

## Responsibilities

- Registration (creates user, organization and owner membership in one transaction).
- Login, refresh-token rotation with reuse detection, logout.
- Issue RS256 access tokens; serve the JWKS; hold and rotate signing keys.
- Organization members: add, change role, remove.
- API keys: create, list, revoke, exchange for a token.
- Publish snapshot events so other services can keep local replicas.

## Explicitly not responsible for

- **On-call schedules and escalation policies** (incident).
- **Contact methods and notification preferences** (notification). Auth knows the account email only.
- **Verifying tokens on behalf of other services.** They verify locally with the JWKS; there is no introspection endpoint.
- **Authorization decisions inside other services.** Auth states the role; each service decides what a role may do.
- **Sending email** (no invitations, no password reset in v1; see [open questions](../open-questions.md)).

## Owned data

`auth_db`: `organizations`, `users`, `memberships`, `refresh_tokens`, `api_keys`, `signing_keys`, `outbox`. Schema: [auth-schema](../data/auth-schema.md).

## Sync APIs

**Exposed** ([auth.yaml](../contracts/openapi/auth.yaml)):

| Group | Endpoints |
|-------|-----------|
| Session | `POST /v1/auth/register`, `/login`, `/refresh`, `/logout`, `/token` |
| Users | `GET`/`PATCH /v1/users/me` |
| Organization | `GET /v1/orgs/current`; `GET`/`POST /v1/orgs/current/members`; `PATCH`/`DELETE /v1/orgs/current/members/{userId}` |
| API keys | `GET`/`POST /v1/api-keys`; `DELETE /v1/api-keys/{id}` |
| Keys | `GET /.well-known/jwks.json` |

**Consumed:** none. Auth calls no other service.

## Events

**Published** (via outbox; [catalog](../contracts/event-catalog.md)):

| Event | When |
|-------|------|
| `auth.org.created.v1` | Registration |
| `auth.user.changed.v1` | User created, profile edited, deactivated |
| `auth.membership.changed.v1` | Member added, role changed, removed |

**Consumed:** none.

A one-off command, `pnpm --filter @microsentinel/auth snapshots:republish`, writes a fresh snapshot event for every user and membership into the outbox. It is used once, in Phase 2, to seed replicas with data that existed before events did.

## Internal components

| Component | Job |
|-----------|-----|
| `SessionService` | Register, login, refresh, logout; sets and clears the `ms_refresh` cookie |
| `PasswordHasher` | Argon2id hash and verify (memory 19 MiB, 2 iterations, parallelism 1) |
| `TokenIssuer` | Builds and signs JWTs with the current signing key (`jose`) |
| `SigningKeyService` | Creates the first key at startup if none exists; serves the JWKS; rotation command |
| `RefreshTokenService` | Rotation, 10 s grace for concurrent tabs, family revocation |
| `MembersService`, `ApiKeysService` | CRUD with role checks; every change adds an outbox row |
| `OutboxRelay` | From `@microsentinel/messaging` |
| `Janitor` | Deletes expired refresh tokens, retired keys, old outbox rows |

## Scaling model

Stateless processes over one database; replicas share signing keys through `signing_keys`. Login is CPU-bound by Argon2 (~50–100 ms each): the limit on logins per second is deliberate, and Argon2 runs in libuv's thread pool so it does not block the event loop. Locally: one replica.

## Failure modes

| Dependency down | Effect |
|-----------------|--------|
| PostgreSQL | Everything fails with `503`. Readiness goes false |
| RabbitMQ | APIs keep working; events accumulate in the outbox and are published when the broker returns. Replicas elsewhere are stale meanwhile |

**When auth is down:**

| Still works | Stops |
|-------------|-------|
| Every request with an unexpired access token (services verify with cached keys) | Login, registration, refresh: sessions end within 15 minutes |
| Checks, incident opening, paging, escalation, deliveries | Member and API-key management |
| Token verification in services that already loaded the JWKS | A service that **restarts** cannot load the JWKS and stays not-ready |

This is the [break-it lab](../roadmap/break-it-lab.md) experiment "Auth down while incidents are being opened".

## Observability

| Kind | Items |
|------|-------|
| Metrics | `auth.logins` by outcome; `auth.refresh` by outcome (`ok`, `expired`, `reused`); `auth.password_hash.duration`; `outbox.pending`, `outbox.oldest_age_seconds` |
| Spans | HTTP server spans; `argon2.verify`; DB spans; `ms.events publish` from the relay |
| Log fields | `user_id`, `org_id`, `family_id`, `outcome`. **Never** passwords, tokens, hashes or full API keys. A `REFRESH_TOKEN_REUSED` event is logged at `warn` |

## Patterns applied

[JWT and JWKS](../learning/concepts/jwt-and-jwks.md) · [browser authentication](../learning/concepts/browser-authentication.md) · [transactional outbox](../learning/concepts/transactional-outbox.md) · [event-driven architecture](../learning/concepts/event-driven-architecture.md) (event-carried state transfer) · [database-per-service](../learning/concepts/database-per-service.md)

## Framework and Nest structure

**NestJS 12.** Ordinary request/response service: Nest's defaults fit completely.

| Nest element | Use |
|--------------|-----|
| Modules | `SessionModule`, `UsersModule`, `OrganizationModule`, `ApiKeysModule`, `KeysModule`, `HealthModule`; shared `MessagingModule` (publish only), `AuthGuardModule`, `ObservabilityModule`; `TypeOrmModule.forFeature` per module |
| Guards | `JwtAuthGuard` globally with a `@Public()` decorator for the session and JWKS routes; `RolesGuard` with `@Roles('ADMIN')` on member and API-key writes |
| Pipes | Global `ValidationPipe` (`whitelist`, `forbidNonWhitelisted`, `transform`); `ParseUUIDPipe` on path parameters |
| Interceptors | `ClassSerializerInterceptor` so `password_hash` can never be serialised (`@Exclude()`); logging interceptor |
| Exception filter | `ProblemDetailsFilter` (shared): maps `HttpException` and unknown errors to problem+json with a `code` |
| Custom decorators | `@CurrentUser()`, `@Public()`, `@Roles()` |

Auth verifies tokens on its own protected routes with the same guard as every other service, reading its own JWKS from the database instead of over HTTP.

## Phase introduced

Phase 1 (register, login, refresh, JWKS) → Phase 2 (events) → Phase 3 (members, API keys).

## Related

Flows: [register-monitor](../flows/register-monitor.md) (token use), [service-down-recovery](../flows/service-down-recovery.md). Frontend: [auth in the browser](../frontend/auth-in-the-browser.md). Security: [security](../cross-cutting/security.md).
