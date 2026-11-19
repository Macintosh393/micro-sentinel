# Flow: notification retry and dead-letter queue

What happens when a delivery cannot be sent: delayed retries through wait queues, the final failure, parking in the DLQ, and a manual redrive. Also the poison-message case. Decisions: [ADR-0023](../adr/0023-retry-topology-ttl-dlx.md), [ADR-0013](../adr/0013-notification-owns-delivery-state.md). Service: [notification](../services/notification.md). Concepts: [retries, backoff and jitter](../learning/concepts/retries-backoff-jitter.md), [dead-letter queues](../learning/concepts/dead-letter-queues.md), [circuit breakers](../learning/concepts/circuit-breakers-and-bulkheads.md).

Queues involved (all quorum except wait queues, which are classic with a TTL):

| Queue | Role |
|-------|------|
| `notification.delivery-jobs` | Work queue; `x-delivery-limit=5`, DLX `ms.dlx` |
| `notification.delivery-jobs.retry.10s` / `.1m` / `.5m` / `.15m` | Wait queues: TTL, no consumer, dead-letter back to the work queue |
| `notification.delivery-jobs.dlq` | Parking; nothing consumes it |

## Happy path for comparison

`delivery.send` → lock row → attempt 1 → `200`/`250` → `SENT` → `notification.delivery.succeeded.v1`. One message, one attempt.

## Transient failure, then success

Setup: `curl -X PUT localhost:8090/control -d '{"target":"sms","mode":"flaky","failCount":2,"status":503}'`.

```mermaid
sequenceDiagram
  participant Q as delivery-jobs
  participant N as notification (Sender)
  participant DB as notification_db
  participant S as dev-stubs /sms
  participant R as ms.retry
  participant W10 as retry.10s
  participant W1 as retry.1m

  Q->>N: delivery.send {d1} (x-ms-attempt absent = 1)
  N->>DB: lock d1 · PENDING · attempts=1 · COMMIT
  N->>S: POST /sms/messages
  S-->>N: 503
  N->>N: ErrorClassifier: retryable
  N->>DB: last_error="HTTP 503"
  N->>R: publish copy, key …retry.10s, x-ms-attempt=2 (await confirm)
  N->>Q: broker-ack original
  R->>W10: route
  Note over W10: 10 s TTL expires → DLX "" → key notification.delivery-jobs
  W10->>Q: message returns
  Q->>N: delivery.send {d1} attempt 2
  N->>S: POST → 503
  N->>R: copy → retry.1m, attempt=3; ack
  Note over W1: 60 s
  W1->>Q: returns
  Q->>N: attempt 3
  N->>S: POST → 202
  N->>DB: SENT · outbox(delivery.succeeded) · COMMIT
  N->>Q: broker-ack
```

The delivery row shows `attempts = 3`, `status = SENT`; the incident timeline gets one `NOTIFICATION_SENT` entry.

## Final failure → DEAD → DLQ

Setup: SMS stub in mode `fail` (always `503`).

| Attempt | When (cumulative) | Outcome | Action |
|---------|-------------------|---------|--------|
| 1 | 0 | 503 | → `retry.10s` |
| 2 | 10 s | 503 | → `retry.1m` |
| 3 | 1 min 10 s | 503 | → `retry.5m` |
| 4 | 6 min 10 s | 503 | → `retry.15m` |
| 5 | 21 min 10 s | 503 | `status = DEAD` · outbox `notification.delivery.failed.v1` · COMMIT · then `nack(requeue=false)` → `ms.dlx` → `notification.delivery-jobs.dlq` |

```mermaid
sequenceDiagram
  participant Q as delivery-jobs
  participant N as notification
  participant DB as notification_db
  participant DLX as ms.dlx
  participant DLQ as delivery-jobs.dlq
  participant I as incident

  Q->>N: attempt 5
  N->>N: send → 503; attempts exhausted
  N->>DB: tx: status DEAD, last_error · outbox(delivery.failed)
  N->>DLX: nack(requeue=false) with x-ms-last-error header
  DLX->>DLQ: parked
  N-->>I: (via outbox + broker) notification.delivery.failed.v1
  I->>I: timeline NOTIFICATION_FAILED "SMS to +3706*****12: HTTP 503 after 5 attempts"
```

Order matters: the database row is marked `DEAD` and the failure event is in the outbox **before** the message is dead-lettered. If the process crashed in between, the redelivered message would find `DEAD` and be acked: the record is the truth, the DLQ is the audit trail.

While this happens, other channels for the same page (Alice's email) are unaffected: each channel is its own delivery and its own message.

### Circuit breaker during a long outage of one provider

After 5 consecutive failures to `dev-stubs` SMS, the breaker for key `SMS:dev-stubs` opens. For the next 30 s, SMS sends fail instantly (`retryable`, error `circuit open`) instead of each waiting up to 5 s. This protects the **sender's** concurrency (prefetch 20) from being consumed by slow failures, so email deliveries keep flowing. After 30 s one trial request is allowed (half-open); success closes the breaker. Attempts still count, so a long outage still ends in `DEAD`.

## Permanent failure: straight to DLQ

| Cause | Classifier result | Effect |
|-------|-------------------|--------|
| Webhook returns `400` / `404` / `410` | permanent | `DEAD` after **1** attempt |
| Webhook URL now resolves to a private IP | permanent (`BLOCKED_TARGET`) | `DEAD` after 1 attempt |
| Contact method was deleted after planning | permanent | `DEAD`; error "target no longer exists" |
| `429` with `Retry-After` | retryable | Next tier as normal (the header is logged; honouring arbitrary delays would need a DB schedule, [ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)) |

## Poison message

A **poison message** fails every time, often before the handler's error handling runs.

| Variant | What stops it |
|---------|---------------|
| Envelope is not valid JSON, or payload fails validation against the `contracts` class | Consumer wrapper → `PermanentError` → immediate `nack(requeue=false)` → DLQ with `x-ms-last-error: "validation: deliveryId must be a UUID"` |
| Handler throws on this one message every time (a bug) | Treated as retryable: goes through the 4 wait tiers, then DLQ after the 5th attempt |
| Message crashes the process (out of memory, `process.exit`, native crash) | No handler runs, no republish. Broker redelivers on reconnect; the quorum queue counts deliveries; after `x-delivery-limit` (5) it dead-letters automatically |

The third row is why the delivery limit exists: it is the only protection that works when your own code never gets to run.

## Manual redrive

After fixing the cause (`PUT /control {"target":"sms","mode":"ok"}`):

**From the UI** (`/deliveries?status=DEAD` → Retry): `POST /api/v1/deliveries/{id}/retry` → in one transaction: `status = PENDING`, `attempts = 0`, outbox row with a new `notification.delivery.send.v1` → `202`. The old message in the DLQ is now obsolete; if it is ever redriven, the sender finds the row already `SENT` and acks.

**From the CLI** (for any DLQ): `pnpm dlq:redrive notification.delivery-jobs` moves messages from `.dlq` back to the work queue, resetting `x-ms-attempt`. For delivery jobs, prefer the UI path: it resets the database record, which the CLI does not.

**Redrive is never automatic.** A DLQ that drains itself on a timer is a slow retry loop, not a DLQ.

## Failure variants of the retry mechanism itself

| # | What goes wrong | Result |
|---|-----------------|--------|
| R1 | Crash after publishing the retry copy, before acking the original | Original redelivered *and* copy returns later: two jobs for one delivery. The row lock serialises them; whichever runs second finds `SENT` and acks, or (if both fail) one extra attempt is counted. The **delivery row's status is the idempotency guard** for this queue, not the inbox (see below) |
| R2 | Broker restarts while messages sit in wait queues | Wait queues are durable and messages persistent: they survive; TTL continues from restart |
| R3 | Wait queue declared with different TTL than before | `PRECONDITION_FAILED` at startup (queue arguments are immutable). Tiers are named by delay, so a new tier is a new queue |
| R4 | DLQ grows unbounded | Alert at depth > 0; dashboard panel. No auto-expiry in v1 |

### Why `notification.delivery-jobs` does not use the inbox

Every retry of a job reuses the same envelope `id`. The sender commits its attempt counter in a transaction *before* sending; if that transaction also inserted the inbox row, the retry copy would be rejected as a duplicate and the delivery would never be retried. So this one consumer subscribes with `inbox: false`, and the `deliveries` row (status `PENDING` / `SENT` / `DEAD` / `CANCELLED`, plus the row lock) does the job the inbox does elsewhere. It is a good example of the general rule: the idempotency record must be whatever the handler's effect is keyed on ([ADR-0012](../adr/0012-idempotent-consumers.md)).

## Observability

- Metrics: `notification.deliveries{status}`, `notification.attempts` histogram, `rabbitmq_queue_messages{queue=~".*retry.*|.*dlq"}` (from the RabbitMQ Prometheus plugin), `notification.circuit.state`.
- RabbitMQ management UI: watch messages move `delivery-jobs` → `retry.10s` → back → … → `dlq`. Inspect a DLQ message's headers: `x-ms-attempt: 5`, `x-ms-last-error`, `x-death`.

## Expected trace

Each attempt is its own consumer span in the **same trace**, because the retry copy carries the original `traceparent`:

```
notification  notification.delivery-jobs process  (attempt 1)  ~5 ms
└─ notification.send → HTTP POST dev-stubs/sms → 503
   └─ notification  ms.retry publish (retry.10s)
… 10 s
notification  notification.delivery-jobs process  (attempt 2, link to attempt 1)
└─ notification.send → 503 → ms.retry publish (retry.1m)
… 60 s
notification  notification.delivery-jobs process  (attempt 3)
└─ notification.send → 202
   └─ pg UPDATE deliveries / INSERT outbox
```

Span attributes: `messaging.message.id` (constant), `ms.attempt`, `ms.retry_tier`, `ms.outcome` (`sent`, `retry`, `dead`).

## Demo script

1. `PUT /control {"target":"sms","mode":"fail"}` on dev-stubs. Use `RETRY_TIERS_SCALE=0.1` (dev only, [configuration](../cross-cutting/configuration.md)) so the schedule runs in ~2 minutes instead of 21.
2. Trigger an outage. Show the email arriving immediately while SMS retries.
3. In the RabbitMQ UI, watch the message hop between wait queues.
4. Delivery becomes `DEAD`; the incident timeline says so; the DLQ has one message with its last error.
5. Switch SMS to `ok`, click Retry on `/deliveries`, see it in the dev-stubs SMS inbox.
