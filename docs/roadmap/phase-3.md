# Phase 3: Resilience and scale

**Effort:** 35–45 h (4 weeks). Overview: [phases](phases.md).

## Goal

Run several workers and prove they coordinate; make failure handling complete (delayed retries, DLQ, redrive, circuit breaker); finish incident response (schedules, multi-level escalation); and give admins the screens to manage it.

## Scope

**In:**

| Area | Content |
|------|---------|
| monitor-worker | 2 replicas by default; per-host concurrency cap; full SSRF classifier and tests; event-loop delay metric and readiness rule; graceful shutdown releasing leases |
| `messaging` | Full retry tiers per queue kind ([ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)); quorum delivery limit; `RETRY_TIERS_SCALE`; `pnpm dlq:redrive` |
| incident | Schedules CRUD + on-call calculation; escalation policy editor with validation against `responders`; escalation timer (`SKIP LOCKED`); `incident.escalated.v1`; `NO_RESPONDER_AVAILABLE` |
| notification | SMS and chat adapters; contact-methods API; circuit breaker; delivery log API; manual redrive; delivery-jobs consumer with `inbox: false` |
| auth | Members (add, role, remove), API keys (create, list, revoke, token exchange) |
| toxiproxy | `chaos` profile with proxies for postgres, rabbitmq, dev-stubs |
| `flaky-target` | `error` and `huge-body` modes |

**Deferred:** analytics, tracing SDK, contract tests, Playwright (Phase 4).

**Frontend slice:** `/on-call` (schedules with who is on call now), schedule form, `/on-call/escalation` editor; `/settings/notifications` (contact methods); `/settings/members`; `/settings/api-keys` (key shown once); `/deliveries` log with status filter and Retry on `DEAD`; deliveries section on incident detail.

## Tasks (one per sitting)

1. Scale workers to 2; write the "no double application" integration test with real concurrency; add `monitor.lease.fenced` and schedule-lag metrics (as log lines until Phase 4).
2. Graceful shutdown in workers (stop claiming, drain, release); `stop_grace_period`.
3. `safe-fetch`: full classifier (IPv6, mapped, numeric forms, multiple A records), per-host cap; event-loop delay readiness.
4. `messaging`: retry tiers as topology config; scaled tiers; integration tests for each tier and for the delivery limit (child-process crash).
5. `scripts/dlq-redrive.mjs`.
6. incident: schedules tables, CRUD, `OnCallCalculator` + tests, `GET …/on-call`.
7. incident: policy editor (`PUT`), validation, `UNKNOWN_RESPONDER`.
8. incident: escalation timer, `incident.escalated.v1`, pages for the new level; tests for the ack-vs-timer race.
9. notification: SMS + chat adapters against dev-stubs; contact-methods API (E.164 validation, ownership).
10. notification: opossum breaker per channel + host; error classifier tests.
11. notification: delivery log API with keyset pagination; redrive endpoint.
12. auth: members API (add with temporary password, change role, remove + revoke refresh tokens); events.
13. auth: API keys and `POST /v1/auth/token`.
14. web: on-call screens.
15. web: contact methods, members, API keys screens.
16. web: deliveries log and redrive; deliveries on incident detail.
17. Toxiproxy wiring; break-it experiments 4–8, 10; fix findings. Measure memory. Record the demo.

## Definition of Done

| # | Check | Expected |
|---|-------|----------|
| 1 | `pnpm ci:local` | Exit 0 including retry, DLQ, delivery-limit, fencing and escalation tests |
| 2 | `docker compose up -d --wait --scale monitor-worker=3` | Three healthy workers |
| 3 | 20 monitors at 10 s for 5 min, then in monitor_db: `select monitor_id, count(*) from monitor_checks where checked_at > now()-interval '5 min' group by 1` | Each ≈ 30 (±2). No monitor at ~60 or ~90 |
| 4 | Same window: `select worker_id, count(*) from monitor_checks … group by 1` | Work spread across all three workers |
| 5 | `docker pause` one worker for 70 s during a `timeout`-mode outage, then `docker unpause` | Its monitors are checked by others within ~60 s; fenced completions logged ≥ 1; no counter jumps (`DOWN` needs 3 real failures) |
| 6 | `docker compose kill monitor-worker` (one replica) | Its leases are picked up within 60 s; no check gap > 70 s for any monitor |
| 7 | dev-stubs SMS `fail`, `RETRY_TIERS_SCALE=0.1`, outage | Delivery attempts 1–5 at ~0, 1 s, 7 s, 37 s, 2 min 7 s; status `DEAD`; one message in `notification.delivery-jobs.dlq`; timeline shows `NOTIFICATION_FAILED` |
| 8 | Switch SMS `ok`, click Retry on `/deliveries` | Delivery `SENT`; message in dev-stubs SMS inbox |
| 9 | No acknowledgement for the level-1 delay (set 60 s for the test) | `incident.escalated.v1`; level-2 responder gets an email; timeline `ESCALATED` |
| 10 | Acknowledge before the delay | No escalation; `next_escalation_at` null |
| 11 | dev-stubs SMS `slow` (8 s) with 30 pages | Email deliveries keep arriving promptly; breaker opens for SMS (log line), closes after recovery |
| 12 | API key: create → `POST /api/v1/auth/token` → `GET /api/v1/monitors` with the token | `200`; revoke → token exchange `401` |
| 13 | MEMBER calls `PUT /api/v1/escalation-policy` | `403` |
| 14 | `docker stats --no-stream` | Core + chaos < 3.5 GB |

## Demo script (4 minutes)

1. Three workers; show the even spread query (DoD #4).
2. `docker pause` a worker mid-check; unpause; point at the fenced log line: "it woke up thinking it still owned the lease; the database said no".
3. Failing SMS: watch the message hop through wait queues in the RabbitMQ UI to the DLQ; fix and redrive from the UI.
4. Escalation: nobody acks, level 2 is paged.
5. "A double check is harmless; a double incident or page is not. Here's each layer that prevents it."

## Concepts learned

[Retries, backoff, jitter](../learning/concepts/retries-backoff-jitter.md) · [dead-letter queues and poison messages](../learning/concepts/dead-letter-queues.md) · [circuit breakers and bulkheads](../learning/concepts/circuit-breakers-and-bulkheads.md) · [distributed locks, leases, fencing](../learning/concepts/distributed-locks-leases-fencing.md) (proven under pause) · [leader election](../learning/concepts/leader-election.md) (why not needed) · [backpressure](../learning/concepts/backpressure.md) · [Node event loop](../learning/concepts/node-event-loop.md) (blocking, measured)

## Interview talking points unlocked

- Redlock debate, fencing tokens, why the database is the arbiter.
- Retry design: tiers, jitter, budgets, permanent vs transient.
- Poison messages and three layers that stop them.
- Bulkheads via prefetch and per-host caps; circuit breakers and where *not* to use them.
- Graceful shutdown and why it matters for leases and acks.
- Escalation as a cancellable timer in the database vs a delayed message.

## Shortcuts taken

| Shortcut | Paid off in |
|----------|-------------|
| Metrics as log lines | Phase 4 |
| No schedule overrides, one policy per organization | Not planned ([open questions](../open-questions.md)) |
| Breaker state per process | Accepted |
| `429 Retry-After` not honoured precisely | Accepted ([ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)) |
