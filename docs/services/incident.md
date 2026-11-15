# incident

**Purpose:** turn a confirmed outage into an incident, decide who must respond, and track it to resolution. **Bounded context:** Incident Response ([domain model](../02-domain-model.md)).

Decision records: [ADR-0011](../adr/0011-incident-state-machine-not-event-sourcing.md) (state machine + timeline), [ADR-0014](../adr/0014-on-call-in-incident-and-local-replicas.md) (on-call ownership, replicas), [ADR-0012](../adr/0012-idempotent-consumers.md), [ADR-0017](../adr/0017-ordering-and-replay.md).

## Responsibilities

- Open exactly one incident when a monitor is confirmed down; resolve it when the monitor recovers, is paused or is deleted.
- Enforce the incident lifecycle (`OPEN → ACKNOWLEDGED → RESOLVED`) for user and system actions.
- Keep the append-only timeline.
- Own on-call schedules and the escalation policy; compute who is on call.
- Decide **who** to page, at open and at each escalation; run escalation timers.
- Keep a local replica of organization members (`responders`).

## Explicitly not responsible for

- **Deciding whether a target is down.** It trusts `monitor.state.changed.v1` ([ADR-0009](../adr/0009-check-results-and-state-changes.md)).
- **How a person is reached** (channels, addresses, retries): notification.
- **Users, roles, login:** auth. Incident reads role from the token and names from its replica.
- **Metrics over time** (MTTA, MTTR): analytics.
- **Pushing to browsers:** realtime.

## Owned data

`incident_db`: `incidents`, `incident_events`, `monitor_cursors`, `escalation_policies`, `escalation_levels`, `escalation_targets`, `oncall_schedules`, `oncall_schedule_members`, replicas `responders` and `responder_profiles`, plus `outbox` and `processed_messages`. Schema: [incident-schema](../data/incident-schema.md).

## Sync APIs

**Exposed** ([incident.yaml](../contracts/openapi/incident.yaml)):

| Group | Endpoints |
|-------|-----------|
| Incidents | `GET /v1/incidents`, `GET /v1/incidents/{id}`, `POST /v1/incidents/{id}/acknowledge`, `POST /v1/incidents/{id}/resolve` |
| On-call | `GET`/`PUT /v1/escalation-policy`; `GET`/`POST /v1/schedules`; `GET`/`PUT`/`DELETE /v1/schedules/{id}`; `GET /v1/schedules/{id}/on-call` |
| Phase 1 only | `POST /internal/v1/monitor-signals` |

**Consumed:** JWKS from auth (cached). Nothing else.

## Events

**Consumed** ([catalog](../contracts/event-catalog.md)):

| Queue | Events | Effect |
|-------|--------|--------|
| `incident.monitor-events` (single active consumer, prefetch 1) | `monitor.state.changed.v1` | `DOWN` → open; `UP` → resolve `RECOVERED`; `PAUSED` → resolve `MONITOR_PAUSED`; stale `stateVersion` → ignore |
| | `monitor.definition.changed.v1` | `deleted: true` → resolve `MONITOR_DELETED`; otherwise ignore |
| `incident.auth-events` | `auth.org.created.v1` | Create default escalation policy |
| | `auth.user.changed.v1`, `auth.membership.changed.v1` | Upsert `responders` / `responder_profiles` if newer |
| `incident.notification-events` | `notification.delivery.succeeded.v1`, `.failed.v1` | Append `NOTIFICATION_SENT` / `NOTIFICATION_FAILED` to the timeline |

**Published:** `incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1`, `incident.page.requested.v1`.

## Internal components

| Component | Job |
|-----------|-----|
| `IncidentStateMachine` | **Pure**: transition table `(status, action) → newStatus | noop | reject(code)`. Mirrors the state diagram |
| `IncidentService.applyTransition` | The **only** writer of incidents. One transaction: conditional `UPDATE … WHERE version = ?`, timeline row, outbox rows; one reload-and-retry on version conflict |
| `MonitorEventsHandler` | Version check against `monitor_cursors`, then open or resolve |
| `PageResolver` | For a level: expand targets (`USER` → that user; `SCHEDULE` → `OnCallCalculator`), drop inactive responders, de-duplicate users |
| `OnCallCalculator` | **Pure**: `(schedule, members, t) → userId, shift bounds` |
| `EscalationTimer` | Every 5 s: claim due incidents with `SKIP LOCKED`; for each, `applyTransition(escalate)` |
| `ResponderReplica` | Versioned upserts from `auth.*` events |
| `PolicyService`, `SchedulesService` | CRUD with validation against `responders` |
| `OutboxRelay`, `Janitor` | Shared |

### Opening an incident (one transaction)

```
inbox insert (dedupe)
upsert monitor_cursors if event.stateVersion > last   → else: stale, commit, ack
INSERT incident (status OPEN, level 1, trigger_event_id, next_escalation_at = now + level1.delay)
     -- unique partial index → conflict means one is already open: commit, ack
INSERT timeline OPENED
responders = PageResolver(level 1)
for each responder: INSERT timeline PAGE_REQUESTED; INSERT outbox incident.page.requested.v1
if none: INSERT timeline NO_RESPONDER_AVAILABLE; escalate immediately if a next level exists
INSERT outbox incident.opened.v1
COMMIT → broker-ack
```

No call leaves the process. If auth, notification and realtime are all down, this still commits.

### Escalation

`next_escalation_at` is set when an incident enters a level that has a next level; cleared by acknowledge and resolve. When it fires and the incident is still `OPEN`: `escalation_level + 1`, timeline `ESCALATED`, pages for the new level, `incident.escalated.v1`, and the next timer if another level exists. The policy is read at firing time, so edits apply to open incidents.

## Scaling model

Stateless processes over one database. API scales freely. Consumers: `incident.monitor-events` is processed by exactly one consumer at a time (single active consumer; a standby replica takes over on failure); other queues use competing consumers. `EscalationTimer` is safe on every replica thanks to `SKIP LOCKED`. Locally: one replica.

## Failure modes

| Dependency down | Effect |
|-----------------|--------|
| PostgreSQL | API `503`. Handlers fail → messages go through retry queues (5 s, 30 s, 2 min) and return when the database does |
| RabbitMQ | No new signals arrive; API (list, acknowledge, resolve) keeps working; outgoing events wait in the outbox |
| auth | None for existing sessions. Replica stops updating. Pages still resolve from the replica |
| notification | Pages wait in `notification.incident-events`. Incident opens, escalates and resolves normally. Timeline lacks delivery entries until notification returns |
| monitor | No new signals. Open incidents stay open; manual resolve works |

**When incident is down:** `monitor.state.changed.v1` events queue in `incident.monitor-events` (durable). Nothing is lost; incidents open late, in order, when it returns. Users cannot list or acknowledge. Escalation timers that came due fire on restart. Flow: [service-down-recovery](../flows/service-down-recovery.md).

## Observability

| Kind | Items |
|------|-------|
| Metrics | `incident.opened`, `incident.resolved` by resolution; `incident.transitions.rejected` by code; `incident.version_conflicts`; `incident.escalations`; `incident.time_to_open_seconds` (event `changedAt` → commit); `incident.stale_events_ignored`; `incident.no_responder`; consumer lag per queue; outbox metrics |
| Spans | `incident.monitor-events process` (child of the monitor relay's publish span) → `incident.open` → DB spans; `POST /v1/incidents/{id}/acknowledge` server span; `escalation.fire` |
| Log fields | `incident_id`, `monitor_id`, `org_id`, `status`, `version`, `state_version`, `escalation_level`, `message_id`, `actor` |

## Patterns applied

[Idempotency and at-least-once](../learning/concepts/idempotency-and-at-least-once.md) · [event sourcing](../learning/concepts/event-sourcing.md) (considered, not used) · [choreography vs orchestration](../learning/concepts/choreography-vs-orchestration.md) · [sync vs async](../learning/concepts/sync-vs-async-communication.md) (local replica instead of a call) · [partitioning and per-key ordering](../learning/concepts/partitioning-and-ordering.md) · [bounded contexts](../learning/concepts/bounded-contexts-and-ddd.md) · [CAP and eventual consistency](../learning/concepts/cap-pacelc-eventual-consistency.md) (`UNKNOWN_RESPONDER`)

## Framework and Nest structure

**NestJS 12.**

| Nest element | Use |
|--------------|-----|
| Modules | `IncidentsModule`, `OnCallModule` (schedules, policy, calculator; kept separable so it could be extracted), `RespondersModule` (replica), `EscalationModule`, `ConsumersModule`, `HealthModule`; shared `MessagingModule`, `AuthGuardModule`, `ObservabilityModule` |
| Guards | `JwtAuthGuard` global; `RolesGuard` on policy and schedule writes. `/internal/v1/monitor-signals` is `@Public()` and exists only in Phase 1 |
| Pipes | Global `ValidationPipe`; `ParseUUIDPipe`; `CursorPipe` decoding the opaque pagination cursor |
| Interceptors | Logging/timing |
| Exception filter | `ProblemDetailsFilter`; domain errors (`IncidentAlreadyResolvedError`, `UnknownResponderError`) map to `409` / `422` with their codes |
| Message handlers | Plain providers registered with `Consumer.subscribe()` in `OnApplicationBootstrap`; payloads validated with the `contracts` classes; each handler receives the transaction's `EntityManager` |

## Phase introduced

Phase 1 (incidents over REST with the synchronous signal; state machine) → Phase 2 (consumers, timeline, outbox, default policy, replica) → Phase 3 (schedules, multi-level escalation, timers).

## Related

Flows: [outage-to-resolution](../flows/outage-to-resolution.md), [flapping-and-duplicates](../flows/flapping-and-duplicates.md), [service-down-recovery](../flows/service-down-recovery.md).
