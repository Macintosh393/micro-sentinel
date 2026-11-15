# Data ownership

**Database-per-service** means each service is the only reader and writer of its own data store. Other services get data by asking through an API or by keeping their own copy from events. Background: [database-per-service](../learning/concepts/database-per-service.md).

## Why bother

Without it, two services that share a table are one service with two deploy pipelines. A column rename in one breaks the other, nobody can change the schema without a meeting, and you cannot tell from the code who writes what. That is the **distributed monolith**: the costs of microservices with none of the independence.

## Who owns what

| Data | System of record | Database | Others that hold a copy | How the copy is made |
|------|------------------|----------|-------------------------|----------------------|
| Users, credentials | auth | `auth_db` | incident (`responders`), notification (`recipients`) | `auth.user.changed.v1`, `auth.membership.changed.v1` |
| Organizations, memberships, roles | auth | `auth_db` | incident (`responders` has org + role) | `auth.membership.changed.v1`; everyone else reads org and role from the JWT |
| API keys, refresh tokens | auth | `auth_db` | nobody | — |
| Monitor definitions | monitor | `monitor_db` | analytics (`monitors_dim`); incident stores name + URL on each incident | `monitor.definition.changed.v1`; `monitor.state.changed.v1` carries name and URL |
| Monitor status, counters, leases | monitor | `monitor_db` | realtime forwards changes; nobody stores them | `monitor.state.changed.v1` |
| Recent checks (24 h) | monitor | `monitor_db` | — | — |
| Check history (30 d raw + rollups) | analytics | `analytics_db` | — | Derived from `monitor.check.completed.v1` |
| Incidents and timeline | incident | `incident_db` | analytics (`incident_facts`) | `incident.*` events |
| On-call schedules, escalation policy | incident | `incident_db` | nobody | — |
| Contact methods, org channels | notification | `notification_db` | nobody | — |
| Deliveries | notification | `notification_db` | incident timeline gets a summary line | `notification.delivery.*` events |
| Rate-limit counters | gateway | Redis | nobody | Ephemeral |
| Event history for replay | the broker | RabbitMQ streams | analytics reads it | [ADR-0017](../adr/0017-ordering-and-replay.md) |

## Local replicas

A **local replica** is a small table in service B containing the few fields B needs from service A, updated by A's events.

| Replica | In | Fields | Source event | Staleness seen in practice |
|---------|----|--------|--------------|---------------------------|
| `responders` | incident | `user_id`, `org_id`, `role`, `display_name`, `active`, `version` | `auth.user.changed.v1`, `auth.membership.changed.v1` | < 1 s |
| `recipients` | notification | `user_id`, `email`, `display_name`, `version` | `auth.user.changed.v1` | < 1 s |
| `monitors_dim` | analytics | `monitor_id`, `org_id`, `name`, `url`, `deleted` | `monitor.definition.changed.v1` | < 2 s |

All replica events are **snapshots with a version**: they carry the full current state of the thing, not a diff. The consumer does `UPSERT … WHERE incoming.version > stored.version`. That one rule makes the replica correct under duplicates and under reordering, with no extra bookkeeping. This style is called **event-carried state transfer**.

### What the replica costs

- **Staleness.** A user added a second ago might not be pageable yet. Acceptable here.
- **Storage and code.** One more table and one more consumer per replica.
- **Bootstrap.** A new service starts with an empty replica. It fills by replaying `ms.archive.lifecycle` from the first offset (the same mechanism as the [analytics rebuild](../flows/analytics-rebuild.md)).
- **Privacy.** Personal data now lives in more than one database. Deleting a user means publishing a snapshot that consumers honour. Out of scope here, but name it if asked.

### What it buys

Incident can route a page while auth is completely down ([break-it lab](../roadmap/break-it-lab.md), experiment "Auth down while incidents are being opened").

## Consistency guarantees

| Scope | Guarantee | Mechanism |
|-------|-----------|-----------|
| Inside one service | **Strong** (ACID) | One PostgreSQL transaction covers the domain write, the timeline entry, the outbox row and the inbox row |
| Between a service's data and its published events | **Atomic, eventually published** | Transactional outbox: the event exists if and only if the data change committed |
| Between services | **Eventual** | At-least-once delivery + idempotent consumers; converges once queues drain |
| Read model vs. source | **Eventual**, typically < 5 s behind | Stream consumer; lag is a metric |
| Browser vs. server | **Eventual**, typically < 2 s | SSE hint, then refetch |

There are **no distributed transactions** (no two-phase commit) anywhere. Where a business operation spans services (down → incident → page → delivery) it is a chain of local transactions connected by events: a **saga** in the loose, choreographed sense. Nothing in this system needs a compensating action, because every step is "record a fact"; that is worth knowing and saying ([choreography vs orchestration](../learning/concepts/choreography-vs-orchestration.md)).

## Cross-service references

IDs cross service boundaries; foreign keys do not.

- `incidents.monitor_id` is a plain `uuid` column with **no foreign key** (the monitor row is in another database).
- Consequence: **no referential integrity across services.** A monitor can be deleted while an incident still references it. The owning event (`monitor.definition.changed.v1` with `deleted: true`) tells incident to resolve the open incident, and the incident keeps the monitor name and URL it copied at open time, so history stays readable.
- All IDs are UUIDv7 (time-ordered), generated by the owning service with PostgreSQL 18's `uuidv7()`.

## Multi-tenancy

Every table that holds tenant data has an `org_id` column, and every query filters on the `org` claim of the verified token. No endpoint takes an organization ID as input. This is application-level isolation; row-level security in PostgreSQL would be a stronger second layer and is listed in [open questions](../open-questions.md).

## One server, five databases: is that cheating?

Short answer: it breaks **operational** isolation and keeps **logical** isolation. The full argument is [ADR-0018](../adr/0018-single-postgres-and-ram-budget.md).

| Property | One DB per service on one server | Shared schema (the anti-pattern) |
|----------|----------------------------------|----------------------------------|
| Can service A query B's tables? | No: separate database, separate role, no grant | Yes |
| Can A's migration break B? | No | Yes |
| Does a slow query in A slow down B? | **Yes** (shared CPU, memory, disk) | Yes |
| Does the server going down take everyone down? | **Yes** | Yes |
| Can A and B run different PostgreSQL versions? | **No** | No |
| Can it be split onto separate servers later without code changes? | Yes: change `DATABASE_URL` | No |

## Retention

| Data | Kept for | Enforced by |
|------|----------|-------------|
| Outbox rows (published) | 24 h | Janitor job in each service |
| `processed_messages` (inbox) | 7 days | Janitor job |
| Recent checks in `monitor_db` | 24 h | Janitor job |
| Raw check results in `analytics_db` | 30 days | Dropping daily partitions |
| Hourly rollups | 13 months | Janitor job |
| `ms.archive.checks` stream | 35 days or 2 GB | Broker (`x-max-age`, `x-max-length-bytes`) |
| `ms.archive.lifecycle` stream | 5 years or 1 GB | Broker |
| Incidents, timeline, deliveries | Forever (v1) | — |

Schemas: [auth](../data/auth-schema.md), [monitor](../data/monitor-schema.md), [incident](../data/incident-schema.md), [notification](../data/notification-schema.md), [analytics](../data/analytics-schema.md), and the shared [messaging tables](../data/messaging-tables.md).
