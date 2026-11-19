# ADR-0014: Own on-call and escalation in Incident, and replace synchronous lookups with local replicas

- Status: Accepted
- Date: 2026-11-15
- Phase: 2 (replicas, default policy), 3 (schedules, multi-level escalation)

## Context

The brief has Incident call Auth/Org synchronously over gRPC when an incident opens: "who is on call for organization 42?". Two problems.

**Temporal coupling.** The moment an incident opens is the moment something is already broken. If auth is down or slow at that moment (perhaps because of the same outage), the incident cannot be routed. Availability multiplies: two services at 99.5 % each give 99.0 % for the combined path, and that path is the one that matters most. Retrying helps with blips, not with an outage.

**Wrong owner.** Is an on-call schedule identity data? Auth's question is "who are you and what may you do". "Who should be woken up for this incident, and who next if they don't answer" is an incident-response rule. It changes when incident-response features change (overrides, rotations, escalation delays), not when login changes.

## Decision drivers

- An incident must be routable when every other service is down.
- Each piece of data should live in the context whose rules it serves.
- No distributed transactions; no synchronous calls on the outage path.
- Staleness of a second or two in "who is a member" is acceptable.

## Options considered

### Where do schedules and escalation policies live?

| Option | Pros | Cons |
|--------|------|------|
| A. Auth/Org (as in the brief) | "Users are there already" | Makes auth a dependency of incident routing; mixes identity with incident response; auth grows every time on-call features grow |
| B. A separate On-Call service | Clean, focused context; could be reused by other products | An eighth service; Incident still needs its data at incident time, so the coupling question returns one hop away |
| C. **Incident** (chosen) | The lookup becomes a local query in the same transaction that opens the incident; escalation timers and policies share one model | Incident is larger: incidents + schedules + policies |

### How does a service get data it does not own?

| Option | Pros | Cons |
|--------|------|------|
| D. Synchronous call (REST or gRPC) at the moment of need | Always fresh; no copy to maintain | Temporal coupling; latency added to the hot path; needs timeouts, retries, a circuit breaker and a fallback |
| E. Synchronous call with a cache | Survives short outages of the owner | Cold cache after a restart = same coupling; invalidation rules to invent |
| F. **Local replica built from events** (chosen) | The data is in the consumer's own database: no call, no coupling, joins work | Eventually consistent; extra table and consumer; needs a bootstrap path |
| G. Put the data in the token (claims) | Zero lookups | Only works for data about the *caller*; an incident opened by a system event has no caller |

## Decision

**C + F.** There are no synchronous calls between domain services at runtime, and no gRPC ([communication](../architecture/communication.md)).

**Ownership**

| Data | Owner |
|------|-------|
| Users, organizations, memberships, roles | auth |
| On-call schedules, escalation policy, escalation timers | incident |
| Contact methods, organization channels | notification |

**Replicas**

| Replica | Service | Built from | Used for |
|---------|---------|------------|----------|
| `responders` | incident | `auth.user.changed.v1`, `auth.membership.changed.v1` | Validating that schedule members and escalation targets belong to the organization; display names in the timeline |
| `recipients` | notification | `auth.user.changed.v1` | The email address and name used in messages |

Rules for replicas ([data ownership](../architecture/data-ownership.md)):

- Replica events are **versioned snapshots**; the consumer upserts when `incoming.version > stored.version`. Duplicates and reordering are harmless.
- A replica is **read-only** for its service. It is changed only by the event consumer.
- A replica holds the minimum fields. `responders` has no email; `recipients` has no role.
- **Bootstrap:** a service with an empty replica replays `ms.archive.lifecycle` from the first offset. For data that existed before events did (Phase 1 users), auth has a one-off command that writes a snapshot for every user and membership into its outbox.

**Choreography on organization creation.** `auth.org.created.v1` → Incident creates the organization's default escalation policy (level 1: the owner, 300 s delay). No call, no orchestrator.

**The page is resolved at incident time, in Incident's database.** Opening an incident, reading escalation level 1, computing who is on call from the schedule, writing the timeline and writing the `incident.page.requested.v1` outbox rows are one local transaction.

**Validation at write time.** When an admin saves a schedule, Incident checks the member IDs against `responders`. If a user was created less than a second ago and the replica has not caught up, the API returns `422` with code `UNKNOWN_RESPONDER`; the UI retries once. This is what eventual consistency looks like at an API boundary, and it is the honest cost of option F.

**Stale membership.** If a user is removed from the organization, `auth.membership.changed.v1` marks the responder inactive. Incident skips inactive responders when resolving targets; if a level then has nobody, it escalates immediately to the next level, and if no level has anybody it records `NO_RESPONDER_AVAILABLE` on the timeline (organization channels still fire).

## Consequences

- Positive: the outage path depends only on Incident's own database and the broker. Auth can be down for an hour and pages still go out ([break-it lab](../roadmap/break-it-lab.md)).
- Positive: no gRPC, no service client code, no circuit breakers between services.
- Negative: Incident is the largest service. Its on-call module must stay a separate Nest module with its own tables so it can be extracted later.
- Negative: data is duplicated; personal data exists in three databases.
- Negative: brief windows where a replica is behind; one API error code exists purely because of it.
- Harder later: extracting an On-Call service would reintroduce the question of how Incident gets on-call data at incident time. The answer would again be a replica.

## Revisit when

- On-call features (overrides, layers, follow-the-sun) grow until they dominate Incident's code: extract option B, feeding Incident a replica.
- A second product needs the same schedules.
- A use case needs strictly fresh data from another service (for example a permission check that must not be stale): that single call may be synchronous, with a timeout and a deny-by-default fallback.

## Learning choice?

Partly. Owning on-call in Incident and avoiding the synchronous call is what an experienced team would do. Not having *any* synchronous inter-service call is stricter than most real systems: a pragmatic team would allow a few synchronous reads on non-critical paths rather than build a replica for each. gRPC is therefore not exercised here; it is covered in the [sync vs async guide](../learning/concepts/sync-vs-async-communication.md).

## How to explain it in an interview

> "The first design had the incident service call the auth service to ask who's on call. That couples them in time: if auth is down exactly when a site goes down, I can't page anyone. I changed two things. First, on-call schedules and escalation policies moved into the incident service, because they're incident-response rules, not identity. Second, for the user data incident still needs, it keeps a small local copy updated from auth's events. The events are versioned snapshots so duplicates and reordering don't matter. The trade-off is eventual consistency: a brand-new user might not be assignable for a second. That's a much better failure than not being able to page."
