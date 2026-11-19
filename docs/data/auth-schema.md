# auth_db schema

Owner: [auth](../services/auth-org.md). Role: `auth_svc`. Nobody else connects.

Also contains `outbox` ([messaging tables](messaging-tables.md)).

## Tables

```sql
CREATE EXTENSION IF NOT EXISTS citext;   -- case-insensitive text, for email

CREATE TABLE organizations (
  id          uuid        PRIMARY KEY DEFAULT uuidv7(),
  name        text        NOT NULL CHECK (length(name) BETWEEN 1 AND 100),
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id             uuid        PRIMARY KEY DEFAULT uuidv7(),
  email          citext      NOT NULL UNIQUE,
  display_name   text        NOT NULL CHECK (length(display_name) BETWEEN 1 AND 100),
  password_hash  text        NOT NULL,                 -- Argon2id, PHC string format
  active         boolean     NOT NULL DEFAULT true,
  version        integer     NOT NULL DEFAULT 1,       -- +1 on every change; sent in auth.user.changed.v1
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
  org_id      uuid        NOT NULL REFERENCES organizations (id) ON DELETE CASCADE,
  user_id     uuid        NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  role        text        NOT NULL CHECK (role IN ('OWNER', 'ADMIN', 'MEMBER')),
  version     integer     NOT NULL DEFAULT 1,          -- sent in auth.membership.changed.v1
  removed_at  timestamptz,                             -- soft delete, so version keeps increasing
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (org_id, user_id)
);

-- Invariant: exactly one owner per organization (at most one enforced here,
-- at least one enforced by the code path that creates an organization).
CREATE UNIQUE INDEX memberships_one_owner_idx
  ON memberships (org_id) WHERE role = 'OWNER' AND removed_at IS NULL;

CREATE INDEX memberships_user_idx ON memberships (user_id) WHERE removed_at IS NULL;

CREATE TABLE refresh_tokens (
  id           uuid        PRIMARY KEY DEFAULT uuidv7(),
  family_id    uuid        NOT NULL,                   -- all rotations of one login share this
  user_id      uuid        NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  org_id       uuid        NOT NULL REFERENCES organizations (id) ON DELETE CASCADE,
  token_hash   bytea       NOT NULL UNIQUE,            -- SHA-256 of the token
  expires_at   timestamptz NOT NULL,
  rotated_at   timestamptz,                            -- set when exchanged for a successor
  replaced_by  uuid        REFERENCES refresh_tokens (id),
  revoked_at   timestamptz,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX refresh_tokens_family_idx  ON refresh_tokens (family_id);
CREATE INDEX refresh_tokens_expires_idx ON refresh_tokens (expires_at);

CREATE TABLE api_keys (
  id            uuid        PRIMARY KEY DEFAULT uuidv7(),
  org_id        uuid        NOT NULL REFERENCES organizations (id) ON DELETE CASCADE,
  name          text        NOT NULL CHECK (length(name) BETWEEN 1 AND 100),
  key_prefix    text        NOT NULL UNIQUE,           -- first 12 chars, shown in the UI, used for lookup
  key_hash      bytea       NOT NULL,                  -- SHA-256 of the full key
  role          text        NOT NULL CHECK (role IN ('ADMIN', 'MEMBER')),
  created_by    uuid        NOT NULL REFERENCES users (id),
  created_at    timestamptz NOT NULL DEFAULT now(),
  last_used_at  timestamptz,
  revoked_at    timestamptz
);

CREATE INDEX api_keys_org_idx ON api_keys (org_id) WHERE revoked_at IS NULL;

CREATE TABLE signing_keys (
  kid          text        PRIMARY KEY,                -- key ID, appears in the JWT header
  algorithm    text        NOT NULL DEFAULT 'RS256',
  public_jwk   jsonb       NOT NULL,                   -- served from /.well-known/jwks.json
  private_pem  text        NOT NULL,                   -- never leaves auth
  created_at   timestamptz NOT NULL DEFAULT now(),
  retired_at   timestamptz                             -- no longer signs; still verifies until tokens expire
);
```

## Design notes

**Why `citext` for email.** `Alice@Example.com` and `alice@example.com` are the same account. `citext` makes the `UNIQUE` constraint and every comparison case-insensitive without remembering `lower()` in each query.

**Why a soft delete on memberships but not elsewhere.** The membership snapshot event carries a `version`, and consumers ignore versions they have already seen. If the row were deleted and later recreated, the version would restart at 1 and replicas would ignore the new membership. Keeping the row with `removed_at` keeps the counter monotonic.

**Why refresh tokens and API keys use SHA-256 and passwords use Argon2id.** A password is short and guessable, so its hash must be slow and salted to resist offline guessing. A refresh token or API key is 256 random bits: guessing is infeasible, so a fast hash is enough and lets auth look the token up by its hash in one indexed query ([security](../cross-cutting/security.md)).

**API key format.** `msk_<12-char prefix>_<43-char secret>`. The prefix is stored in clear for lookup and display; the whole key is shown once at creation and never again.

**Refresh token rotation.** Exchange is one transaction:

```sql
SELECT … FROM refresh_tokens WHERE token_hash = $1 FOR UPDATE;
-- not found / expired / revoked           → 401
-- rotated_at IS NOT NULL and older than 10 s → reuse detected:
--     UPDATE refresh_tokens SET revoked_at = now() WHERE family_id = $family;  → 401
-- rotated_at within the last 10 s         → return the successor (two tabs raced)
-- otherwise: insert successor, set rotated_at and replaced_by on this row
```

The `FOR UPDATE` row lock makes two simultaneous refreshes with the same token serialise instead of both succeeding. Decision: [ADR-0016](../adr/0016-browser-session.md).

**Signing keys in the database.** Every auth replica must sign with the same key, and keys must survive restarts. Storing the private key in the service's own database is acceptable for a local system; a real deployment would use a key management service and is noted in [security](../cross-cutting/security.md).

## Indexes and the queries they serve

| Index | Query |
|-------|-------|
| `users.email` (unique) | Login: find user by email |
| `memberships` primary key | Role check for (org, user) |
| `memberships_user_idx` | "Which organization does this user belong to?" at login |
| `memberships_one_owner_idx` | Enforces the one-owner invariant |
| `refresh_tokens.token_hash` (unique) | Refresh: find token by hash |
| `refresh_tokens_family_idx` | Revoke a whole family on logout or reuse |
| `refresh_tokens_expires_idx` | Janitor |
| `api_keys.key_prefix` (unique) | Token exchange: find key by prefix, then compare hash |

## Events written to the outbox

| Change | Event |
|--------|-------|
| Registration (organization + user + owner membership, one transaction) | `auth.org.created.v1`, `auth.user.changed.v1`, `auth.membership.changed.v1` |
| Profile edit, deactivation | `auth.user.changed.v1` |
| Member added, role changed, member removed | `auth.membership.changed.v1` (and `auth.user.changed.v1` when a new user is created) |

Payloads: [event catalog](../contracts/event-catalog.md).

## Retention

| Data | Rule |
|------|------|
| `refresh_tokens` | Deleted 7 days after `expires_at` (kept that long so reuse of a recently expired token is still recognised) |
| `signing_keys` | Retired keys kept 24 h after `retired_at`, then deleted |
| `outbox` | Published rows deleted after 24 h |
| Everything else | Kept |

## Migrations

`apps/auth/src/migrations/`. Approach: [messaging tables → migration approach](messaging-tables.md#migration-approach-all-services).
