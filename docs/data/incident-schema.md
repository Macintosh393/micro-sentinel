# incident_db schema

Owner: [incident](../services/incident.md). Role: `incident_svc`.

Also contains `outbox` and `processed_messages` ([messaging tables](messaging-tables.md)).

Three groups of tables: incidents (the core), on-call (schedules and the escalation policy), and replicas of other services' data.

## Incidents

```sql
CREATE TABLE incidents (
  id                     uuid        PRIMARY KEY DEFAULT uuidv7(),
  org_id                 uuid        NOT NULL,
  monitor_id             uuid        NOT NULL,          -- no FK: the monitor lives in monitor_db
  monitor_name           text        NOT NULL,          -- copied from the triggering event
  monitor_url            text        NOT NULL,
  status                 text        NOT NULL DEFAULT 'OPEN'
                                     CHECK (status IN ('OPEN', 'ACKNOWLEDGED', 'RESOLVED')),
  cause                  jsonb       NOT NULL,          -- { failureReason, statusCode }
  opened_at              timestamptz NOT NULL,
  acknowledged_at        timestamptz,
  acknowledged_by        uuid,
  resolved_at            timestamptz,
  resolved_by            uuid,                           -- NULL when resolved by the system
  resolution             text CHECK (resolution IN
                           ('RECOVERED', 'MANUAL', 'MONITOR_PAUSED', 'MONITOR_DELETED')),
  escalation_level       smallint    NOT NULL DEFAULT 1,
  next_escalation_at     timestamptz,                    -- the escalation timer; NULL = none pending
  trigger_event_id       uuid        NOT NULL UNIQUE,    -- envelope id of the DOWN event
  trigger_state_version  integer     NOT NULL,
  version                integer     NOT NULL DEFAULT 1, -- optimistic lock; sent as incidentVersion
  created_at             timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT resolved_fields CHECK (
    (status = 'RESOLVED') = (resolved_at IS NOT NULL AND resolution IS NOT NULL)),
  CONSTRAINT no_timer_unless_open CHECK (
    next_escalation_at IS NULL OR status = 'OPEN')
);

-- Invariant FR-I1: at most one unresolved incident per monitor.
CREATE UNIQUE INDEX incidents_one_unresolved_per_monitor_idx
  ON incidents (monitor_id) WHERE status <> 'RESOLVED';

-- Incident list: newest first, optionally filtered by status. Supports keyset pagination.
CREATE INDEX incidents_org_opened_idx ON incidents (org_id, opened_at DESC, id DESC);
CREATE INDEX incidents_org_status_idx ON incidents (org_id, status, opened_at DESC, id DESC);

-- Escalation timer poll.
CREATE INDEX incidents_escalation_due_idx
  ON incidents (next_escalation_at) WHERE next_escalation_at IS NOT NULL;

CREATE TABLE incident_events (                 -- the timeline; append-only
  id           uuid        PRIMARY KEY DEFAULT uuidv7(),
  incident_id  uuid        NOT NULL REFERENCES incidents (id) ON DELETE CASCADE,
  seq          integer     NOT NULL,           -- 1, 2, 3 … per incident
  type         text        NOT NULL CHECK (type IN
                 ('OPENED', 'PAGE_REQUESTED', 'NOTIFICATION_SENT', 'NOTIFICATION_FAILED',
                  'ACKNOWLEDGED', 'ESCALATED', 'RESOLVED', 'NO_RESPONDER_AVAILABLE')),
  actor_type   text        NOT NULL CHECK (actor_type IN ('USER', 'SYSTEM')),
  actor_id     uuid,                            -- user id when actor_type = 'USER'
  data         jsonb       NOT NULL DEFAULT '{}'::jsonb,
  occurred_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (incident_id, seq)
);

CREATE TABLE monitor_cursors (                 -- last state version seen per monitor
  monitor_id          uuid    PRIMARY KEY,
  last_state_version  integer NOT NULL
);
```

**Optimistic locking.** Every transition runs:

```sql
UPDATE incidents
   SET status = 'ACKNOWLEDGED', acknowledged_at = now(), acknowledged_by = $user,
       next_escalation_at = NULL, version = version + 1
 WHERE id = $id AND org_id = $org AND version = $expected_version
RETURNING *;
-- 0 rows → someone changed it first: reload, re-evaluate once.
```

No row lock is held while deciding; the `version` comparison detects the conflict at write time. This is **optimistic** concurrency: assume no conflict, verify on write. Details: [ADR-0011](../adr/0011-incident-state-machine-not-event-sourcing.md).

**Timeline sequence.** `seq` is computed as `COALESCE(max(seq), 0) + 1` inside the same transaction that holds the incident's updated row, so two writers cannot produce the same number; `UNIQUE (incident_id, seq)` backs that up.

**Keyset pagination** for `GET /v1/incidents`:

```sql
SELECT … FROM incidents
 WHERE org_id = $1
   AND ($2::text IS NULL OR status = $2)
   AND (opened_at, id) < ($cursor_opened_at, $cursor_id)   -- omitted on the first page
 ORDER BY opened_at DESC, id DESC
 LIMIT $limit + 1;                                          -- one extra row tells us if there is a next page
```

Unlike `OFFSET`, this stays fast on page 500 and does not skip or repeat rows when new incidents arrive between requests.

**Escalation timer poll** (every 5 s):

```sql
SELECT id, version FROM incidents
 WHERE next_escalation_at <= now()
 ORDER BY next_escalation_at
 LIMIT 20
   FOR UPDATE SKIP LOCKED;
```

Same pattern as the check leases and the outbox relay. The transition itself re-checks `status = 'OPEN'`.

## On-call and escalation

```sql
CREATE TABLE escalation_policies (
  id          uuid        PRIMARY KEY DEFAULT uuidv7(),
  org_id      uuid        NOT NULL UNIQUE,        -- one policy per organization in v1
  name        text        NOT NULL DEFAULT 'Default',
  version     integer     NOT NULL DEFAULT 1,
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE escalation_levels (
  id             uuid     PRIMARY KEY DEFAULT uuidv7(),
  policy_id      uuid     NOT NULL REFERENCES escalation_policies (id) ON DELETE CASCADE,
  level          smallint NOT NULL CHECK (level >= 1),
  delay_seconds  integer  NOT NULL DEFAULT 300 CHECK (delay_seconds BETWEEN 60 AND 3600),
  UNIQUE (policy_id, level)
);

CREATE TABLE escalation_targets (
  id           uuid PRIMARY KEY DEFAULT uuidv7(),
  level_id     uuid NOT NULL REFERENCES escalation_levels (id) ON DELETE CASCADE,
  target_type  text NOT NULL CHECK (target_type IN ('USER', 'SCHEDULE')),
  target_id    uuid NOT NULL,                     -- responders.user_id or oncall_schedules.id
  UNIQUE (level_id, target_type, target_id)
);

CREATE TABLE oncall_schedules (
  id                  uuid        PRIMARY KEY DEFAULT uuidv7(),
  org_id              uuid        NOT NULL,
  name                text        NOT NULL CHECK (length(name) BETWEEN 1 AND 100),
  rotation_start      timestamptz NOT NULL,       -- when position 0 starts their first shift
  shift_length_hours  integer     NOT NULL DEFAULT 168 CHECK (shift_length_hours BETWEEN 1 AND 720),
  created_at          timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, name)
);

CREATE TABLE oncall_schedule_members (
  schedule_id  uuid     NOT NULL REFERENCES oncall_schedules (id) ON DELETE CASCADE,
  position     smallint NOT NULL CHECK (position >= 0),
  user_id      uuid     NOT NULL,
  PRIMARY KEY (schedule_id, position)
);
```

**Who is on call at time `t`:**

```
shiftsElapsed = floor((t − rotation_start) / shift_length_hours)
onCall        = members[ shiftsElapsed mod count(members) ]
```

A pure function with no state, trivial to unit-test, and computable inside the transaction that opens an incident. Overrides ("Bob covers for Alice on Friday") are out of scope ([open questions](../open-questions.md)).

`delay_seconds` on a level is how long the incident may stay `OPEN` at that level before escalating to the next. The last level's delay is unused.

Replacing the policy (`PUT /v1/escalation-policy`) deletes and re-inserts levels and targets in one transaction and increments `version`. Incidents already open keep their `escalation_level` number and read the new policy at their next escalation.

## Replicas

```sql
CREATE TABLE responders (                       -- copy of auth's users + memberships
  user_id       uuid    NOT NULL,
  org_id        uuid    NOT NULL,
  display_name  text    NOT NULL DEFAULT '',
  role          text,                           -- NULL = no longer a member
  active        boolean NOT NULL DEFAULT true,
  user_version        integer NOT NULL DEFAULT 0,   -- from auth.user.changed.v1
  membership_version  integer NOT NULL DEFAULT 0,   -- from auth.membership.changed.v1
  PRIMARY KEY (org_id, user_id)
);
```

Updated only by the `incident.auth-events` consumer:

```sql
-- auth.membership.changed.v1
INSERT INTO responders (org_id, user_id, role, membership_version)
VALUES ($1, $2, $3, $4)
ON CONFLICT (org_id, user_id) DO UPDATE
   SET role = EXCLUDED.role, membership_version = EXCLUDED.membership_version
 WHERE responders.membership_version < EXCLUDED.membership_version;
```

The `WHERE` on the conflict branch is the whole ordering defence: an older or duplicate snapshot changes nothing. Two version columns are needed because the row merges two independent event streams. If `auth.user.changed.v1` arrives before any membership for that user, the name is stored in a row per known organization on the next membership event; the consumer keeps the latest user snapshot in a small side table `responder_profiles (user_id, display_name, active, user_version)` to make that possible.

```sql
CREATE TABLE responder_profiles (
  user_id       uuid    PRIMARY KEY,
  display_name  text    NOT NULL,
  active        boolean NOT NULL,
  user_version  integer NOT NULL
);
```

## Indexes and the queries they serve

| Index | Query |
|-------|-------|
| `incidents_one_unresolved_per_monitor_idx` (unique, partial) | Enforces FR-I1; also "find the open incident for this monitor" |
| `incidents_org_opened_idx`, `incidents_org_status_idx` | Incident list with keyset pagination |
| `incidents_escalation_due_idx` (partial) | Escalation poll |
| `incidents.trigger_event_id` (unique) | Same DOWN event cannot open two incidents |
| `incident_events (incident_id, seq)` (unique) | Timeline in order |
| `escalation_policies.org_id` (unique) | Policy lookup at incident open |

## Events written to the outbox

`incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1`, `incident.page.requested.v1`. See [event catalog](../contracts/event-catalog.md).

## Retention

Incidents, timeline, policies and schedules are kept indefinitely in v1. `outbox` 24 h; `processed_messages` 7 days.

## Migrations

`apps/incident/src/migrations/`, run by `incident-migrate`. Approach: [messaging tables](messaging-tables.md#migration-approach-all-services).
