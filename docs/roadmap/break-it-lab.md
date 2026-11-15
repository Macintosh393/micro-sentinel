# Break-it lab

Deliberate failure experiments. Each one states a **hypothesis**, how to **induce** the failure, what **should** happen, how to **observe** it, and what it **teaches**. Run them at the end of the phase where they become possible; rerun all of them before an interview. When an experiment does not behave as expected, that is a finding: fix it or write down why the expectation was wrong.

## Setup

- Lab flags in `.env` (never outside the lab): `ALLOW_FAST_INTERVALS=true`, `RETRY_TIERS_SCALE=0.1`, `LAB_DEBUG_ENDPOINTS=true`, `CONSUMER_ACK_DELAY_MS=0` (raised per experiment) ([configuration](../cross-cutting/configuration.md)).
- Chaos profile: `COMPOSE_PROFILES=chaos docker compose -f compose.yaml -f compose.chaos.yaml up -d --wait`. `compose.chaos.yaml` routes three connections through Toxiproxy: incident → PostgreSQL (`toxiproxy:15432`), notification → dev-stubs (`toxiproxy:13000`), monitor-worker → flaky-target (`toxiproxy:13001`). Toxics are added with Toxiproxy's HTTP API on `localhost:8474`.
- Have open: RabbitMQ UI (`:15672`), Mailpit (`:8025`), dev-stubs (`:8090`), Grafana (`:3030`, Phase 4), the app (`:8080`).
- Shorthand: `down checkout` = `curl -X PUT localhost:8091/control/checkout -H 'content-type: application/json' -d '{"mode":"down"}'`.

## Index

| # | Experiment | Possible from | Automated? |
|---|-----------|---------------|-----------|
| 0 | Synchronous signal lost (the motivating failure) | Phase 1 | — |
| 1 | Notification killed during an outage | Phase 2 | Phase 4 |
| 2 | Broker restarted mid-flow | Phase 2 | — |
| 3 | The same event delivered twice | Phase 2 | Phase 4 |
| 4 | A consumer crashes on a poison message | Phase 3 | integration test |
| 5 | Slow downstream (latency injected) | Phase 3 | — |
| 6 | Monitor replica paused past its lease | Phase 3 | integration test |
| 7 | Database connection pool exhausted | Phase 3 | — |
| 8 | The pinger's event loop blocked | Phase 3 | — |
| 9 | Browser live connection dropped while incidents change | Phase 2 | Playwright |
| 10 | Auth down while incidents are being opened | Phase 2 (full value Phase 3) | — |
| 11 | Analytics read model deleted and rebuilt | Phase 4 | — |

---

## 0. Synchronous signal lost

- **Hypothesis:** With Phase 1's synchronous worker → incident call, an outage that starts while incident is down never produces an incident.
- **Induce:** `docker compose stop incident`; `down checkout`; wait 2 min; `docker compose start incident`.
- **Should happen:** The worker logs three failed signal attempts and gives up. No incident is ever opened, even after incident returns.
- **Observe:** `GET /api/v1/incidents?status=OPEN` empty; worker log `incident signal failed after 3 attempts`.
- **Teaches:** Temporal coupling and why retries in memory do not fix it. Motivates the outbox and broker ([ADR-0008](../adr/0008-transactional-outbox.md)).

## 1. Notification killed during an outage

- **Hypothesis:** Pages wait in the broker; incident behaviour is unaffected; on restart nothing is lost and pages for already-acknowledged incidents are cancelled.
- **Induce:** `docker compose stop notification`; `down checkout`; wait for the incident; acknowledge it; `docker compose start notification`.
- **Should happen:** Incident opens and is acknowledged normally. `notification.incident-events` holds the messages. On start the queue drains in seconds; page deliveries end `CANCELLED`; org-channel messages arrive late.
- **Observe:** RabbitMQ UI queue depth and consumer count; `select status, count(*) from deliveries group by 1`; timeline gains `NOTIFICATION_SENT` entries late or not at all for pages. Phase 4: the trace shows minutes between publish and consume.
- **Teaches:** Durable queues as a buffer; eventual consistency; why cancel-on-ack exists ([service-down-recovery](../flows/service-down-recovery.md)).

## 2. Broker restarted mid-flow

- **Hypothesis:** Restarting RabbitMQ during an outage delays events but loses none and duplicates none at the effect level.
- **Induce:** `down checkout`; as soon as the second failed check is recorded, `docker compose restart rabbitmq`.
- **Should happen:** Relays' confirms fail; outbox rows stay unpublished and accumulate. Consumers lose connections; amqplib recovery reconnects with backoff and re-declares topology. Unacked in-flight messages are redelivered and deduplicated by the inbox. realtime's exclusive queue is recreated; browsers refetch on reconnect. The incident opens once.
- **Observe:** `select count(*) from outbox where published_at is null` in monitor_db rising then falling; service logs `amqp reconnecting` / `topology declared`; `/health/ready` false during the gap; `consumer.duplicates` ≥ 0; exactly one incident.
- **Teaches:** Publisher confirms + outbox; connection recovery; why topology declaration must be idempotent; durable vs transient queues.

## 3. The same event delivered twice

- **Hypothesis:** A duplicate message has no second effect, whether it is an exact redelivery or the same fact with a new message ID.
- **Induce:** (a) Set `CONSUMER_ACK_DELAY_MS=5000` on incident; `down checkout`; when the incident's log says `committed`, `docker compose kill incident`; start it. (b) In the RabbitMQ UI, copy the `monitor.state.changed.v1` from `ms.archive.lifecycle` (Get messages, requeue) and publish it to `ms.events` with the same `message_id`; then again with a new `message_id`.
- **Should happen:** (a) Redelivery hits the inbox: ack, no effect. (b) Same ID: inbox. New ID: `stateVersion` ≤ cursor: ignored; even without that, the unique partial index would refuse a second open incident.
- **Observe:** One incident; `consumer.duplicates` +1 for (a) and (b-same); `incident.stale_events_ignored` +1 for (b-new); consumer span attributes `ms.duplicate` / `ms.stale` (Phase 4).
- **Teaches:** At-least-once delivery; inbox in the same transaction; versions for ordering; constraints as the last line ([flapping-and-duplicates](../flows/flapping-and-duplicates.md)).

## 4. A consumer crashes on a poison message

- **Hypothesis:** A bad message ends in a DLQ and never blocks the queue, whether it fails validation, throws every time, or kills the process.
- **Induce:** Publish to `ms.events` with routing key `incident.page.requested.v1`: (a) body `{"nonsense":true}`; (b) a valid envelope whose `userId` is `00000000-0000-0000-0000-000000000000` with lab flag `LAB_THROW_ON_ZERO_UUID=true` (handler throws); (c) the same with `LAB_EXIT_ON_ZERO_UUID=true` (handler calls `process.exit(1)`).
- **Should happen:** (a) Straight to `notification.incident-events.dlq` with `x-ms-last-error: validation…`. (b) Four attempts through wait queues, then DLQ. (c) The container restarts five times (Compose `restart: unless-stopped`); the quorum queue's delivery limit dead-letters the message; the service stays up afterwards. Meanwhile, a real outage still pages normally.
- **Observe:** DLQ depth; message headers (`x-death`, `x-ms-attempt`); `docker compose ps` restart count; real pages still delivered.
- **Teaches:** Poison messages; three layers of protection; why the delivery limit matters when your error handling never runs ([ADR-0023](../adr/0023-retry-topology-ttl-dlx.md)).

## 5. Slow downstream

- **Hypothesis:** A slow SMS gateway delays SMS but not email; the circuit breaker opens; nothing piles up unbounded.
- **Induce:** `curl -X POST localhost:8474/proxies/dev_stubs/toxics -d '{"name":"slow","type":"latency","attributes":{"latency":8000}}'` (beyond the 5 s send timeout); create 30 incidents quickly (pause/resume 30 monitors pointed at a down site, or seed script `pnpm seed:outage 30`).
- **Should happen:** SMS sends time out at 5 s; after 5 failures the breaker for `SMS:dev-stubs` opens and SMS attempts fail instantly as retryable; email deliveries continue at normal speed; SMS deliveries walk the retry tiers. Remove the toxic → half-open trial → breaker closes → SMS sends succeed.
- **Observe:** Notification log `circuit open SMS:dev-stubs`; Mailpit receiving promptly; `deliveries` by channel and status; Phase 4 dashboard "Incidents and notifications".
- **Teaches:** Timeouts, circuit breakers, bulkheads via prefetch, why per-channel isolation matters ([resilience](../cross-cutting/resilience.md)).

## 6. Monitor replica paused past its lease

- **Hypothesis:** A worker frozen longer than its lease cannot write a stale result; another worker takes over; at worst a target sees one extra request.
- **Induce:** 3 workers. Put `checkout` in mode `timeout` (each check takes the full 10 s). Find which worker holds its lease: `select lease_owner, lease_token from monitors where name='Checkout API'`. `docker pause <that container>` for 70 s (> 60 s lease), then `docker unpause`.
- **Should happen:** After 60 s another worker claims (token +1) and checks. On unpause, the paused worker's completion matches 0 rows and is discarded with a `fenced` log line. Counters advance only from real, fenced-in results.
- **Observe:** `lease_token` increments in the table; `monitor.lease.fenced` +1; `monitor_checks` has no row from the paused worker for that interval; flaky-target's request log may show two requests close together.
- **Teaches:** Leases, why lease expiry alone is unsafe, fencing tokens, the Redlock debate in practice ([ADR-0010](../adr/0010-postgres-leases-for-scheduling.md)).

## 7. Database connection pool exhausted

- **Hypothesis:** When incident's pool is exhausted, its HTTP requests fail fast with `503` instead of hanging; message handlers retry; other services are unaffected.
- **Induce:** Set `DB_POOL_MAX=2` for incident; add latency to its database: `curl -X POST localhost:8474/proxies/postgres_incident/toxics -d '{"name":"slowdb","type":"latency","attributes":{"latency":1500}}'`; fire 30 parallel requests: `for i in $(seq 30); do curl -s -o /dev/null -w '%{http_code}\n' -H "authorization: Bearer $T" localhost:8080/api/v1/incidents & done; wait`.
- **Should happen:** A few `200`s; most `503` with code `DB_POOL_EXHAUSTED` after the 2 s acquire timeout; none waits for the gateway's 10 s. Readiness may flip to false. Monitor and notification APIs answer normally. Remove the toxic and everything recovers without restarts.
- **Observe:** Status code tally; `db.client.connection.wait_time`; other services' latency flat.
- **Teaches:** Bounded resources, fail-fast vs queueing, timeout nesting, the shared PostgreSQL server as the remaining shared resource ([ADR-0018](../adr/0018-single-postgres-and-ram-budget.md)).

## 8. The pinger's event loop blocked

- **Hypothesis:** Synchronous work in a worker delays every check it is running, makes healthy targets look slow or timed out, and is visible as event-loop delay.
- **Induce:** (a) `docker compose exec monitor-worker node -e "fetch('http://localhost:3000/debug/block?ms=8000',{method:'POST'})"` (busy-wait for 8 s; endpoint exists only with `LAB_DEBUG_ENDPOINTS=true`). (b) Set `LAB_READ_BODY=true` on one worker and put a target in mode `huge-body` (500 MB) with `JSON.parse`-style buffering enabled by the lab flag.
- **Should happen:** (a) Checks in flight on that worker record inflated latencies, some exceed their deadline and become `TIMEOUT` even though targets are healthy; heartbeat timers fire late; `nodejs.eventloop.delay` p99 spikes; the worker reports not-ready. With thresholds ≥ 2, no false incident (one blocked burst rarely produces consecutive failures). (b) Memory climbs toward `mem_limit`; the container may be OOM-killed and restarted; leases expire and other workers take over. With the lab flag off (the default), the same target is harmless because the body is never read.
- **Observe:** Event-loop delay metric; `monitor.check.duration` outliers; `docker stats`; `docker compose ps` restart count.
- **Teaches:** Node's single thread, why I/O concurrency is cheap and CPU work is not, why the executor never reads bodies, why confirmation thresholds also protect against our own hiccups ([node event loop](../learning/concepts/node-event-loop.md), [ADR-0026](../adr/0026-pinger-safety.md)).

## 9. Browser live connection dropped while incidents change

- **Hypothesis:** When the SSE stream drops, the UI says so, keeps working by polling, and after reconnecting shows the correct state without a reload.
- **Induce:** Two browsers on `/incidents`. `docker compose stop realtime`. In browser A acknowledge an incident; cause a new outage. Wait 30 s. `docker compose start realtime`.
- **Should happen:** Browser B shows "Live updates paused" within ~3 s and keeps updating every 5 s by polling. After restart, B reconnects within the backoff window (≤ 30 s), refetches, indicator turns green. Also try DevTools → Network → Offline for 20 s on B: same behaviour, triggered by the watchdog.
- **Observe:** Indicator; DevTools Network shows `/api/v1/realtime/stream` ending and a new one opening; requests to `/api/v1/incidents` on reconnect.
- **Teaches:** Best-effort push with refetch-on-reconnect as the correctness mechanism; jittered reconnects; heartbeat watchdogs ([frontend realtime](../frontend/realtime.md)).

## 10. Auth down while incidents are being opened

- **Hypothesis:** Incidents open and pages go out with auth completely down; logged-in users keep working until their tokens expire.
- **Induce:** `docker compose stop auth`; `down checkout`; wait for the incident; acknowledge it in an already-open browser; wait 16 min; try to reload.
- **Should happen:** Incident opens; on-call resolved from incident's own tables; email sent. Acknowledge works (cached JWKS). After the token expires, refresh fails and the UI shows "Sign-in is temporarily unavailable". Restart any *other* service while auth is down: it stays not-ready (cannot load JWKS), the honest gap ([Phase 5 S11](phase-5.md)).
- **Observe:** Incident timeline complete; Mailpit; gateway logs `401`s only after expiry.
- **Teaches:** Temporal coupling avoided through local replicas and self-verifying tokens; the remaining coupling at startup ([ADR-0014](../adr/0014-on-call-in-incident-and-local-replicas.md), [ADR-0015](../adr/0015-jwt-rs256-jwks-validate-everywhere.md)).

## 11. Analytics read model deleted and rebuilt

- **Hypothesis:** Truncating the read model and replaying the streams reproduces identical numbers.
- **Induce:** Follow the [analytics-rebuild flow](../flows/analytics-rebuild.md): save the verification queries' output, `POST /internal/v1/analytics/rebuild`, wait for `RUNNING`, rerun. Variant: `docker compose kill analytics` halfway through the replay, then start it.
- **Should happen:** Identical results (except the current hour); the kill variant resumes from the stored offset and still ends identical; dashboard panels show "rebuilding" meanwhile; nothing else in the system notices.
- **Observe:** `/internal/v1/analytics/status` offsets and lag; query output diff.
- **Teaches:** CQRS read models as disposable projections; replayable logs vs queues; offset-in-transaction as effectively-once processing; retention as a hard horizon ([ADR-0017](../adr/0017-ordering-and-replay.md)).

---

## Writing up a run

For each experiment keep a short note in `docs/lab-notes/<date>-<n>.md` (created when you run them): what you expected, what happened, screenshots or query output, and what you changed. Three or four of these notes are excellent interview material: "here's a failure I induced, here's what I expected, here's what actually happened and what I fixed".
