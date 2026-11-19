# Security

Authentication, authorization, tenant isolation, service-to-service trust, secrets in local development, and the SSRF risk of checking user-supplied URLs. This is a local learning system; each section says what a production deployment would add. Decisions: [ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md), [ADR-0016](../adr/0016-browser-session.md), [ADR-0026](../adr/0026-pinger-safety.md).

## Threat model in one table

| Asset | Threat | Main control |
|-------|--------|--------------|
| User accounts | Credential stuffing, password guessing | Argon2id, login rate limit, uniform error message |
| Sessions | Token theft via XSS; refresh-token theft | Access token in memory only; httpOnly refresh cookie; rotation with reuse detection; CSP |
| Tenant data | One organization reading another's data | `org` from the verified token on every query; no client-supplied org ID |
| Internal network | SSRF through monitor URLs and webhook URLs | Resolve-validate-connect guard; no redirects; allow-list only for lab hosts |
| Internal services | Requests that bypass the gateway | Every service verifies the JWT itself |
| Token signing | Forged tokens | RS256; private key only in auth |
| Webhook receivers | Spoofed MicroSentinel calls | HMAC signature header |
| Availability | Request floods; hostile targets | Rate limits; timeouts, caps, never reading bodies |

## Authentication

| Client | Mechanism |
|--------|-----------|
| Browser | Login → access JWT (memory) + rotating refresh cookie ([auth in the browser](../frontend/auth-in-the-browser.md)) |
| Script | API key → `POST /v1/auth/token` → access JWT (15 min) |
| Service → service | None needed; no synchronous inter-service calls ([ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md)) |
| Service → broker / database | Per-service credentials from environment |

### Passwords

- **Argon2id**, memory 19 MiB, iterations 2, parallelism 1 (the OWASP Password Storage Cheat Sheet's minimum configuration for Argon2id). Stored as a PHC string, which records the parameters, so they can be raised later and old hashes rehashed on next login.
- Minimum length 12, maximum 128 (very long inputs are a denial-of-service vector for slow hashes). No composition rules.
- Login always runs a hash verification, even for unknown emails (against a fixed dummy hash), so response time does not reveal which emails exist.

### Tokens

- JWT verification pins `algorithms: ['RS256']`, `issuer`, `audience`; 30 s clock tolerance. The `alg` header from the token is never used to choose the algorithm (prevents the `alg: none` and RS/HS confusion attacks).
- Access token 15 min; refresh 7 days, rotated, hashed (SHA-256) at rest, family revocation on reuse.
- API keys: shown once; stored as SHA-256; prefix for lookup; revocable; `last_used_at` recorded.
- **Revocation gap:** an access token stays valid until `exp`. Removing a member revokes their refresh tokens immediately; their access token dies within 15 min. A `jti` denylist would close the gap ([ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md) "Revisit when").

## Authorization

| Rule | Where enforced |
|------|----------------|
| Request must carry a valid token | Gateway (early reject) **and** every service (`JwtAuthGuard`) |
| Role required for an operation | The owning service (`RolesGuard` + `@Roles()`) |
| Resource belongs to the caller's organization | The owning service, in every query: `WHERE org_id = $tokenOrg` |
| User may only edit own contact methods | notification, `WHERE user_id = $tokenSub` |

Role matrix:

| Action | OWNER | ADMIN | MEMBER | API key (ADMIN) | API key (MEMBER) |
|--------|-------|-------|--------|-----------------|------------------|
| Read anything in the organization | ✓ | ✓ | ✓ | ✓ | ✓ |
| Acknowledge / resolve incidents | ✓ | ✓ | ✓ | ✓ | ✓ |
| Create / edit / delete monitors | ✓ | ✓ | — | ✓ | — |
| Schedules, escalation policy, channels | ✓ | ✓ | — | ✓ | — |
| Members, API keys | ✓ | ✓ | — | — | — |
| Change or remove the owner | — | — | — | — | — |
| Own profile and contact methods | ✓ | ✓ | ✓ | n/a | n/a |
| Internal endpoints (analytics rebuild) | ✓ | ✓ | — | — | — |

**Tenant isolation details:**

- Cross-tenant access returns **`404`**, not `403`, so the existence of another organization's IDs is not confirmed.
- UUIDv7 IDs are not guessable in practice, but isolation never relies on that.
- An integration test per service creates two organizations and asserts every endpoint refuses cross-access ([testing strategy](testing-strategy.md)).
- Events carry `orgId`; consumers write it from the event, never infer it.
- PostgreSQL row-level security as a second layer is in [open questions](../open-questions.md).

## Service-to-service trust

| Channel | Trust basis | Gap |
|---------|-------------|-----|
| Gateway → service (HTTP) | The JWT, verified by the service | None for identity; traffic is plain HTTP on the internal network |
| Service → JWKS | Public data | A spoofed JWKS endpoint on the internal network could serve attacker keys. Mitigated only by network isolation locally; production: TLS + pinning the issuer URL |
| Publisher → broker | Per-service broker credentials (dev: one shared `ms` user) | Any holder of broker credentials can publish any event. Production: one RabbitMQ user per service with **topic permissions** restricting which routing keys it may publish (RabbitMQ supports per-vhost topic authorisation), and TLS |
| Service → database | Per-service role, own database only | — |

Locally all services share the broker user `ms` for simplicity; this is listed as a shortcut paid off in Phase 5 ([phases](../roadmap/phases.md)).

## SSRF: checking user-supplied URLs

The biggest product-specific risk. Full decision: [ADR-0026](../adr/0026-pinger-safety.md).

What an attacker could do without a guard, from inside our network:

- `http://postgres:5432/`, `http://rabbitmq:15672/api/overview`, `http://lgtm:3000/`: probe or talk to internal services. Even a status code and latency leak information ("port open, responds in 2 ms").
- `http://169.254.169.254/latest/meta-data/`: the cloud metadata address, which on real cloud hosts can return credentials.
- `http://127.0.0.1:3000/internal/…`: the worker's own internal endpoints.
- A public URL that **redirects** to any of the above.
- A hostname that resolves to a public IP at save time and to `127.0.0.1` at check time (**DNS rebinding**).

Controls, in order:

1. Scheme `http`/`https`; ports 80, 443, ≥ 1024; no userinfo in the URL.
2. Resolve at request time with a custom `lookup`; reject loopback, private, link-local, CGNAT, multicast, unspecified, IPv4-mapped IPv6 and documentation ranges; connect to the validated IP.
3. Redirects not followed.
4. Response body never read.
5. Same guard for webhook and chat channel URLs in notification, at save and at send.
6. `SSRF_ALLOW_HOSTS` exempts only named lab hosts (`flaky-target`, `dev-stubs`); empty by default.

Tests for the classifier include: `127.1`, `0x7f000001`, `2130706433`, `[::1]`, `[::ffff:127.0.0.1]`, `localhost.`, a name with both public and private A records, and `0.0.0.0`.

**Production addition:** run workers in a network that can reach only the internet and the broker/database through an egress proxy that enforces the same block list. Defence in depth: code-level checks can have bugs; network rules catch what they miss.

## Webhook signing

Outgoing `WEBHOOK` deliveries carry:

```
X-MS-Signature: t=1763203327,v1=5f2b…(hex HMAC-SHA256)
X-MS-Event: incident.opened.v1
Idempotency-Key: 019a7c40-3c4d-…   (delivery id)
```

`v1 = HMAC_SHA256(signing_secret, "<t>.<raw body>")`. Receivers verify with a constant-time comparison and reject timestamps older than 5 minutes (replay protection). `dev-stubs` displays the verification result. The secret is per channel, shown once, stored in clear because it must be used (production: encrypt at rest with a key from a secret manager).

## Browser security headers (set by the gateway)

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';
                         img-src 'self' data:; connect-src 'self'; object-src 'none';
                         frame-ancestors 'none'; base-uri 'self'
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Cross-Origin-Opener-Policy: same-origin
```

`style-src 'unsafe-inline'` is needed by Mantine's runtime styles; check whether the pinned Mantine version still needs it and remove it if not. No `Strict-Transport-Security` locally (plain HTTP); required with TLS.

## Secrets in local development

| Secret | Source | Committed? |
|--------|--------|-----------|
| Database passwords per role | `.env` (from `.env.example`) | Example values only, labelled `dev-only` |
| Broker user `ms` / `ms-dev` | `infra/rabbitmq/definitions.json` + `.env` | Dev value, labelled |
| JWT signing key | Generated by auth on first start, stored in `auth_db` | **Never** |
| Webhook signing secrets | Generated per channel | Never |
| Real SMTP/SMS credentials | Not used | — |

Rules: `.env` is git-ignored; `.env.example` contains only obviously fake development values; a pre-commit check (`gitleaks` or a grep for key patterns) runs in `pnpm ci:local`. Production would inject secrets from a secret manager as files or environment variables, never bake them into images ([configuration](configuration.md)).

## Input validation and output encoding

- All HTTP input validated by `ValidationPipe` with whitelisting; unknown fields rejected (`400`).
- All event input validated against `contracts` classes; invalid → DLQ ([ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)).
- All SQL parameterised, including raw queries; no string concatenation of values.
- Request body limit 100 KB at the services; the gateway streams bodies but rejects `Content-Length` > 1 MB.
- User-supplied strings rendered as text by React; URLs rendered as links only for `http(s)` schemes.

## Supply chain

Exact version pins, a lockfile, `pnpm audit` in local CI (report only), official images pinned by tag. Production would pin images by digest, scan them, and generate an SBOM.

## Known gaps (accepted for a local system)

- Plain HTTP everywhere inside and outside.
- Shared broker user.
- No audit log of admin actions beyond incident timelines.
- No account lockout or MFA; no password reset.
- Signing secrets and the JWT private key stored unencrypted in databases.
- 15-minute access-token revocation window.

Each is a fair interview question; [interview prep](../learning/interview-prep.md#weak-spots-and-honest-answers) has the answers.
