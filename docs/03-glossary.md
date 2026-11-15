# Glossary

This is the **single source of truth for names**. Every other document, table, event, queue and class uses these words with exactly these meanings. If a doc and the glossary disagree, the glossary wins and the doc has a bug.

## Naming conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Service (Compose name, folder under `apps/`) | lowercase, one word | `monitor`, `auth` |
| Event type | `<service>.<entity>.<past-tense verb>.v<major>` | `monitor.state.changed.v1` |
| Command message (internal to one service) | `<service>.<entity>.<imperative verb>.v<major>` | `notification.delivery.send.v1` |
| Exchange | `ms.<purpose>` for shared, `<service>.<purpose>` for private | `ms.events`, `notification.work` |
| Queue | `<consuming service>.<what it holds>` | `incident.monitor-events` |
| Wait queue / dead-letter queue | `<queue>.retry.<delay>` / `<queue>.dlq` | `notification.delivery-jobs.retry.1m` |
| Database | `<service>_db`, role `<service>_svc` | `incident_db`, `incident_svc` |
| Table and column | `snake_case`, plural tables | `incident_events.occurred_at` |
| JSON field (HTTP and events) | `camelCase` | `stateVersion` |
| Enum value | `UPPER_SNAKE` | `ACKNOWLEDGED` |
| HTTP path | plural nouns, kebab-case, versioned | `/v1/contact-methods` |
| Environment variable | `UPPER_SNAKE`, service-agnostic names | `DATABASE_URL` |
| Workspace package | `@microsentinel/<name>` | `@microsentinel/contracts` |

## Domain terms

| Term | Meaning | Owner |
|------|---------|-------|
| **Organization** (org) | The tenant. Every monitor, incident, channel and schedule belongs to exactly one. | auth |
| **User** | A person who can log in. | auth |
| **Membership** | The link between a user and an organization, with a **role**. | auth |
| **Role** | `OWNER`, `ADMIN` or `MEMBER`. | auth |
| **API key** | A long-lived secret for scripts, exchanged for an access token. | auth |
| **Access token** | A JWT, valid 15 minutes, sent as `Authorization: Bearer`. | auth |
| **Refresh token** | An opaque, rotating secret in an httpOnly cookie, used to get a new access token. | auth |
| **Monitor** | A URL plus the settings for checking it. | monitor |
| **Target** | The external endpoint a monitor points at. | — |
| **Check** | One HTTP request to a target, with its outcome. | monitor |
| **Check outcome** | `SUCCESS` or `FAILURE`. A failure has a **failure reason**: `TIMEOUT`, `DNS_ERROR`, `CONNECTION_ERROR`, `TLS_ERROR`, `UNEXPECTED_STATUS`, `BLOCKED_TARGET`. | monitor |
| **Monitor status** | The *confirmed* state: `UNKNOWN`, `UP`, `DOWN`, `PAUSED`. Not the same as the last check outcome. | monitor |
| **Confirmation** | The rule that turns check outcomes into a status change: `failureThreshold` consecutive failures, or `recoveryThreshold` consecutive successes. | monitor |
| **Flapping** | A target alternating between failing and succeeding. Damped by confirmation. | monitor |
| **State version** (`stateVersion`) | A counter per monitor, +1 on every status change. Lets consumers discard stale or duplicate changes. | monitor |
| **Definition version** (`definitionVersion`) | A counter per monitor, +1 on every edit of its settings. | monitor |
| **Lease** | A time-limited, exclusive claim by one worker on one monitor's next check. | monitor |
| **Lease token** | A number that increases every time a lease is granted; used as a **fencing token**. | monitor |
| **Worker** | A `monitor-worker` process that claims leases and runs checks. | monitor |
| **Incident** | The record that a monitor was confirmed down, from open to resolution. | incident |
| **Incident status** | `OPEN`, `ACKNOWLEDGED`, `RESOLVED`. | incident |
| **Acknowledge** (ack) | A user says "I am on it". Stops escalation. Not the same as a broker ack. | incident |
| **Resolve** | The incident is over, manually or automatically. | incident |
| **Resolution** | Why it ended: `RECOVERED`, `MANUAL`, `MONITOR_PAUSED`, `MONITOR_DELETED`. | incident |
| **Timeline** | The append-only list of things that happened to an incident (`incident_events` table). | incident |
| **Responder** | Incident's view of a user who can be paged. | incident |
| **On-call schedule** | A rotation: an ordered list of responders and a shift length. | incident |
| **On call** | The one responder a schedule selects at a given instant. | incident |
| **Escalation policy** | Ordered **levels**; each level has **targets** (users or schedules) and a delay. | incident |
| **Escalation** | Moving an unacknowledged incident to the next level. | incident |
| **Page** | A request to notify one specific responder about one incident at one level. | incident |
| **Contact method** | A way to reach a user: `EMAIL` or `SMS` with an address or number. | notification |
| **Organization channel** | A shared destination for incident events: `WEBHOOK` or `CHAT_WEBHOOK`. | notification |
| **Recipient** | Notification's view of a user. | notification |
| **Delivery** | One message to one target through one channel, with its attempts and status (`PENDING`, `SENT`, `DEAD`). | notification |
| **Dedupe key** | A string that is identical for two attempts to create the same delivery. | notification |
| **Redrive** | Manually retrying a `DEAD` delivery. | notification |
| **Uptime %** | Successful checks ÷ total checks in a time window. | analytics |
| **MTTA** | Mean time to acknowledge: average of (acknowledged − opened). | analytics |
| **MTTR** | Mean time to resolve: average of (resolved − opened). | analytics |
| **Read model** | Tables derived from events, shaped for queries, disposable. | analytics |

## Technical terms

| Term | Meaning | Learn more |
|------|---------|-----------|
| **Event** | A fact that already happened, published for anyone interested. Named in the past tense. | [event-driven architecture](learning/concepts/event-driven-architecture.md) |
| **Command** | A request for one specific receiver to do something. Named in the imperative. | same |
| **Envelope** | The standard wrapper around every event: `id`, `type`, `source`, `time`, `subject`, `orgId`, `sequence`, `data`. | [versioning](contracts/versioning.md) |
| **Broker** | RabbitMQ: the server that routes and stores messages. | [ADR-0006](adr/0006-rabbitmq-as-broker.md) |
| **Exchange** | Where publishers send messages; it routes them to queues by routing key. | same |
| **Queue** | Where messages wait for a consumer. A message is removed when acked. | same |
| **Stream** | A RabbitMQ queue type that keeps messages after they are read, so they can be replayed. | [ADR-0017](adr/0017-ordering-and-replay.md) |
| **Routing key** | A label on a message used by the exchange to choose queues. Always equals the event type. | same |
| **Broker ack / nack** | A consumer telling the broker a message was (not) handled. Unrelated to acknowledging an incident. | [idempotency guide](learning/concepts/idempotency-and-at-least-once.md) |
| **Prefetch** | How many unacked messages the broker will give one consumer at a time. | [backpressure](learning/concepts/backpressure.md) |
| **Publisher confirm** | The broker telling a publisher a message is safely stored. | [outbox](learning/concepts/transactional-outbox.md) |
| **Outbox** | A table where a service writes events in the same transaction as its data. | [ADR-0008](adr/0008-transactional-outbox.md) |
| **Relay** | The background loop that reads the outbox and publishes to the broker. | same |
| **Inbox** | The `processed_messages` table a consumer uses to detect duplicates. | [ADR-0012](adr/0012-idempotent-consumers.md) |
| **Idempotent** | Doing it twice has the same effect as doing it once. | same |
| **DLQ** (dead-letter queue) | A parking queue for messages that could not be processed. | [DLQ guide](learning/concepts/dead-letter-queues.md) |
| **Poison message** | A message that fails every time it is processed. | same |
| **Wait queue** | A queue with a TTL and no consumer; messages expire back to the main queue. Implements delayed retry. | [ADR-0023](adr/0023-retry-topology-ttl-dlx.md) |
| **Fencing token** | A number that lets a resource reject writes from a holder whose lease has expired. | [locks guide](learning/concepts/distributed-locks-leases-fencing.md) |
| **Local replica** | A service's own copy of another service's data, kept up to date from events. | [ADR-0014](adr/0014-on-call-in-incident-and-local-replicas.md) |
| **Temporal coupling** | Service A can only work if service B is up *at the same moment*. | [sync vs async](learning/concepts/sync-vs-async-communication.md) |
| **Eventual consistency** | Copies of data may differ briefly but converge if no new writes arrive. | [CAP guide](learning/concepts/cap-pacelc-eventual-consistency.md) |
| **JWKS** | JSON Web Key Set: the public keys used to verify JWTs, served by auth. | [JWT guide](learning/concepts/jwt-and-jwks.md) |
| **SSE** | Server-Sent Events: a one-way HTTP stream from server to browser. | [realtime guide](learning/concepts/realtime-delivery.md) |
| **Trace / span** | A trace is the record of one operation across services; a span is one step in it. | [tracing guide](learning/concepts/distributed-tracing.md) |
| **SSRF** | Server-side request forgery: tricking a server into calling an internal address. | [security](cross-cutting/security.md) |
| **Learning choice** | A decision made mainly to learn a pattern; each ADR says what a pragmatic team would do instead. | [ADR index](adr/README.md) |

## Words to avoid

| Don't say | Say instead | Why |
|-----------|-------------|-----|
| alert | **page** (to a person) or **delivery** (one message) | "Alert" is ambiguous between the two |
| ping | **check** | A check is HTTP, not ICMP |
| site, endpoint | **monitor** (our record) or **target** (their server) | Keeps ours and theirs apart |
| ack (alone) | **acknowledge** (incident) or **broker ack** (message) | Two unrelated meanings |
| tenant, team, account | **organization** | One word |
| status (alone) | **monitor status**, **incident status**, **delivery status** | Three different enums |
| topic | **routing key** or **event type** | RabbitMQ has no topics in the Kafka sense |
