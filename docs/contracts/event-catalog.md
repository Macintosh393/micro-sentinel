# Event catalog

The single source of truth for event names, producers, consumers and payloads. Machine-readable form: [asyncapi.yaml](asyncapi.yaml). Envelope, headers and compatibility rules: [versioning](versioning.md).

Every example below shows only the `data` object. The surrounding envelope (`id`, `type`, `source`, `time`, `subject`, `orgId`, `sequence`) is described once in [versioning](versioning.md#the-envelope). All events are published to the exchange `ms.events` with routing key = event type, through the producer's outbox.

## Index

| Event | Producer | Consumers (queue or stream) | `subject` | `sequence` |
|-------|----------|-----------------------------|-----------|------------|
| [`auth.org.created.v1`](#authorgcreatedv1) | auth | incident (`incident.auth-events`); archive | orgId | — |
| [`auth.user.changed.v1`](#authuserchangedv1) | auth | incident (`incident.auth-events`), notification (`notification.auth-events`); archive | userId | user version |
| [`auth.membership.changed.v1`](#authmembershipchangedv1) | auth | incident (`incident.auth-events`); archive | userId | membership version |
| [`monitor.definition.changed.v1`](#monitordefinitionchangedv1) | monitor | incident (`incident.monitor-events`), analytics (`ms.archive.lifecycle`) | monitorId | definitionVersion |
| [`monitor.check.completed.v1`](#monitorcheckcompletedv1) | monitor | analytics (`ms.archive.checks`) | monitorId | — |
| [`monitor.state.changed.v1`](#monitorstatechangedv1) | monitor | incident (`incident.monitor-events`), realtime; archive | monitorId | stateVersion |
| [`incident.opened.v1`](#incidentopenedv1) | incident | notification (`notification.incident-events`), realtime, analytics (`ms.archive.lifecycle`) | incidentId | incidentVersion |
| [`incident.acknowledged.v1`](#incidentacknowledgedv1) | incident | notification, realtime, analytics | incidentId | incidentVersion |
| [`incident.escalated.v1`](#incidentescalatedv1) | incident | notification, realtime, analytics | incidentId | incidentVersion |
| [`incident.resolved.v1`](#incidentresolvedv1) | incident | notification, realtime, analytics | incidentId | incidentVersion |
| [`incident.page.requested.v1`](#incidentpagerequestedv1) | incident | notification (`notification.incident-events`); archive | incidentId | — |
| [`notification.delivery.succeeded.v1`](#notificationdeliverysucceededv1) | notification | incident (`incident.notification-events`); archive | deliveryId | — |
| [`notification.delivery.failed.v1`](#notificationdeliveryfailedv1) | notification | incident (`incident.notification-events`); archive | deliveryId | — |

"Archive" means the event is also retained in the stream `ms.archive.lifecycle` with no service-specific consumer today. Internal command: [`notification.delivery.send.v1`](#internal-command-notificationdeliverysendv1).

All enums are **open**: consumers must tolerate unknown values ([versioning](versioning.md#consumer-obligations-tolerant-reader)).

---

## auth.org.created.v1

An organization was created (at registration). **Ordering:** none needed. **Consumer effect:** incident creates the default escalation policy (level 1 → owner, 300 s).

```json
{ "orgId": "019a7a00-1111-7aaa-8bbb-000000000042", "name": "Acme Ops",
  "ownerUserId": "019a7a00-2222-7aaa-8bbb-0000000000a1", "createdAt": "2026-11-15T09:00:00.000Z" }
```

## auth.user.changed.v1

**Snapshot** of a user's public profile after any change, including creation. Envelope `orgId` is `null`. **Ordering:** latest `version` wins; consumers upsert only if newer.

```json
{ "userId": "019a7a00-2222-7aaa-8bbb-0000000000a1", "email": "alice@acme.test",
  "displayName": "Alice", "active": true, "version": 3 }
```

## auth.membership.changed.v1

**Snapshot** of one user's membership in one organization. `role: null` means the user is no longer a member. **Ordering:** latest `version` wins.

```json
{ "orgId": "019a7a00-1111-7aaa-8bbb-000000000042", "userId": "019a7a00-2222-7aaa-8bbb-0000000000a1",
  "role": "ADMIN", "version": 2 }
```

`role`: `OWNER` | `ADMIN` | `MEMBER` | `null`.

## monitor.definition.changed.v1

**Snapshot** of a monitor's definition after create, edit, pause, resume or delete. **Ordering:** latest `definitionVersion` wins. **Consumer effects:** analytics upserts `monitors_dim`; incident resolves the monitor's unresolved incident when `deleted` is `true` (resolution `MONITOR_DELETED`).

```json
{ "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01", "name": "Checkout API",
  "url": "http://flaky-target:3000/site/checkout", "method": "GET", "intervalSeconds": 30,
  "paused": false, "deleted": false, "definitionVersion": 4 }
```

## monitor.check.completed.v1

One check finished. High volume (~17/s at design scale). **Ordering:** none; each is an independent fact keyed by `checkId`. `statusCode` and `latencyMs` are `null` when no response was received; `failureReason` is `null` on success.

```json
{ "checkId": "019a7c3e-4f00-7b3a-9c10-aa01bb02cc03", "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "checkedAt": "2026-11-15T10:42:07.100Z", "outcome": "FAILURE", "statusCode": 503,
  "latencyMs": 41, "failureReason": "UNEXPECTED_STATUS", "workerId": "monitor-worker-2" }
```

`outcome`: `SUCCESS` | `FAILURE`. `failureReason`: `TIMEOUT` | `DNS_ERROR` | `CONNECTION_ERROR` | `TLS_ERROR` | `UNEXPECTED_STATUS` | `BLOCKED_TARGET`.

## monitor.state.changed.v1

A monitor's **confirmed** status changed ([ADR-0009](../adr/0009-check-results-and-state-changes.md)). **Ordering: required per monitor.** `stateVersion` increases by 1 per change; consumers ignore a version ≤ the last one they processed for that monitor ([ADR-0017](../adr/0017-ordering-and-replay.md)). Carries the monitor's name and URL so consumers need no lookup. `cause` is set when `to` is `DOWN`, otherwise `null`.

```json
{ "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01", "monitorName": "Checkout API",
  "url": "http://flaky-target:3000/site/checkout", "from": "UP", "to": "DOWN",
  "stateVersion": 12, "changedAt": "2026-11-15T10:42:07.100Z",
  "cause": { "checkId": "019a7c3e-4f00-7b3a-9c10-aa01bb02cc03", "failureReason": "UNEXPECTED_STATUS",
             "statusCode": 503, "consecutiveFailures": 3 } }
```

`from`, `to`: `UNKNOWN` | `UP` | `DOWN` | `PAUSED`. **Consumer effects:** incident opens an incident on `to: DOWN`, resolves on `to: UP` (`RECOVERED`) or `to: PAUSED` (`MONITOR_PAUSED`), ignores `to: UNKNOWN`; realtime pushes `monitor.status.changed`.

## incident.opened.v1

An incident was opened. `incidentVersion` is 1.

```json
{ "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777", "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "monitorName": "Checkout API", "monitorUrl": "http://flaky-target:3000/site/checkout",
  "openedAt": "2026-11-15T10:42:07.900Z", "cause": { "failureReason": "UNEXPECTED_STATUS", "statusCode": 503 },
  "escalationLevel": 1, "incidentVersion": 1 }
```

## incident.acknowledged.v1

A user acknowledged the incident. **Consumer effect in notification:** cancels pending page deliveries for the incident and informs organization channels.

```json
{ "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777", "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "monitorName": "Checkout API", "acknowledgedAt": "2026-11-15T10:44:30.000Z",
  "acknowledgedBy": "019a7a00-2222-7aaa-8bbb-0000000000a1", "incidentVersion": 2 }
```

## incident.escalated.v1

The escalation timer moved an unacknowledged incident to the next level. A separate `incident.page.requested.v1` is published for each responder of the new level.

```json
{ "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777", "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "monitorName": "Checkout API", "fromLevel": 1, "toLevel": 2,
  "escalatedAt": "2026-11-15T10:47:08.000Z", "incidentVersion": 2 }
```

## incident.resolved.v1

The incident ended. `resolvedBy` is `null` unless `resolution` is `MANUAL`.

```json
{ "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777", "monitorId": "019a7b10-0c55-7d1e-8a11-5e0f2c9b7a01",
  "monitorName": "Checkout API", "resolvedAt": "2026-11-15T10:55:02.000Z",
  "resolution": "RECOVERED", "resolvedBy": null, "incidentVersion": 3 }
```

`resolution`: `RECOVERED` | `MANUAL` | `MONITOR_PAUSED` | `MONITOR_DELETED`.

**Ordering for all four `incident.*` lifecycle events:** not required by any consumer. Notification acts on each independently; analytics upserts each event's own columns; realtime sends hints. `incidentVersion` is provided for consumers that may need it later.

## incident.page.requested.v1

Incident decided that one specific user must be notified. One event per user per level. **Ordering:** none. Self-contained: notification needs no other data to send the page.

```json
{ "pageId": "019a7c40-2b3c-7d4e-9f50-000000000901", "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777",
  "userId": "019a7a00-2222-7aaa-8bbb-0000000000a1", "escalationLevel": 1, "reason": "OPENED",
  "requestedAt": "2026-11-15T10:42:07.900Z",
  "incident": { "monitorName": "Checkout API", "monitorUrl": "http://flaky-target:3000/site/checkout",
                "openedAt": "2026-11-15T10:42:07.900Z", "failureReason": "UNEXPECTED_STATUS" } }
```

`reason`: `OPENED` | `ESCALATED`.

## notification.delivery.succeeded.v1

One delivery was accepted by its receiver. `pageId` and `recipientUserId` are `null` for organization-channel deliveries. `targetLabel` is a masked, human-readable form of the destination, safe for the incident timeline. **Consumer effect:** incident appends `NOTIFICATION_SENT` to the timeline.

```json
{ "deliveryId": "019a7c40-3c4d-7e5f-8a60-000000000a01", "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777",
  "pageId": "019a7c40-2b3c-7d4e-9f50-000000000901", "recipientUserId": "019a7a00-2222-7aaa-8bbb-0000000000a1",
  "channelType": "EMAIL", "targetLabel": "a***@acme.test", "attempts": 1, "sentAt": "2026-11-15T10:42:09.200Z" }
```

`channelType`: `EMAIL` | `SMS` | `WEBHOOK` | `CHAT_WEBHOOK`.

## notification.delivery.failed.v1

One delivery was given up on (status `DEAD`) after its last attempt or a permanent error. **Consumer effect:** incident appends `NOTIFICATION_FAILED` to the timeline.

```json
{ "deliveryId": "019a7c40-3c4d-7e5f-8a60-000000000a02", "incidentId": "019a7c40-1a2b-7c3d-8e4f-000000000777",
  "pageId": "019a7c40-2b3c-7d4e-9f50-000000000901", "recipientUserId": "019a7a00-2222-7aaa-8bbb-0000000000a1",
  "channelType": "SMS", "targetLabel": "+3706*****12", "attempts": 5,
  "lastError": "HTTP 503 from SMS gateway", "failedAt": "2026-11-15T11:03:20.000Z" }
```

---

## Internal command: notification.delivery.send.v1

Not an event and not part of the public contract. Published by notification to its **own** direct exchange `notification.work` with routing key `delivery.send`, consumed by notification from `notification.delivery-jobs`. It uses the same envelope so the shared consumer wrapper, inbox and tracing apply. `subject` is the delivery ID.

```json
{ "deliveryId": "019a7c40-3c4d-7e5f-8a60-000000000a01" }
```

The command carries only the ID; the sender loads the delivery row, which is the source of truth for status and attempts ([notification](../services/notification.md)).

## Queues and their bindings

| Queue / stream | Type | Bound to `ms.events` with | Consumer | Prefetch |
|----------------|------|---------------------------|----------|----------|
| `incident.monitor-events` | quorum, single active consumer | `monitor.state.changed.v1`, `monitor.definition.changed.v1` | incident | 1 |
| `incident.auth-events` | quorum | `auth.org.created.v1`, `auth.user.changed.v1`, `auth.membership.changed.v1` | incident | 10 |
| `incident.notification-events` | quorum | `notification.delivery.succeeded.v1`, `notification.delivery.failed.v1` | incident | 10 |
| `notification.incident-events` | quorum | `incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1`, `incident.page.requested.v1` | notification | 10 |
| `notification.auth-events` | quorum | `auth.user.changed.v1` | notification | 10 |
| `notification.delivery-jobs` | quorum | (bound to `notification.work`, key `delivery.send`) | notification | 20 |
| `realtime.events.<instance>` | classic, exclusive, auto-delete | `incident.opened.v1`, `incident.acknowledged.v1`, `incident.escalated.v1`, `incident.resolved.v1`, `monitor.state.changed.v1` | realtime | 100, auto-ack |
| `ms.archive.checks` | stream | `monitor.check.completed.v1` | analytics | 500 |
| `ms.archive.lifecycle` | stream | `monitor.state.changed.v1`, `monitor.definition.changed.v1`, `incident.#`, `auth.#`, `notification.delivery.#` | analytics | 100 |

Retry and dead-letter queues for each work queue: [ADR-0023](../adr/0023-retry-topology-ttl-dlx.md).

## Which events analytics uses

From `ms.archive.checks` analytics projects `monitor.check.completed.v1`. From `ms.archive.lifecycle` it projects `monitor.definition.changed.v1` and the four `incident.*` lifecycle events, and skips the rest (`monitor.state.changed.v1`, `auth.*`, `incident.page.requested.v1`, `notification.delivery.*`) without error. Those are retained so that a future read model, or a service bootstrapping a replica, can replay them.
