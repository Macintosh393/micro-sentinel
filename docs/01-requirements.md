# Requirements

Terms in this document are defined in the [glossary](03-glossary.md). Numbers here are the single source of truth for defaults; service docs and ADRs repeat them only by reference.

## Actors

| Actor | Description |
|-------|-------------|
| **Owner** | Created the organization. Has every admin right and cannot be removed. |
| **Admin** | Manages monitors, members, channels, schedules, the escalation policy and API keys. |
| **Member** | Views everything in the organization, acknowledges and resolves incidents, manages their own contact methods. |
| **API client** | A script holding an API key (for example a CI job that creates monitors). |
| **Target** | The external HTTP endpoint being checked. Untrusted. |

## User stories

Phase numbers show when each story is delivered ([roadmap](roadmap/phases.md)).

| ID | Story | Phase |
|----|-------|-------|
| US-01 | As a visitor I can register, which creates my user and my organization. | 1 |
| US-02 | As a user I can log in, stay logged in across page reloads, and log out. | 1 |
| US-03 | As an admin I can create, edit, pause, resume and delete monitors. | 1 |
| US-04 | As a user I can see each monitor's current status and its recent checks. | 1 |
| US-05 | As a user I can see a list of incidents and open one to see its details. | 1 |
| US-06 | As a user I can acknowledge and resolve an incident. | 1 |
| US-07 | As a user I see incidents open, get acknowledged and resolve without refreshing the page. | 2 |
| US-08 | As a user I can read an incident's timeline: opened, paged, notified, acknowledged, escalated, resolved. | 2 |
| US-09 | As the on-call user I receive an email when an incident opens. | 2 |
| US-10 | As an admin I can add organization channels (generic webhook, chat webhook) that receive incident events. | 2 |
| US-11 | As a user I can add contact methods (email, SMS number) used when I am paged. | 3 |
| US-12 | As an admin I can define an on-call schedule (a rotation of users). | 3 |
| US-13 | As an admin I can define the escalation policy: who is paged first, and who is paged if nobody acknowledges in time. | 3 |
| US-14 | As an admin I can see failed deliveries and retry them. | 3 |
| US-15 | As an admin I can add members to my organization and change their role. | 3 |
| US-16 | As an admin I can create and revoke API keys. | 3 |
| US-17 | As a user I can see uptime %, latency, MTTA and MTTR per monitor and for the organization. | 4 |
| US-18 | As an operator I can follow one outage as a single trace from the failed check to the delivered email. | 4 |

## Functional requirements

### Monitoring

- **FR-M1** A monitor has: name, URL (http/https), method (`GET` or `HEAD`), `intervalSeconds`, `timeoutMs`, `acceptedStatusCodes`, `failureThreshold`, `recoveryThreshold`, and a paused flag.
- **FR-M2** Each active monitor is checked once per interval. A check is a single HTTP request with a timeout; redirects are not followed (a 3xx is judged against `acceptedStatusCodes`).
- **FR-M3** A monitor's status is one of `UNKNOWN`, `UP`, `DOWN`, `PAUSED`. Status changes only after confirmation: `failureThreshold` consecutive failed checks to go `DOWN`, `recoveryThreshold` consecutive successful checks to go `UP` ([ADR-0009](adr/0009-check-results-and-state-changes.md)).
- **FR-M4** While a status change is unconfirmed, the next check runs after `retryIntervalSeconds` instead of the full interval.
- **FR-M5** Targets that resolve to private, loopback or link-local addresses are refused unless explicitly allow-listed ([security](cross-cutting/security.md)).
- **FR-M6** Every check result and every confirmed status change is published as an event.

### Incidents

- **FR-I1** When a monitor becomes `DOWN`, exactly one incident is opened. There is never more than one unresolved incident per monitor.
- **FR-I2** Incident status is `OPEN`, `ACKNOWLEDGED` or `RESOLVED`. Legal transitions and the handling of illegal ones are in the [domain model](02-domain-model.md#incident-lifecycle).
- **FR-I3** When the monitor becomes `UP` again, its unresolved incident is resolved automatically. Pausing or deleting the monitor also resolves it.
- **FR-I4** Every change to an incident is appended to its timeline.
- **FR-I5** On open, the incident pages the targets of escalation level 1. If the incident is still `OPEN` when the level's delay expires, it escalates to the next level and pages its targets.
- **FR-I6** Acknowledging stops escalation. Resolving stops everything.

### Notifications

- **FR-N1** A page is delivered through every enabled contact method of the paged user.
- **FR-N2** Organization channels receive the incident events they subscribe to.
- **FR-N3** One logical notification (one event, one channel, one target) is delivered at most once under normal operation, and at-least-once under failure; duplicates are suppressed by a dedupe key ([ADR-0013](adr/0013-notification-owns-delivery-state.md)).
- **FR-N4** Failed sends are retried with increasing delay; after the last attempt the delivery is marked `DEAD`, parked in a dead-letter queue, and can be retried manually.

### Analytics

- **FR-A1** Uptime % per monitor for 24 h, 7 d, 30 d and custom ranges, in hourly or daily buckets.
- **FR-A2** Latency (average and p95) per monitor per bucket.
- **FR-A3** Incident count, MTTA and MTTR per monitor and per organization.
- **FR-A4** The read model can be deleted and rebuilt from the retained event history ([flow](flows/analytics-rebuild.md)).

### Identity and access

- **FR-U1** Registration, login, token refresh, logout.
- **FR-U2** Every request is scoped to exactly one organization, taken from the token. No endpoint accepts an organization ID from the client.
- **FR-U3** Role checks as in the Actors table.
- **FR-U4** API keys can be exchanged for short-lived access tokens.

## Non-functional requirements

Targets are for the reference dev machine (Windows 11, Docker Desktop on WSL2, ~8 GB for the stack).

### Capacity (design targets)

| Quantity | Target | Note |
|----------|--------|------|
| Monitors | 500 | Design ceiling for the core phases |
| Minimum check interval | 30 s (10 s when `ALLOW_FAST_INTERVALS=true`, dev only) | |
| Peak check rate | ~17 checks/s (500 monitors ÷ 30 s) | ~1.44 M check results/day |
| Concurrent in-flight checks per worker | 50 | Hard cap ([event loop guide](learning/concepts/node-event-loop.md)) |
| Organizations | 20 | |
| Concurrent browser sessions | 50 | Each holds one SSE connection |

### Latency

| Path | Target |
|------|--------|
| Read API (p95) | < 200 ms |
| Write API (p95) | < 400 ms |
| Failure detection, default settings (30 s interval, threshold 3, retry 10 s, 10 s timeout) | Incident opened ≤ 90 s after the target starts failing |
| Incident opened → page requested | ≤ 2 s (p95) |
| Page requested → first delivery attempt | ≤ 5 s (p95) |
| Incident change → visible in browser | ≤ 2 s (p95) |

### Reliability

- **NFR-R1** No confirmed status change, incident event or page request is lost if any single application container or the broker is killed and restarted.
- **NFR-R2** Duplicate delivery of any event causes no duplicate incident and no duplicate delivery record.
- **NFR-R3** A service being down delays, but does not drop, the work destined for it. Messages wait in durable queues.
- **NFR-R4** Killing one of N Monitor workers delays its in-flight checks by at most one lease duration (60 s).
- **NFR-R5** A single poison message cannot block a queue for more than 5 delivery attempts.

### Defaults and limits

| Setting | Default | Range |
|---------|---------|-------|
| `intervalSeconds` | 60 | 30–3600 |
| `timeoutMs` | 10 000 | 1 000–30 000 |
| `acceptedStatusCodes` | `200-299` | list of codes and ranges |
| `failureThreshold` | 3 | 1–10 |
| `recoveryThreshold` | 2 | 1–10 |
| `retryIntervalSeconds` | 10 | fixed in v1 |
| Check lease duration | 60 s | must exceed max timeout |
| Escalation delay per level | 300 s | 60–3600 |
| Access token lifetime | 15 min | |
| Refresh token lifetime | 7 days, rotated on use | |
| Gateway → upstream timeout | 10 s | SSE exempt |
| Notification send timeout | 5 s | |
| Delivery attempts | 5 (delays 10 s, 1 min, 5 min, 15 min) | |

### Operability

- **NFR-O1** `docker compose up` brings the full stack to healthy in under 120 s once images are pulled, with no network access.
- **NFR-O2** Total memory of the full stack (all profiles) stays under 8 GB; the core profile under 4 GB.
- **NFR-O3** Every service exposes `/health/live` and `/health/ready`.
- **NFR-O4** Every log line is JSON and carries `service`, `trace_id` and `span_id` once tracing is on (Phase 4), and a `request_id` from Phase 0.

### Security

- **NFR-S1** Passwords hashed with Argon2id. Refresh tokens and API keys stored only as hashes.
- **NFR-S2** No secret in the repository except clearly labelled development defaults.
- **NFR-S3** Every service verifies the JWT itself ([ADR-0015](adr/0015-jwt-rs256-jwks-validate-everywhere.md)).
- **NFR-S4** User-supplied URLs (monitor targets and webhook channels) pass the SSRF guard before any request is made.

## Assumptions

1. A single region and a single check location. "Down" means "down as seen from our workers".
2. Clocks of all containers are in sync (they share the host kernel clock). Lease correctness does not depend on this: leases use the database clock ([ADR-0010](adr/0010-postgres-leases-for-scheduling.md)).
3. A user belongs to exactly one organization in v1. The schema allows more.
4. One escalation policy per organization in v1; it applies to all of the organization's monitors.
5. Uptime % is check-based: successful checks ÷ total checks in the window ([Analytics](services/analytics.md)).
6. Development happens with the repository inside the WSL2 filesystem ([deployment-local](architecture/deployment-local.md)); see [open questions](open-questions.md).

## Out of scope

See [non-goals](00-vision-and-scope.md#non-goals). Requests that come up later go to [open questions](open-questions.md), not into the phases.
