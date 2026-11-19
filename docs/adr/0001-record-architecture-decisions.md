# ADR-0001: Record architecture decisions

- Status: Accepted
- Date: 2026-11-15
- Phase: 0

## Context

MicroSentinel is built to be explained. In six months, "why RabbitMQ and not Kafka?" must have a written answer that includes what was known at the time, not a reconstruction. The project also has one developer working ~10 hours a week, so decisions made in week 2 will be forgotten by week 12 unless they are recorded.

An **Architecture Decision Record (ADR)** is a short document that captures one significant decision: the context, the options, the choice and its consequences. ADRs are immutable once accepted; a changed decision gets a new ADR that supersedes the old one.

## Decision drivers

- Every decision must be defensible in an interview.
- The cost of writing must be low, or it will not happen.
- Some decisions are made for learning rather than engineering merit, and that must be visible.

## Options considered

### A. No formal record (commit messages, memory)
- Pro: zero effort.
- Con: the reasoning and the rejected options are lost, which are exactly the parts interviews probe.

### B. One large design document
- Pro: one place to read.
- Con: grows without bound; nobody can tell what changed and when; no status per decision.

### C. ADRs in the repository (Michael Nygard's format, extended)
- Pro: small, numbered, versioned with the code, each with a status.
- Con: many files; needs an index.

## Decision

Use ADRs, stored in `docs/adr/`, numbered `NNNN-short-title.md`, indexed in [README.md](README.md). The template is Nygard's with four additions that serve this project:

- **Phase**: when the decision starts to matter.
- **Revisit when**: concrete triggers, so "it depends" becomes "it depends on X exceeding Y".
- **Learning choice?**: yes/no, and what a pragmatic team would do instead.
- **How to explain it in an interview**: a 30-second spoken answer.

Rules:

1. A decision that is hard to reverse, affects more than one service, or is likely to be questioned gets an ADR.
2. Each ADR lists at least two real options with pros and cons.
3. Decisions the author fixed in advance (RabbitMQ, TypeORM, class-validator) still get a full ADR with status Accepted and an honest comparison.
4. Accepted ADRs are not edited except for typos and links. To change a decision, write a new ADR and set the old one's status to `Superseded by ADR-XXXX`.
5. No other doc may contradict an accepted ADR. The [consistency checklist](../../README.md#consistency-rules) says so.

## Consequences

- Positive: the reasoning survives; the ADR index doubles as an interview revision sheet.
- Negative: ~28 files to keep linked; writing one takes 20–40 minutes.
- Harder later: none.

## Revisit when

- ADRs stop being written during implementation. Then the rule is too heavy and should be relaxed to "one paragraph in the PR description, promoted to an ADR if questioned".

## Learning choice?

No. Small teams use ADRs in practice. The two extra sections (learning choice, interview answer) are specific to this project.

## How to explain it in an interview

> "I kept architecture decision records: one short file per significant decision with the context, the options I considered, what I chose and what it costs. It meant that when I came back to something weeks later, or when someone asks 'why not Kafka', the reasoning is written down, including where the alternative would have been better."
