# ADR-0011: Model incidents as a state machine with an append-only timeline, not with event sourcing

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (state machine), 2 (timeline)

## Context

The brief suggests "possibly event sourcing" for Incident. Incidents do have an obvious history (opened, paged, acknowledged, escalated, resolved), which makes event sourcing tempting.

**Event sourcing** means the list of events *is* the stored state. There is no `incidents` row holding the current status; to know an incident's status you load its events and replay them through a function. Current-state tables, if any, are derived **projections** that can be rebuilt.

That is different from **keeping an audit log next to the state**, where a normal row holds the current state and a second table records what happened.

## Decision drivers

- Users need a timeline of each incident (US-08).
- The current status must be queried and filtered constantly ("all open incidents for my organization").
- The "at most one unresolved incident per monitor" invariant must be enforced under concurrency.
- Implementation effort and the number of new concepts introduced at once (Phase 2 already adds the broker, the outbox and idempotency).

## Options considered

### A. State machine only (current-state row, no history)
- Pro: simplest.
- Con: no timeline; cannot answer "who acknowledged, and when did we escalate?".

### B. State machine + append-only timeline table, written in the same transaction (chosen)
- Pro: current state is one indexed row: trivial queries, a unique partial index enforces the one-open-incident invariant, optimistic locking with a `version` column.
- Pro: the timeline gives the full history the UI needs.
- Pro: ordinary TypeORM and SQL; nothing new to learn at the persistence layer.
- Con: the timeline is **not** the source of truth. If a code path changed the row and forgot the timeline entry, the two would disagree. (Mitigated: one function performs every transition and writes both.)
- Con: no "time travel" beyond what the timeline records; state cannot be rebuilt from history if the row is corrupted.

### C. Full event sourcing
- Pro: perfect audit trail by construction; state at any past moment can be reconstructed; new projections can be built later from old events; no dual write between state and history because history is the state.
- Con: every read of current state needs a projection; "list open incidents" means maintaining a read model, updated synchronously (then why not option B?) or asynchronously (then the API is eventually consistent with itself).
- Con: uniqueness invariants across aggregates ("one open incident per monitor") are hard: there is no row to put a unique index on.
- Con: events are stored forever, so their schema must be versioned and **upcast** (old shapes converted on load) forever.
- Con: needs an event store (PostgreSQL tables with optimistic concurrency per stream at minimum), snapshots for long streams, and a mental shift for every later reader of the code.
- Con: incidents are short-lived with 5–15 events each. The benefits (temporal queries, new projections from history) have almost nothing to work on.

### D. Event-source only Incident, keep the rest conventional
- Same as C, limited to one aggregate. Still pays the full conceptual cost for one small aggregate.

## Decision

Option B.

- `incidents` holds current state, with `status`, `escalation_level`, `next_escalation_at`, `version` ([schema](../data/incident-schema.md)).
- `incident_events` is the timeline: append-only, one row per thing that happened, with a per-incident sequence number, an actor (user or system) and a JSON payload.
- **One function, `applyTransition`, is the only code that changes an incident.** Inside one transaction it: checks that the transition is legal for the current status; updates the row with `WHERE id = $id AND version = $v`; inserts the timeline row; inserts the outbox row(s). If the conditional update matches 0 rows, it reloads and re-evaluates once (the other writer may have made this call a no-op).
- The legal transitions are one table in code, mirrored by the state diagram in the [domain model](../02-domain-model.md#incident-lifecycle).
- Database constraints back the code: `UNIQUE (monitor_id) WHERE status <> 'RESOLVED'`, a `CHECK` on status values, and `UNIQUE (incident_id, seq)` on the timeline.

The timeline is an **audit log**, not an event store. The published events (`incident.opened.v1` and friends) are **integration events** for other services; they are related to the timeline rows but are separate contracts and may differ in shape.

Event sourcing is still taught properly in its [concept guide](../learning/concepts/event-sourcing.md), with a small optional exercise in Phase 5: write a pure function that folds an incident's timeline rows into a status, and assert in a test that it equals the stored row for every incident. That is the essential mechanism of event sourcing, practised without betting the service on it.

## Consequences

- Positive: simple queries, strong invariants from the database, small code.
- Positive: Phase 2 stays focused on messaging.
- Negative: two representations (row and timeline) that must be kept in step by discipline and tests.
- Negative: questions the timeline was not designed to answer cannot be answered retroactively.
- Harder later: migrating to event sourcing means treating the timeline as the seed of an event store and backfilling missing detail; possible, not free.

## Revisit when

- Regulators or customers require a provable, tamper-evident history.
- Product asks for "show me the state of everything as it was last Tuesday at 14:00".
- More than two different read models of incident history are needed and the timeline table is being queried in awkward ways.

## Learning choice?

No. This is what a pragmatic team would do. Recommending *against* event sourcing here, and being able to say why, is the lesson.

## How to explain it in an interview

> "I considered event sourcing for incidents because they have a natural history, and decided against it. With event sourcing the events are the source of truth and current state is a projection, which makes 'list all open incidents' and 'only one open incident per monitor' harder, and you take on event versioning forever. My incidents live a few hours and have maybe ten events. So I used a normal row with a status and a version for optimistic locking, plus an append-only timeline table written in the same transaction. I get the audit trail and simple queries. I'd reconsider if I needed to reconstruct past state or build new read models from history."
