# realtime

**Purpose:** tell connected browsers that something in their organization changed, so the UI can refetch. **Bounded context:** none. It owns no domain data and makes no decisions.

This service is not in the original brief. It exists so that no domain service has to hold browser connections. Decision record: [ADR-0019](../adr/0019-sse-realtime-service.md). Concept: [real-time delivery](../learning/concepts/realtime-delivery.md). Browser side: [frontend/realtime](../frontend/realtime.md).

## Responsibilities

- Accept authenticated SSE connections and keep them open.
- Consume incident and monitor status events from the broker.
- Forward a small hint to every open connection of the event's organization.
- Send heartbeats; close connections when their token expires.

## Explicitly not responsible for

- **Delivering data.** Hints carry IDs and a status; the browser reads the real data from the REST APIs.
- **Guaranteed delivery or replay.** A hint missed during a disconnect is gone. Correctness comes from the client refetching on reconnect.
- **Authorization of resources.** It filters by organization only; the REST refetch enforces everything else.
- **Client → server messages.** User actions are REST calls.
- **Being a BFF.** It aggregates nothing.

## Owned data

None persistent. In memory: `Map<orgId, Set<Connection>>`.

## Sync APIs

**Exposed** ([realtime.yaml](../contracts/openapi/realtime.yaml)): `GET /v1/realtime/stream` (`text/event-stream`).

SSE events sent:

| `event:` | `data` | Sent when |
|----------|--------|-----------|
| `ready` | `{ connectionId, heartbeatSeconds }` | Connection accepted |
| `incident.changed` | `{ incidentId, status, version, cause }` | Any of the four incident lifecycle events |
| `monitor.status.changed` | `{ monitorId, status, stateVersion }` | `monitor.state.changed.v1` |
| *(comment)* `: ping` | — | Every 15 s |

**Consumed:** JWKS from auth (cached).

## Events

**Consumed** ([catalog](../contracts/event-catalog.md)), from its own queue `realtime.events.<instanceId>` (classic, exclusive, auto-delete, non-durable, auto-ack):

`incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1`, `monitor.state.changed.v1`.

**Published:** none.

Why an exclusive queue per instance instead of one shared queue: with a shared queue, replicas would be competing consumers and each event would reach only one replica, so only the browsers connected to that replica would hear about it. Each instance having its own queue bound to the same routing keys turns the topic exchange into a **broadcast**.

## Internal components

| Component | Job |
|-----------|-----|
| `StreamController` | `GET /v1/realtime/stream`: sets headers (`Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`), registers the connection, sends `ready`, schedules closure at token `exp` |
| `ConnectionRegistry` | The org → connections map; add, remove on `close`, count; enforces 5 streams per user and a global cap (`REALTIME_MAX_CONNECTIONS`, default 500) |
| `EventFanout` | Broker consumer: envelope → hint → write to each connection of `envelope.orgId` |
| `Heartbeat` | Every 15 s writes `: ping` to all connections; a failed write removes the connection |
| `SlowClientGuard` | If a connection's outgoing buffer exceeds 64 KB (the client is not reading), close it. A slow browser must not grow the server's memory |

The mapping from event to hint is the only "logic": drop `data` except IDs, status and version.

## Scaling model

Scales with **open connections**, not with events. Each replica holds its own connections in memory and receives every event through its own queue. Replicas need no coordination and no sticky sessions (a reconnect may land anywhere). One Node process comfortably holds a few thousand idle SSE connections; the local cap is 500. Locally: one replica.

A restart drops all of that replica's connections at once; clients reconnect with jittered backoff so they do not return in one wave ([retries, backoff and jitter](../learning/concepts/retries-backoff-jitter.md)).

## Failure modes

| Dependency down | Effect |
|-----------------|--------|
| RabbitMQ | Connections stay open and receive heartbeats but no hints. Readiness goes false; on broker return the queue is re-declared and hints resume. Hints during the gap are lost; **clients are not told**, which is why the client also refetches on a slow timer (60 s) as a safety net |
| auth | Existing connections continue until token expiry; reconnects need a refreshed token, which needs auth |
| gateway | All streams drop; browsers retry |

**When realtime is down:** the SSE request fails; the web app shows "Live updates paused" and switches its queries back to 5-second polling until the stream reconnects. Nothing else in the system notices: no domain service depends on realtime.

## Observability

| Kind | Items |
|------|-------|
| Metrics | `realtime.connections` (gauge) by org count; `realtime.connections.opened` / `.closed` by reason (`client`, `token_expired`, `slow_client`, `shutdown`, `limit`); `realtime.hints.sent`; `realtime.events.without_listeners`; `realtime.fanout.duration` |
| Spans | `realtime.events process` per event (child of the publish span), with attribute `connections_notified`. The long-lived HTTP stream is **not** traced as one span |
| Log fields | `connection_id`, `user_id`, `org_id`, `reason`, `duration_s` |

## Patterns applied

[Real-time delivery](../learning/concepts/realtime-delivery.md) · [API gateway and BFF](../learning/concepts/api-gateway-and-bff.md) (why this is not in the gateway) · [backpressure](../learning/concepts/backpressure.md) (slow-client guard) · [event-driven architecture](../learning/concepts/event-driven-architecture.md) (pub/sub fan-out) · [health checks](../learning/concepts/health-checks.md)

## Framework and Nest structure

**NestJS 12.** Small, but it reuses the JWT guard, the messaging module and lifecycle hooks.

| Nest element | Use |
|--------------|-----|
| Modules | `StreamModule`, `FanoutModule`, `HealthModule`; shared `MessagingModule` (consume only), `AuthGuardModule`, `ObservabilityModule` |
| Controller | Writes to the raw Express response (`@Res()`) rather than using Nest's `@Sse()` decorator: full control over headers, heartbeats, back-pressure checks and closing on token expiry. `@Sse()` with an RxJS `Observable` is the idiomatic alternative and is worth trying once |
| Guards | `JwtAuthGuard` (the token's `exp` is also used to schedule closure) |
| Pipes / interceptors | None on the stream route (an interceptor that wraps the response would break streaming) |
| Exception filter | `ProblemDetailsFilter` for failures **before** the stream starts; after headers are sent, errors just close the stream |
| Lifecycle | `OnApplicationShutdown`: stop accepting, end every stream, close the broker channel |

## Phase introduced

Phase 2. (Phase 1 uses polling only.)

## Related

Frontend: [realtime](../frontend/realtime.md). Flows: [outage-to-resolution](../flows/outage-to-resolution.md).
