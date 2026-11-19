# Schema evolution and versioning

**TL;DR:** Contracts between services (event payloads, HTTP APIs, database schemas) must change without breaking whoever is on the other side, because the two sides are never deployed at the same instant and old messages outlive the code that wrote them. The tools: additive changes, tolerant readers, explicit versions for breaking changes, and the expand → migrate → contract sequence.

## The problem it solves

Monitor renames `stateVersion` to `version` in `monitor.state.changed.v1`. Incident, still on the old code, reads `undefined`, treats every event as stale (or as new), and either never opens incidents or opens duplicates. Messages written before the rename sit in queues and in the 5-year lifecycle stream; a replay next year feeds them to code that no longer understands them.

## How it works

Compatibility directions:

| Term | Meaning | Lets you |
|------|---------|----------|
| **Backward compatible** | New readers can read old data | Upgrade consumers first |
| **Forward compatible** | Old readers can read new data | Upgrade producers first |
| **Full** | Both | Deploy in any order |

Safe and unsafe changes to a JSON event:

| Change | Safe? |
|--------|-------|
| Add an optional field | Yes (old readers ignore it, if they are tolerant) |
| Add an enum value | Only if readers handle unknown values |
| Remove a field | No (old readers may need it) |
| Rename a field | No (= remove + add) |
| Change type or unit | No |
| Change meaning, same shape | No, and the hardest to catch |

**Tolerant reader:** read only the fields you need, ignore the rest, handle unknown enum values deliberately.

**Breaking change procedure:** publish `v2` *alongside* `v1`; move consumers one by one; stop `v1` when nobody binds it; keep an **upcaster** for `v1` if old messages may be replayed.

**Expand → migrate → contract** is the same idea for databases: add the new column, write both, backfill, switch reads, remove the old column later.

Schema registries (Confluent Schema Registry with Avro/Protobuf) enforce compatibility rules centrally; with JSON and one repository, tests can do the same job.

## In MicroSentinel

- Event names include the major version: `monitor.state.changed.v1`; the routing key is the full name, so `v1` and `v2` can coexist on the exchange ([versioning](../../contracts/versioning.md)).
- Consumers validate with `whitelist: true` but never `forbidNonWhitelisted` (tolerant reader) ([ADR-0005](../../adr/0005-typeorm-and-class-validator-usage.md)).
- All enums documented as **open**.
- Three sources (catalog, AsyncAPI, `contracts` classes) are checked against each other in CI; Pact pacts record what consumers actually use ([contract testing](contract-testing.md)).
- The monorepo makes atomic producer+consumer changes possible; the rule is to sequence them as if deploys were independent anyway ([ADR-0022](../../adr/0022-contracts-package-and-shared-libs.md)).
- HTTP: `/api/v1/…`, additive changes only within `v1`; problem+json `code`s are stable.
- Database: migrations follow expand → migrate → contract.

## Trade-offs & when NOT to use it

- Dual-publishing and upcasters are real work; avoid breaking changes by designing payloads carefully (IDs, explicit units, no internal enums leaking).
- Long retention (the lifecycle stream) multiplies the cost of every breaking change.
- A private, same-service contract (a command to yourself) can change more freely, but queued messages still need to be readable after a deploy.

## Common misconceptions

- **"JSON is schemaless, so it's flexible."** The schema exists in every consumer's code; it's just undocumented.
- **"Adding a field is always safe."** Only for tolerant readers. A strict validator that rejects unknown fields turns every addition into an outage.
- **"Version numbers on every field change."** Version the *major* compatibility line; additive changes don't need a new version.
- **"The monorepo means we deploy atomically."** Containers restart one at a time; queues hold old messages.

## Interview questions

1. **How do you change an event's schema without breaking consumers?**
   "Additive changes only within a version: new optional fields. Consumers are tolerant readers that ignore fields they don't know. For a breaking change I publish a new version alongside the old one, move consumers over one by one, then stop the old one."
2. **What's a tolerant reader?**
   "A consumer that reads only what it needs and ignores everything else, including unknown enum values handled deliberately. It's what makes adding fields safe."
3. **Backward vs forward compatibility?**
   "Backward: new code reads old data. Forward: old code reads new data. With independent deploys and messages sitting in queues, you usually need both for events."
4. **How do you rename a database column with zero downtime?**
   "Expand, migrate, contract: add the new column, deploy code that writes both and reads the old, backfill, switch reads to the new, then drop the old column in a later release."

## Further reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 4 "Encoding and Evolution".
- Martin Fowler, "TolerantReader" (martinfowler.com bliki) and "ParallelChange" (expand/contract).
- Greg Young, *Versioning in an Event Sourced System* (Leanpub).
- AsyncAPI specification 3.0 documentation.

Used in: [versioning](../../contracts/versioning.md), [ADR-0022](../../adr/0022-contracts-package-and-shared-libs.md), [ADR-0005](../../adr/0005-typeorm-and-class-validator-usage.md), [phase 2](../../roadmap/phase-2.md).
