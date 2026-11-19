# ADR-0013: Give Notification its own database for delivery state

- Status: Accepted
- Date: 2026-11-15
- Phase: 2

## Context

The brief describes Notification as "Stateless + DLQ". A truly stateless notifier consumes an event, sends, and acks. That cannot meet the requirements:

| Need | State it requires |
|------|-------------------|
| Do not send the same page twice when a message is redelivered (FR-N3) | Which deliveries already exist: a **dedupe key** |
| Retry with increasing delay, give up after 5 attempts (FR-N4) | An attempt count per delivery |
| Show admins what was sent, what failed and why; allow manual retry (US-14) | A delivery record with status and last error |
| Know a user's email address, phone number and enabled channels | Contact methods and a copy of the user's email |
| Know an organization's webhook and chat destinations | Channel configuration |
| Escalate if nobody acknowledges | A timer |

The question is where each piece of state lives.

## Decision drivers

- A duplicate page is one of the two failures this product must not have (the other is a missed page).
- State that changes together should be stored together and committed in one transaction.
- Notification should answer "*how* do we reach them and did it work"; it should not decide "*who* must be told".

## Options considered

### A. Truly stateless: dedupe and attempts carried in message headers
- Pro: no database.
- Con: redelivered duplicates carry the same headers, so nothing distinguishes "second copy" from "first copy"; there is no record for the UI; contact data has to come from somewhere at send time (a synchronous call, which was just rejected in [ADR-0014](0014-on-call-in-incident-and-local-replicas.md)).

### B. Dedupe keys in Redis, everything else stateless
- Pro: small; fast.
- Con: the dedupe mark is not atomic with anything ([ADR-0012](0012-idempotent-consumers.md), option D); Redis in this stack has no persistence, so a restart forgets what was sent; still no delivery history.

### C. Incident owns delivery state; Notification is a stateless sender
- Pro: one fewer database.
- Con: Incident would need to know about channels, SMTP errors and retry schedules: delivery concerns leaking into the incident model. Every send result becomes a write to another service's data.

### D. Notification owns a database with contact methods, channels and deliveries (chosen)
- Pro: the dedupe key is a unique constraint; attempts, status and errors are columns; creation of deliveries is transactional with the inbox row; the delivery log is a table the API can list.
- Con: Notification is now a stateful service with migrations, an outbox and a replica to maintain.

### Escalation timers: who owns them?

| Option | Verdict |
|--------|---------|
| Notification ("if no ack in 5 minutes, send to the next person") | Rejected. It would need to know incident status and the escalation policy, i.e. Incident's model |
| A delayed message in the broker | Rejected. A queued timer cannot be cancelled when the incident is acknowledged, and long TTLs in queues are fragile |
| **Incident, as a `next_escalation_at` column polled with `SKIP LOCKED`** | Chosen. The timer is part of the incident's state and is cancelled by the same transaction that acknowledges it ([incident](../services/incident.md)) |

## Decision

Option D, with a clear split of responsibility:

- **Incident decides who.** It publishes `incident.page.requested.v1` naming one user, and the lifecycle events for organization channels.
- **Notification decides how, and records what happened.** It owns `notification_db` ([schema](../data/notification-schema.md)): `recipients` (replica of user email and name), `contact_methods`, `org_channels`, `deliveries`, plus `outbox` and `processed_messages`.

**Two stages inside Notification**, connected by its own work queue:

1. **Plan** (consumer of `notification.incident-events`). In one transaction: inbox insert; look up contact methods or channels; insert one `deliveries` row per target with `ON CONFLICT (dedupe_key) DO NOTHING`; insert one outbox row per *new* delivery carrying the command `notification.delivery.send.v1`.
2. **Send** (consumer of `notification.delivery-jobs`). Load the delivery; if it is already `SENT` or `DEAD`, ack and stop. Otherwise attempt the send with a 5 s timeout, then record the outcome and publish `notification.delivery.succeeded.v1` or, after the last attempt, `notification.delivery.failed.v1`. Retry and dead-letter routing: [ADR-0023](0023-retry-topology-ttl-dlx.md).

**Dedupe key:** `<source event id>:<channel type>:<target id>`, where target id is the contact method or organization channel ID. The same incoming event, redelivered any number of times, maps to the same keys and creates no new rows.

**The unavoidable gap.** Sending (SMTP or HTTP) and recording `SENT` are two systems. If the process dies after the receiver accepted the message but before the row is updated, the job is redelivered and the message is sent again. Mitigations, in order:

- The row is locked (`SELECT … FOR UPDATE`) while sending, so two consumers never send the same delivery concurrently.
- Webhooks carry `Idempotency-Key: <delivery id>` so a well-behaved receiver can discard the repeat; emails carry the delivery ID in the `Message-ID` header.
- The remaining risk, one duplicate message after a crash at exactly the wrong moment, is accepted: for a paging system, at-least-once is the right side to err on.

## Consequences

- Positive: duplicate suppression is a database constraint, not a hope.
- Positive: the delivery log, failure reasons and manual redrive come almost for free.
- Positive: Notification works while auth and incident are down, from its replica and its queues.
- Negative: the brief's "stateless" property is gone; Notification needs migrations, an outbox and retention like every other stateful service.
- Negative: the plan/send split adds a queue and a message type.
- Harder later: horizontal scaling of the send stage is easy (competing consumers; rows are locked per delivery). Rate limiting per provider would need shared counters.

## Revisit when

- Delivery volume makes the `deliveries` table large enough to need partitioning or archiving (hundreds of thousands of rows).
- A provider requires strict rate limits: add a token bucket per channel type.

## Learning choice?

No. Real notification systems keep exactly this state. "Stateless" in the brief was the idea that did not survive scrutiny.

## How to explain it in an interview

> "The original idea was a stateless notifier, but you can't avoid duplicate pages without remembering what you've sent. So notification has its own database. For each incoming event it creates delivery rows with a dedupe key built from the event ID, channel and target, under a unique constraint, in the same transaction as the processed-message record. A redelivered event just hits the constraint. A second stage does the actual send with retries. The one gap I can't close is between the email server accepting the message and me recording that: if I crash there I'll resend once. I pass the delivery ID as an idempotency key and accept that, because for paging, twice is better than never."
