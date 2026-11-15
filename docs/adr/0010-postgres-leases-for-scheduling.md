# ADR-0010: Coordinate check workers with PostgreSQL leases and fencing tokens

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (single worker, same mechanism), 3 (several workers)

## Context

Several `monitor-worker` replicas run the same scheduler. Without coordination, every replica checks every monitor: N times the load on targets, N copies of each result, and counters incremented N times, so `failureThreshold: 3` is reached after one real failure with three workers.

First, how bad is each kind of duplicate?

| Duplicate | Harm | Why |
|-----------|------|-----|
| Double **check** (two HTTP GETs to the target) | Almost none | `GET`/`HEAD` are safe methods; the target sees one extra request |
| Double **result applied** (counters incremented twice) | Real | Corrupts confirmation; can cause a false `DOWN` |
| Double **incident** | Serious | Two incidents, two escalation chains |
| Double **page** | Serious | A person woken twice; erodes trust in the system |

So the scheduler must make double checks **rare** and double *application of results* **impossible**. Double incidents and pages are prevented separately, downstream, by idempotent consumers and unique constraints ([ADR-0012](0012-idempotent-consumers.md), [ADR-0013](0013-notification-owns-delivery-state.md)). Defence in depth: the scheduler is not the only thing standing between a race and a 3 a.m. page.

## Decision drivers

- Correctness when a worker stalls (GC pause, `docker pause`, CPU starvation) and then resumes.
- No new infrastructure if possible.
- Workers can be added or killed at any time with no rebalancing step.
- Small enough to implement and reason about in a sitting or two.

## Options considered

### A. Redis lock per monitor (`SET key value NX PX ttl`)
- Pro: simple; fast; widely known.
- Con: a lock with a timeout is a **lease**, and a lease can expire while its holder still believes it holds it (a pause longer than the TTL). Then two workers run at once. The lock alone cannot prevent that.
- Con: the result is written to PostgreSQL, a different system from the lock. To be safe, PostgreSQL must reject the stale writer, which needs a **fencing token** checked by the database. At that point the database is doing the real work.
- Con: single-node Redis loses locks on restart; multi-node **Redlock** is the subject of a well-known disagreement (Martin Kleppmann's critique: it depends on timing assumptions and provides no fencing token; Salvatore Sanfilippo's reply defends it under bounded clock drift). The safe conclusion for both sides: do not rely on a timed lock alone for correctness.

### B. Leases in PostgreSQL with `FOR UPDATE SKIP LOCKED` and a fencing token (chosen)
- Pro: the claim, the schedule, the result and the event are all in one database, so they can share transactions.
- Pro: `SKIP LOCKED` gives a contention-free work queue: each worker grabs different rows.
- Pro: the fencing check is a `WHERE lease_token = $n` on the write, enforced by the same system that stores the data.
- Pro: a dead worker's claims simply expire; nothing to clean up, no rebalance.
- Con: polling the database; throughput bounded by PostgreSQL (thousands of claims per second, far above need).
- Con: lease expiry depends on a clock; using the **database's** `now()` everywhere removes worker clock skew from the picture.

### C. Static partitioning (`hash(monitorId) % N == myIndex`)
- Pro: no coordination at runtime.
- Con: every worker must know N and its own index; changing N reshuffles everything; a dead worker's monitors are not checked until something reassigns them.

### D. Consistent hashing with membership
- Pro: adding or removing a worker moves only ~1/N of the monitors.
- Con: needs a membership service (who is alive?), which is its own distributed-systems problem. Far too much machinery here.

### E. Leader election (one scheduler assigns work to the others)
- Pro: a single decision-maker; no duplicate scheduling.
- Con: needs election and failover; the leader is a bottleneck; a deposed leader that has not noticed still needs fencing.

### F. Let the broker do it (publish "check due" messages; workers compete on a queue)
- Pro: RabbitMQ's competing consumers already hand each message to one worker.
- Con: something must still produce the "due" messages exactly once per interval, which is the same problem moved one step.

## Decision

Option B. All state is in the `monitors` table of `monitor_db` ([schema](../data/monitor-schema.md)): `next_check_at`, `lease_owner`, `lease_expires_at`, `lease_token`.

**Claim** (each worker, every 1 s, while it has free capacity):

```sql
UPDATE monitors m
   SET lease_owner = $worker_id,
       lease_expires_at = now() + interval '60 seconds',
       lease_token = m.lease_token + 1
 WHERE m.id IN (
         SELECT id FROM monitors
          WHERE paused = false
            AND next_check_at <= now()
            AND (lease_expires_at IS NULL OR lease_expires_at < now())
          ORDER BY next_check_at
          LIMIT $free_slots
          FOR UPDATE SKIP LOCKED)
RETURNING m.id, m.url, m.method, m.timeout_ms, m.lease_token, …;
```

**Run** the check outside any transaction (never hold a database transaction open across a network call).

**Complete** (one transaction):

```sql
UPDATE monitors
   SET status = …, consecutive_failures = …, state_version = …,
       next_check_at = …, lease_owner = NULL, lease_expires_at = NULL
 WHERE id = $id AND lease_token = $token_from_claim;   -- the fencing check
-- 0 rows updated → someone else has claimed since: discard this result, write nothing.
-- 1 row updated  → insert recent check, outbox rows; COMMIT.
```

Rules:

- **Lease duration 60 s** is greater than the longest possible check (30 s timeout) plus margin, so a healthy worker never loses its lease mid-check.
- **`lease_token` is the fencing token.** It only ever increases. A worker paused past its lease resumes, tries to complete, matches 0 rows, and its stale result is discarded. This is exactly the "storage rejects old tokens" scheme from Kleppmann's article, with the convenient property that the lock service and the storage are the same database.
- **Time comes from PostgreSQL `now()`** in every statement. Workers' clocks are never compared.
- `$free_slots` = the worker's concurrency cap (50) minus in-flight checks, so claiming is also **backpressure** ([ADR-0026](0026-pinger-safety.md)).
- A worker that crashes leaves leases that expire within 60 s; another worker then claims them (NFR-R4).

What can still happen: the lease expires during a long stall, worker B claims and checks, then worker A wakes and sends its (already in flight) HTTP request too. That is a **double check**, harmless. A's result is fenced out.

## Consequences

- Positive: correct under pauses and crashes; no Redis dependency for Monitor (the brief's "PostgreSQL + Redis" for Monitor becomes PostgreSQL only).
- Positive: the same claim pattern is reused for the outbox relay and for escalation timers in incident. One idea, three uses.
- Negative: one claim query per worker per second even when idle.
- Negative: `monitors` mixes definition, state and scheduling columns in one row, so every completed check updates a row that users also edit. Acceptable at this size; a separate `monitor_schedule` table is the first refactoring if contention shows up.
- Harder later: beyond a few thousand checks per second, a single table becomes the bottleneck; partitioned scheduling (options C/D) becomes necessary.

## Revisit when

- More than ~10 000 monitors or sub-10-second intervals at scale.
- Workers in several regions with no shared database.

## Learning choice?

No. `SKIP LOCKED` job queues are a mainstream PostgreSQL pattern. The pragmatic alternative is a job library that does the same thing for you; writing the SQL by hand is the part done for learning.

## How to explain it in an interview

> "A double ping is harmless, but applying a result twice would corrupt my failure counters, so I needed real mutual exclusion on the write. I use leases in Postgres: workers claim due monitors with `SELECT … FOR UPDATE SKIP LOCKED`, which hands each row to exactly one worker without blocking the others, and each claim increments a lease token. When a worker writes its result it includes `WHERE lease_token = the one I was given`. If it was paused past its lease and another worker took over, that update matches zero rows and the stale result is thrown away. That's a fencing token. I didn't use a Redis lock because a lock with a TTL can expire while you still think you hold it, and then you need the database to fence anyway."
