# ADR-0019: Push live updates with Server-Sent Events from a dedicated Realtime service

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (polling), 2 (SSE)

## Context

Users should see incidents open, get acknowledged and resolve without refreshing (US-07). Two decisions: the **transport** from server to browser, and **which component** holds the browser connections and how it learns about changes.

The traffic is one-directional (server → browser). Everything the user *does* is an ordinary REST call. The volume is low: a few events per minute per organization.

## Decision drivers

- Updates visible within 2 s.
- Domain services must not become frontend servers: Incident should not hold thousands of browser connections or know what a browser is.
- Must work through the gateway and with bearer-token auth ([ADR-0016](0016-browser-session.md)).
- A dropped connection must not leave the UI silently stale.
- Proportionate effort.

## Options considered

### Transport

| | Polling | Server-Sent Events (SSE) | WebSocket (raw or socket.io) |
|-|---------|--------------------------|------------------------------|
| How | Client asks every N seconds | One long-lived HTTP response; server writes `event:`/`data:` lines | Upgraded, two-way, message-based connection |
| Direction | Client pull | Server → client | Both ways |
| Latency | Up to N seconds | Immediate | Immediate |
| Through proxies | Trivial | Plain HTTP; needs response buffering off | Needs upgrade support in every hop |
| Reconnect | n/a | Built into the browser's `EventSource`; simple to write by hand | Hand-written, or socket.io's protocol |
| Auth | Normal headers | Normal headers if read with `fetch`; `EventSource` cannot set headers | No headers from the browser API; token in the first message or query string |
| Server cost | Requests even when nothing changes | One open response per client | One open socket per client; socket.io adds its own protocol, rooms, fallbacks |
| Nest support | Nothing needed | `@Sse()` returning an Observable | `@WebSocketGateway()` with the `ws` or socket.io adapter |
| When it is the right tool | Low urgency, simplest ops | Server-to-client notifications, feeds, progress | Chat, collaboration, games: frequent client → server messages |

### Who pushes, and how it learns

| Option | Pros | Cons |
|--------|------|------|
| A. Incident exposes the SSE endpoint itself | No new component | A domain service now holds long-lived browser connections, scales with open tabs instead of incidents, and must also relay monitor status. It becomes a frontend server |
| B. Gateway holds connections and consumes events | No new component | The gateway becomes stateful and gains a broker dependency; a gateway restart drops every stream; mixes edge routing with application fan-out |
| C. **A small Realtime service fed by events** (chosen) | Single purpose; owns no domain data; domain services only publish events they publish anyway; scales with connections, independently | An eighth process (~160 MB) |
| D. A BFF (backend-for-frontend) that also aggregates REST data for the UI | Would also remove client-side composition | The UI's REST needs are simple; a full BFF is more than is needed. Realtime is the push-only slice of one |

## Decision

**SSE**, from a dedicated **`realtime`** service (option C). Phase 1 uses **polling** (TanStack Query `refetchInterval: 5000`) so the UI works before the broker exists and so both approaches are experienced.

**How it learns.** Each realtime instance declares its own **exclusive, auto-delete** queue `realtime.events.<instanceId>` bound to `ms.events` for `incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1` and `monitor.state.changed.v1`. Every instance therefore receives every event (broadcast, not competing consumers), and the queue disappears with the instance. Messages are auto-acked: these are hints for whoever is connected right now.

**What it sends.** `GET /api/v1/realtime/stream` (`text/event-stream`). The service keeps a map of organization ID → open responses, and forwards a small **hint**, never the full resource:

```
event: incident.changed
id: 019a7c3e-…            (the envelope id)
data: {"incidentId":"…","status":"ACKNOWLEDGED","version":3}

event: monitor.status.changed
data: {"monitorId":"…","status":"DOWN","stateVersion":12}

: ping                     (comment line every 15 s, keeps proxies from closing the connection)
```

The browser reacts by invalidating the matching TanStack Query caches, which refetches through the normal REST API. So authorization, pagination and data shape stay in one place (the REST endpoints), and the stream cannot leak data the user could not otherwise read.

**Auth.** The stream is read with `fetch` + `ReadableStream` and parsed with `eventsource-parser`, so the request carries `Authorization: Bearer …` like every other call. Realtime verifies the JWT with the shared guard and takes the organization from the `org` claim. It ends the stream when the token's `exp` passes; the client reconnects with a fresh token.

**Reconnection and missed events.** There is **no replay buffer**. On every (re)connect the client invalidates the incident and monitor queries, so it converges on current state regardless of what it missed. Reconnect uses exponential backoff with jitter (1 s, 2 s, 4 s … capped at 30 s). Details: [frontend/realtime](../frontend/realtime.md).

**Gateway.** Proxies the path with buffering and the 10 s upstream timeout disabled for this route.

## Consequences

- Positive: domain services are unaware of browsers.
- Positive: the catch-up rule (refetch on connect) makes the system correct even though the push channel is best-effort.
- Positive: SSE is plain HTTP: visible in browser dev tools, testable with `curl -N`.
- Negative: one more service to run and deploy.
- Negative: reading SSE with `fetch` means writing reconnect logic by hand (about 60 lines), instead of getting it from `EventSource`.
- Negative: each hint costs a refetch; chatty under high event rates.
- Negative: each realtime instance holds its connections in memory; a restart drops them all at once and they reconnect together (jitter spreads the load).
- Harder later: client → server realtime features (typing indicators, presence) would need WebSockets.

## Revisit when

- Hints per organization exceed a few per second: send data in the event and patch the cache instead of refetching.
- A two-way feature appears: WebSockets.
- Thousands of concurrent connections: connection limits, HTTP/2, and sticky-free scaling need attention.

## Learning choice?

Partly. SSE is the pragmatic transport here. The **separate service** is partly a learning choice: a small team would poll every few seconds, or put the SSE endpoint in the gateway/BFF, until scale forced a split.

## How to explain it in an interview

> "Updates only flow from server to browser, so I used Server-Sent Events rather than WebSockets: it's plain HTTP, it goes through the gateway without upgrade handling, and reconnection is simple. I didn't want the incident service holding browser connections, so there's a small realtime service that subscribes to incident events from RabbitMQ with its own exclusive queue and pushes a tiny hint to connected clients of that organization. The browser then refetches through the normal REST API, so permissions stay in one place. Because the push channel is best-effort, the client refetches whenever it reconnects, which covers anything it missed. I started with plain polling in the first phase and that would honestly be fine for many products."
