# ADR-0015: Sign access tokens with RS256 and verify them at the gateway and in every service

- Status: Accepted
- Date: 2026-11-15
- Phase: 1

## Context

Users log in at auth and receive an **access token**: a JWT (JSON Web Token), a signed JSON document stating who the user is, which organization they act in and with what role. Three decisions:

1. **Signing algorithm.** Symmetric (HS256: one shared secret both signs and verifies) or asymmetric (RS256/ES256/EdDSA: a private key signs, a public key verifies).
2. **Where tokens are verified.** Only at the gateway, with services trusting what the gateway forwards, or also in each service.
3. **How services authenticate to each other.**

How the browser stores tokens is a separate decision: [ADR-0016](0016-browser-session.md).

## Decision drivers

- A service that can verify tokens must not thereby be able to *create* them.
- Auth being down must not stop other services from verifying tokens.
- A request that reaches a service without passing the gateway (misconfiguration, a compromised container on the internal network) must not be trusted.
- Understandable end to end; a common interview topic.

## Options considered

### Algorithm and key distribution

| Option | Pros | Cons |
|--------|------|------|
| A. HS256, one secret shared by all services | Simplest; fastest | Every verifier can also mint tokens for any user. One leaked service config compromises all identities. Rotation means redeploying every service at once |
| B. **RS256 with a JWKS endpoint** (chosen) | Only auth holds the private key. Verifiers fetch public keys from `/.well-known/jwks.json` and cache them. Rotation: publish the new key, start signing with it, retire the old one after tokens expire. Universally supported | Larger tokens (~700–800 bytes); slower signing (irrelevant here) |
| C. EdDSA (Ed25519) or ES256 with JWKS | Smaller, faster signatures | Slightly less universal in older tooling; nothing here benefits from the size difference |
| D. Opaque tokens + introspection (every service asks auth "is this valid?") | Instant revocation | A synchronous call to auth on every request: the temporal coupling this design avoids |

### Where to verify

| Option | Pros | Cons |
|--------|------|------|
| E. Gateway only; it forwards `X-User-Id` / `X-Org-Id` headers that services trust | Services contain no auth code; one place to change | The internal network becomes the security boundary. Anything that can reach a service can impersonate anyone by setting a header. A routing mistake that exposes a service is a full bypass |
| F. **Gateway and every service** verify the same token (chosen) | **Defence in depth**: a request is trusted because of a signature, not because of where it came from (the "zero trust" idea). Services can be tested and called directly with a real token | Verification code in every service (one shared guard); a few hundred microseconds of CPU per request per hop |
| G. Services only, gateway does not verify | Simplest gateway | Unauthenticated traffic reaches services before being rejected; rate limiting per user is impossible at the edge |

### Service-to-service

| Option | Pros | Cons |
|--------|------|------|
| H. Nothing (trust the network) | Zero work | See option E |
| I. Service tokens (client credentials): each service gets its own identity from auth | Proper identity for machines | Auth must issue and rotate them; more to build |
| J. Mutual TLS, usually through a service mesh | Strong identity at the transport layer | Certificates and a mesh: heavy for Compose |

## Decision

**B + F**, and for service-to-service: **there is nothing to authenticate.**

**Token**

| Claim | Value |
|-------|-------|
| `iss` | `microsentinel-auth` |
| `aud` | `microsentinel-api` |
| `sub` | user ID, or `apikey:<id>` for API-key tokens |
| `org` | organization ID |
| `role` | `OWNER` \| `ADMIN` \| `MEMBER` |
| `iat`, `exp` | lifetime 15 minutes |
| `jti` | unique token ID (for logs) |
| header `kid` | ID of the signing key; header `alg` = `RS256` |

**Verification, identical everywhere**, in the shared Nest guard `JwtAuthGuard` from `@microsentinel/auth-guard` (built on `jose`):

1. Read `Authorization: Bearer <token>`.
2. Verify the signature with the key matching `kid` from the cached JWKS. Accepted algorithms are pinned to `["RS256"]`; the token's `alg` header is never trusted to choose.
3. Check `iss`, `aud`, `exp` (30 s clock tolerance).
4. Attach `{ userId, orgId, role }` to the request. Every query filters by `orgId`.

A second guard, `RolesGuard`, enforces `@Roles('ADMIN')`-style metadata.

**JWKS caching.** Each service fetches `http://auth:3000/.well-known/jwks.json` at startup (readiness waits for it) and caches it for 10 minutes. An unknown `kid` triggers one refetch, rate-limited to once per 30 s. If auth is unreachable, cached keys keep working. So auth is needed to **log in**, not to **use** the system.

**Gateway.** Verifies the token (rejects with `401` early, applies per-user rate limits) and forwards the `Authorization` header **unchanged**. It adds no identity headers.

**Service-to-service.** After [ADR-0014](0014-on-call-in-incident-and-local-replicas.md) no service calls another's API. The only internal HTTP calls are JWKS fetches, which are public by design. Events are trusted because only services can reach the broker; each service has broker credentials. Internal operational endpoints (such as the analytics rebuild trigger) are not routed by the gateway and require an `ADMIN` token like any other.

**Revocation.** A JWT cannot be revoked before `exp` without a lookup. With a 15-minute lifetime the exposure is bounded: logout and "remove member" revoke the refresh token immediately; the access token dies within 15 minutes. Accepted; stated in [security](../cross-cutting/security.md).

**Keys in development.** Auth generates an RSA-2048 key pair on first start and stores it in `auth_db` (`signing_keys` table), so tokens survive restarts and every auth replica uses the same key. No key material is committed to the repository.

## Consequences

- Positive: compromise of any non-auth service does not allow forging tokens.
- Positive: services are individually secure and individually testable.
- Positive: auth is off the request path.
- Negative: 15 minutes of validity after logout or role change.
- Negative: every service depends on the shared guard package (a technical library, permitted by [ADR-0022](0022-contracts-package-and-shared-libs.md)).
- Negative: broker messages are trusted by network position and broker credentials only; there is no per-message signature.
- Harder later: instant revocation needs a denylist checked on every request (a shared cache) or shorter tokens.

## Revisit when

- A requirement for immediate revocation appears: add a `jti` denylist in Redis checked by the guard.
- Synchronous service-to-service calls are introduced: add option I (service tokens) at minimum.
- Deployment moves to Kubernetes with a mesh: add mTLS underneath.

## Learning choice?

Partly. RS256 + JWKS + verifying in each service is sound practice. **Building the identity provider at all is the learning choice**: a pragmatic team would run Keycloak or use a hosted provider and write only the verification side.

## How to explain it in an interview

> "Auth signs tokens with a private RSA key and publishes the public keys on a JWKS endpoint. Every service verifies the JWT itself with cached public keys, so auth isn't on the request path and a service that can verify tokens can't mint them, which is the problem with a shared HS256 secret. The gateway also verifies, to reject junk early and rate-limit per user, but services don't trust headers from the gateway: they check the signature. That's defence in depth. The cost is that a token stays valid until it expires, so I keep access tokens at fifteen minutes and revoke the refresh token on logout."
