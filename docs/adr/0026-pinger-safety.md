# ADR-0026: Guard the check executor against SSRF and overload

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (timeouts, concurrency cap, basic guard), 3 (full guard, body cap, event-loop metrics)

## Context

The check executor makes HTTP requests to URLs typed in by users. Two classes of problem follow.

**SSRF (server-side request forgery).** Our worker sits inside the internal network. A user who registers `http://postgres:5432`, `http://rabbitmq:15672/api/users`, `http://169.254.169.254/` (cloud metadata address) or `http://localhost:3000/internal/…` is using our server to reach things they cannot reach themselves, and the check result (status code, latency, error type) leaks information about them. Notification has the same exposure with user-supplied webhook URLs.

**Overload.** Node.js runs JavaScript on one thread. It is excellent at waiting on many sockets at once, but:

- Starting unlimited concurrent requests exhausts sockets, memory and file descriptors.
- A target that returns a multi-gigabyte body, or drips bytes forever, ties up memory or a connection.
- Any synchronous CPU work (parsing a huge body, a pathological regular expression) **blocks the event loop**: every other check's timer and response handling stalls, timeouts fire late, and healthy targets appear down.

## Decision drivers

- No request from our network to private, loopback, link-local or metadata addresses unless explicitly allowed for the lab.
- One hostile or broken target must not affect checks of other targets.
- The local lab must still be able to check `flaky-target`, which has a private address.
- Mechanisms should make the event-loop lesson concrete and measurable.

## Options considered

### SSRF defence

| Option | Pros | Cons |
|--------|------|------|
| A. Validate the URL string when the monitor is saved (reject `localhost`, `10.*`, …) | Easy | Hostnames resolve to anything; DNS can change after validation; decimal/hex/IPv6 spellings bypass string checks |
| B. Resolve DNS at save time and check the IP | Catches names pointing at private IPs | **DNS rebinding**: the name resolves to a public IP when checked and to `127.0.0.1` when the request is made |
| C. **Resolve at request time, validate the resolved IP, and connect to exactly that IP** (chosen) | Closes the rebinding gap: the address that was validated is the address used | Needs a custom DNS lookup hook in the HTTP client |
| D. Network isolation: run workers in a network with no route to internal services; egress through a filtering proxy | Strongest; defence in depth | Workers need the database and (via relay) the broker, so full isolation needs an egress proxy container; more infrastructure |
| E. Do not follow redirects | Stops a public URL redirecting to a private one | A monitor behind a redirect reports the 3xx |

### Overload defence

| Option | Pros | Cons |
|--------|------|------|
| F. Unbounded `Promise.all` over due monitors | Simple | The failure mode described above |
| G. **Fixed concurrency cap with a semaphore; claim only as many leases as there are free slots** (chosen) | Bounded memory and sockets; backpressure reaches the scheduler | Throughput per worker is capped (scale with replicas) |
| H. Worker threads per check | True isolation of CPU work | Heavy; checks are I/O-bound and do no CPU work if bodies are not parsed |
| I. **Never read the body** (or read at most a small prefix) (chosen) | Removes the largest CPU and memory risk | No keyword-matching checks in v1 |

## Decision

A shared package `@microsentinel/safe-fetch` (used by monitor-worker and by notification's webhook senders) built on `undici`, plus scheduler rules in the worker.

### SSRF guard (C + E, with an allow-list)

1. Scheme must be `http` or `https`; port must be 80, 443 or ≥ 1024; no credentials in the URL. Checked when the monitor or channel is **saved** (fast feedback: `422`, code `BLOCKED_TARGET`) and again on **every request**.
2. A custom `lookup` function resolves the hostname, then rejects the address if it is in a blocked range: loopback (`127.0.0.0/8`, `::1`), private (`10/8`, `172.16/12`, `192.168/16`, `fc00::/7`), link-local (`169.254/16`, `fe80::/10`), carrier-grade NAT (`100.64/10`), multicast, unspecified, and IPv4-mapped IPv6 forms of all of these. The socket connects to the **validated address**, not to a second resolution of the name.
3. Redirects are **not followed** (`redirect: 'manual'`). A 3xx is judged against `acceptedStatusCodes`.
4. `SSRF_ALLOW_HOSTS` (default empty; `flaky-target,dev-stubs` in the local `.env`) lists hostnames exempt from step 2. This is the lab's escape hatch and is the only way a private address is ever contacted.
5. A blocked request is recorded as a `FAILURE` check with reason `BLOCKED_TARGET`; it does not throw.

Option D (network isolation) is not built; it is described in [security](../cross-cutting/security.md) as the production-grade addition.

### Timeouts

One overall deadline per check, `timeoutMs` (default 10 s, max 30 s), enforced with `AbortSignal.timeout()`, covering DNS, connect, TLS and headers. Separate connect timeout of 5 s. The latency recorded is time to response headers.

### Body handling (I)

The response body is **not read**. After headers arrive the request is aborted and the socket destroyed. This is deliberate: it removes the "huge body" and "slow drip" risks entirely. `HEAD` is offered for targets that support it. Keyword checks are out of scope for v1 ([open questions](../open-questions.md)).

### Concurrency and backpressure (G)

- `CHECK_CONCURRENCY=50` in-flight checks per worker process, enforced with `p-limit`.
- The claim query asks for `LIMIT (50 − inFlight)` leases ([ADR-0010](0010-postgres-leases-for-scheduling.md)). A saturated worker claims nothing; due monitors wait in the database for another worker or a later tick. Nothing queues up in memory.
- undici connection pool: at most 2 connections per origin; keep-alive off for check traffic (every check should measure a fresh connection, and idle sockets to hundreds of hosts are wasted memory).
- **Per-host fairness:** at most 5 concurrent checks to the same hostname, so one organization registering 200 monitors on a slow host cannot occupy all 50 slots.

### Event-loop health

- `perf_hooks.monitorEventLoopDelay()` is sampled and exported as the metric `nodejs.eventloop.delay` (p99). Above 200 ms for 10 s the worker logs a warning and reports itself **not ready**, so it stops being preferred while it recovers.
- Schedule lag (`now − next_check_at` at claim time) is a metric and the main overload signal.
- The [break-it lab](../roadmap/break-it-lab.md) blocks the loop on purpose (a synchronous busy-wait behind a debug endpoint, and `flaky-target`'s `huge-body` mode with body reading temporarily enabled) to make the effect visible.

## Consequences

- Positive: user input cannot steer requests at internal services; rebinding is closed.
- Positive: memory and socket use per worker are bounded by configuration; overload shows up as schedule lag, not as a crash.
- Positive: the same guard protects notification's webhooks.
- Negative: a custom lookup hook is security-sensitive code; it needs thorough tests (IPv6 forms, mapped addresses, multiple A records).
- Negative: not following redirects and not reading bodies make checks less capable than commercial tools.
- Negative: the allow-list is a foot-gun if set carelessly outside the lab.
- Harder later: body-based checks need streaming with a byte cap and a strict time budget, and reopen the event-loop risk.

## Revisit when

- Keyword or JSON assertions are requested: stream at most 64 KB, never buffer more, never run user-supplied regular expressions.
- The system is deployed anywhere with a cloud metadata service: add network-level egress filtering (option D).
- Schedule lag grows with free CPU: raise concurrency or add workers.

## Learning choice?

No. Any product that fetches user-supplied URLs needs this. Building the lookup hook by hand rather than using a library is partly for learning; it must then be tested like security code.

## How to explain it in an interview

> "The pinger calls URLs that users give me, so it's a textbook SSRF risk: someone could point a monitor at my database or a cloud metadata address. I resolve the hostname myself at request time, reject private, loopback and link-local ranges, and connect to the exact IP I validated, which also defeats DNS rebinding. I don't follow redirects. For load, Node is great at I/O concurrency but it's one thread, so I cap in-flight checks per worker with a semaphore and only claim as many jobs as I have free slots, which is backpressure. I never read response bodies, so a huge response can't eat memory or block the event loop, and I export event-loop delay as a metric so I can see if something does block it."
