# Versioning, envelope and schema evolution

How events are shaped, how they change without breaking consumers, and how HTTP APIs are versioned. Background: [schema evolution and versioning](../learning/concepts/schema-evolution.md).

## The envelope

Every message published to `ms.events` has the same outer structure, the **envelope**. The event-specific content is in `data`.

```json
{
  "id": "019a7c3e-5b1a-7c2e-9f00-3d2b8a4e6f10",
  "type": "monitor.state.changed.v1",
  "source": "monitor",
  "time": "2026-11-15T10:42:07.512Z",
  "subject": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "orgId": "019a7a00-1111-7aaa-8bbb-000000000042",
  "sequence": 12,
  "data": { }
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `id` | UUIDv7 | yes | Unique per event. Assigned when the outbox row is written; identical on every republish. **The idempotency key** for consumers |
| `type` | string | yes | Event type including major version. Equals the AMQP routing key |
| `source` | string | yes | Publishing service: `auth`, `monitor`, `incident`, `notification` |
| `time` | RFC 3339 UTC, milliseconds | yes | When the fact happened (not when it was published) |
| `subject` | UUID | yes | ID of the aggregate the event is about: the **ordering key** |
| `orgId` | UUID or `null` | yes | Tenant. `null` only for events not scoped to one organization (`auth.user.changed.v1`) |
| `sequence` | integer or `null` | yes | Per-`subject` version, increasing. `null` for events with no ordering requirement |
| `data` | object | yes | Payload; schema depends on `type` |

The shape is modelled on CloudEvents (`id`, `type`, `source`, `time`, `subject`) without claiming compliance: there is no `specversion`, and `orgId` and `sequence` are our own.

TypeScript definition in `@microsentinel/contracts`:

```ts
export interface Envelope<TType extends string = string, TData = unknown> {
  id: string;
  type: TType;
  source: 'auth' | 'monitor' | 'incident' | 'notification';
  time: string;
  subject: string;
  orgId: string | null;
  sequence: number | null;
  data: TData;
}
```

## AMQP properties and headers

The envelope is the message **body** (UTF-8 JSON). Transport metadata goes in AMQP properties so that tooling and the broker can use it without parsing the body.

| AMQP property | Value |
|---------------|-------|
| `contentType` | `application/json` |
| `contentEncoding` | `utf-8` |
| `deliveryMode` | `2` (persistent) |
| `messageId` | envelope `id` |
| `type` | envelope `type` |
| `appId` | envelope `source` |
| `timestamp` | publish time, seconds |

| Header | Set by | Meaning |
|--------|--------|---------|
| `traceparent` | outbox relay | W3C Trace Context of the publish span ([observability](../cross-cutting/observability.md)) |
| `tracestate` | outbox relay | W3C Trace Context, vendor state; forwarded unchanged |
| `x-ms-attempt` | consumer, on retry | Attempt number, starting at 1 (absent = 1) |
| `x-ms-last-error` | consumer, on retry or dead-letter | Short error text, max 500 characters |
| `x-death` | broker | Dead-lettering history |
| `x-stream-offset` | broker | Offset of a message read from a stream |

Trace context is deliberately **not** in the body: it describes the delivery, not the fact.

## Naming

| Thing | Rule | Example |
|-------|------|---------|
| Event | `<service>.<entity>.<past-tense verb>.v<major>` | `incident.resolved.v1` |
| Command | `<service>.<entity>.<imperative>.v<major>` | `notification.delivery.send.v1` |
| Routing key | Equals the event type | `incident.resolved.v1` |
| Payload fields | `camelCase`; timestamps end in `At`; IDs end in `Id` | `acknowledgedAt`, `monitorId` |
| Enum values | `UPPER_SNAKE` | `MONITOR_PAUSED` |

## Compatibility rules for events

A change is **backward compatible** if consumers written against the old schema keep working with new messages. Producers and consumers are deployed at different times, and messages written by old code sit in queues and streams for a long time (up to 5 years in `ms.archive.lifecycle`), so compatibility is not optional.

### Allowed within a version (no rename needed)

- Add an **optional** field to `data`.
- Add a new enum value, **only** if the event's catalog entry marks the enum as open (all of ours are).
- Add a new event type.
- Widen a numeric range or lengthen a string limit.

### Not allowed within a version (requires a new major: `…v2`)

- Remove or rename a field.
- Change a field's type or unit (`latencyMs` → seconds).
- Make an optional field required.
- Change what a field or the event **means**, even if the shape is identical.
- Change what `subject` or `sequence` refer to.

### Consumer obligations: tolerant reader

1. **Ignore unknown fields.** Validation uses `whitelist: true` and never `forbidNonWhitelisted` ([ADR-0005](../adr/0005-typeorm-and-class-validator-usage.md)).
2. **Validate only what you use.** A consumer's check should fail for a missing field it needs, not for one it never reads.
3. **Handle unknown enum values deliberately.** Either a defined fallback or a `PermanentError` that dead-letters the message with a clear reason. Never an unhandled `default:`.
4. **Ignore unknown event types** that reach the queue through a wildcard binding: broker-ack and count them in a metric.

### Producer obligations

1. Never emit a field that is absent from the catalog entry.
2. Emit `null` rather than omitting a documented nullable field.
3. Publish old and new versions side by side during a migration.

## Introducing a breaking change: the procedure

Example: `monitor.state.changed.v1` must change `cause` from an object to an array.

1. **Add** `monitor.state.changed.v2` to the catalog, AsyncAPI and `contracts`.
2. **Producer dual-publishes**: both `v1` and `v2` are written to the outbox in the same transaction. They have different envelope `id`s.
3. **Consumers migrate** one at a time: bind `v2`, handle it, unbind `v1`. A consumer must not process both versions of the same fact; binding one at a time ensures that.
4. When queue bindings show no consumer of `v1`, the **producer stops** publishing it.
5. `v1` stays in the catalog marked *deprecated* for as long as it exists in a stream. Any consumer that replays from a stream must still be able to read it (an **upcaster**: a function that converts an old shape to the current one on read).

This is the **expand → migrate → contract** sequence, the same one used for database changes.

In a monorepo all of this could be a single commit. It is still done in steps, because the habit is the point ([ADR-0022](../adr/0022-contracts-package-and-shared-libs.md)).

## Keeping the three sources in step

| Source | Role |
|--------|------|
| [event-catalog.md](event-catalog.md) | For people: meaning, producer, consumers, ordering, example |
| [asyncapi.yaml](asyncapi.yaml) | For machines: JSON Schema per message |
| `@microsentinel/contracts` | For code: classes and constants |

A test in local CI parses every JSON example in the catalog and validates it against both the AsyncAPI schema and the class in `contracts`. A second test checks that every event type constant in `contracts` has a catalog entry and vice versa.

## HTTP API versioning

- Version in the path: `/api/v1/…`. One major version at a time in this project.
- Within `v1`: adding endpoints, optional request fields and response fields is allowed. Clients must ignore unknown response fields (the generated TypeScript client does).
- Removing or changing anything requires `/api/v2/…` served side by side.
- Errors use RFC 9457 problem details (`application/problem+json`) with a stable machine-readable `code`:

```json
{
  "type": "https://microsentinel.local/problems/incident-already-resolved",
  "title": "Incident already resolved",
  "status": 409,
  "code": "INCIDENT_ALREADY_RESOLVED",
  "detail": "Incident 019a7c40-… was resolved at 2026-11-15T10:55:02Z.",
  "instance": "/v1/incidents/019a7c40-…/acknowledge",
  "requestId": "0f8c2a5e-…"
}
```

  Validation failures (`400`, code `VALIDATION_FAILED`) add `errors: [{ "field": "intervalSeconds", "message": "must not be less than 30" }]`. Clients branch on `code`, never on `title` or `detail`. The `type` URI is an identifier, not a page that must exist.
- The OpenAPI files in [openapi/](openapi/) are the contract. Each service serves its own document in development, and a test fails if the served document and the committed file differ.

## SSE event versioning

The realtime stream's event names (`incident.changed`, `monitor.status.changed`) follow the same additive rule: new event names and new fields may appear; clients ignore what they do not recognise ([realtime](../frontend/realtime.md)).
