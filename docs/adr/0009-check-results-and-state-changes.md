# ADR-0009: Publish both check results and confirmed state changes; confirm failures in Monitor

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (confirmation logic), 2 (events)

## Context

The brief says Monitor emits "site down" signals. That hides three decisions:

1. **What is published?** Every check result (~17 per second at design scale; needed by analytics) or only status transitions (a few per day; needed by incident)?
2. **Where is a failure confirmed?** One failed check is weak evidence: a dropped packet, a slow garbage collection on the target, a brief network blip. Paging a human for that is how on-call rotations burn out.
3. **Where is flapping handled?** A target that alternates between failing and succeeding could open and resolve incidents every minute.

## Decision drivers

- Incident must not be flooded: it needs a few meaningful signals, each worth opening or closing an incident for.
- Analytics needs every check to compute uptime and latency.
- "Is this monitor down?" is Monitoring's core question ([domain model](../02-domain-model.md)); the answer should have one owner.
- A false page is costly; a page delayed by ~30 seconds is acceptable.

## Options considered

### What to publish

| Option | Pros | Cons |
|--------|------|------|
| A. Only raw check results; consumers derive status | One event type; Monitor stays simple | Every consumer re-implements confirmation, possibly differently. Incident handles 1.4 M events a day to find a handful that matter. The definition of "down" leaks out of its bounded context |
| B. Only state transitions | Low volume; exactly what incident needs | Analytics cannot compute latency or check-based uptime |
| C. **Both, as two event types on separate routing keys** (chosen) | Each consumer subscribes to what it needs; volumes are isolated; the high-volume flow can have different retention and guarantees | Two contracts to maintain |

### Where to confirm a failure

| Option | Pros | Cons |
|--------|------|------|
| D. **In Monitor**, with consecutive-result thresholds (chosen) | One owner of "down"; uses data Monitor already has; trivial to test | A real outage is detected ~20–30 s later than the first failed check |
| E. In Incident (open on first failure, auto-close if it recovers quickly) | Fastest signal | Incident needs check-level detail and timing rules that belong to Monitoring; produces incident noise |
| F. Second-worker confirmation: a different worker must also see the failure | Rules out a fault local to one worker | All workers run in the same Docker network on one machine, so they share every fault that matters. Real value only with workers in different locations |

### Where to handle flapping

| Option | Pros | Cons |
|--------|------|------|
| G. **Hysteresis in Monitor**: separate thresholds for going down and for recovering (chosen) | Simple; damps fast flapping at the source | Slow flapping (minutes down, minutes up) still produces separate incidents |
| H. Flap detection with suppression (count transitions in a window, mark the monitor "flapping", hold notifications) | Quieter | More state and rules; risk of suppressing a real outage |
| I. Reopen a recently resolved incident instead of opening a new one | Fewer incidents | `RESOLVED` stops being terminal; MTTR and escalation semantics get complicated |

## Decision

**C + D + G.**

**Two events:**

| Event | When | Volume | Consumers |
|-------|------|--------|-----------|
| `monitor.check.completed.v1` | After every check | ~17/s at design scale | analytics (via stream `ms.archive.checks`) |
| `monitor.state.changed.v1` | Only when the confirmed monitor status changes | A few per monitor per month | incident, realtime (also retained in `ms.archive.lifecycle`) |

**Confirmation rules** (the state diagram is in the [domain model](../02-domain-model.md)):

- From `UP` or `UNKNOWN`: a failed check increments `consecutive_failures`. When it reaches `failureThreshold` (default 3) the status becomes `DOWN`.
- From `DOWN`: a successful check increments `consecutive_successes`. When it reaches `recoveryThreshold` (default 2) the status becomes `UP`.
- From `UNKNOWN`: the first successful check sets `UP` immediately.
- An opposite result resets the counter to 0.
- While a counter is non-zero (a change is suspected but unconfirmed), the next check is scheduled after `retryIntervalSeconds` (10 s) instead of the full interval, so confirmation takes ~20 s, not `2 × interval`.
- Each status change increments `stateVersion` by 1 and is written, with its outbox rows, in the same transaction as the check result.

Detection time with defaults (60 s interval, 10 s timeout, threshold 3): first failure up to 60 s after the outage begins, then two retries 10 s apart; worst case with timeouts ≈ 60 + 10 + 10 + 10 + 10 + 10 = 110 s, typical ≈ 50 s. With a 30 s interval the worst case is ≈ 80 s, inside the 90 s target in the [requirements](../01-requirements.md).

**`monitor.state.changed.v1` carries what incident needs** (monitor name, URL, cause of the change), so incident never calls Monitor back.

**Second-worker confirmation: not in the core phases.** Retries are claimed through leases, so in practice they are often run by a different worker, but nothing requires it. Real multi-location confirmation is a Phase 5 stretch.

**Slow flapping: accepted.** Each confirmed down period is its own incident and its own page. Suppression (option H) is listed in [open questions](../open-questions.md).

## Consequences

- Positive: incident's consumer is tiny and low-volume; "down" is defined in exactly one place.
- Positive: analytics gets complete data without incident ever seeing it.
- Positive: thresholds are per monitor, so a critical endpoint can use `failureThreshold: 1`.
- Negative: detection is slower than the first failure by design.
- Negative: two event types from one transaction can reach different consumers in different orders; nothing depends on their relative order.
- Harder later: multi-location checks turn confirmation into a quorum decision ("down in 2 of 3 locations"), which changes the state machine.

## Revisit when

- Users complain about slow detection: lower `retryIntervalSeconds` or add per-monitor configuration.
- Users complain about repeated pages from one unstable target: implement option H.
- Workers run in more than one location: implement option F as a quorum.

## Learning choice?

No. This is how uptime monitors generally work. Sending raw results to an analytics pipeline separately from alerting signals is standard.

## How to explain it in an interview

> "Monitor publishes two things: every check result, which is high volume and only analytics cares about, and a state-changed event only when a monitor is confirmed down or confirmed recovered. Confirmation lives in the monitor service because 'is it down' is its question to answer: three consecutive failures with fast retries to go down, two successes to come back. That hysteresis stops a single timeout from paging someone and damps flapping. The incident service just trusts a confirmed transition, so it stays simple and low-volume."
