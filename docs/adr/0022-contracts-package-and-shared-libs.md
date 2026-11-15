# ADR-0022: Share event contracts and technical libraries, never domain code

- Status: Accepted
- Date: 2026-11-15
- Phase: 0 (policy), 2 (contracts package)

## Context

A monorepo makes sharing code trivial ([ADR-0003](0003-monorepo-pnpm-workspaces.md)). That is the danger. Every shared import is a compile-time dependency between services. Enough of them and the services can no longer be changed, built or deployed independently: a **distributed monolith**, with the costs of both architectures.

Two specific questions: do event types live in a shared `contracts` package, and what else may be shared?

## Decision drivers

- A producer and its consumers must agree on event shapes; drift should be caught at compile time or in CI, not at 3 a.m.
- A service must be changeable without rebuilding the others.
- Avoid copy-pasting infrastructure code (broker wrapper, JWT guard, tracing setup) seven times.
- The rule must be simple enough to follow without thinking and checkable by a script.

## Options considered

### Event types

| Option | Pros | Cons |
|--------|------|------|
| A. No sharing: each consumer writes its own types for the events it reads | Maximum independence; each consumer declares only the fields it uses (consumer-driven) | Duplicated typing; producer changes are invisible to the compiler; needs contract tests to catch drift |
| B. **One shared `contracts` package** with envelope and payload types (chosen) | One definition; the compiler finds every consumer affected by a change; the same classes validate payloads at runtime | A change to the package rebuilds everyone; tempting to "just add a helper"; makes breaking changes easy to commit atomically, which hides the fact that real deployments are not atomic |
| C. Schema registry + code generation from AsyncAPI/JSON Schema | Language-neutral; compatibility checked centrally | A registry and a generation step for a one-language, one-developer project |

### Everything else

| Kind of code | Share? | Reason |
|--------------|--------|--------|
| Technical plumbing (broker wrapper, outbox relay, inbox, JWT guard, logger and tracing setup, SSRF-safe HTTP client, test helpers) | **Yes** | Identical everywhere by design; a bug fixed once is fixed everywhere; contains no domain knowledge |
| Event contracts | **Yes**, with rules | See below |
| Domain entities, enums with business meaning beyond the contract, business rules, repositories | **No** | This is the coupling that kills independence |
| HTTP DTOs | **No** | A service's API is its own; the web app gets types generated from OpenAPI |
| "Utils" (date helpers, string helpers) | **No** shared `utils`/`common` package | It becomes a junk drawer every service depends on. Small helpers are copied; real needs get a named package |

## Decision

**B for events; technical libraries shared; domain code never shared.**

### Packages

| Package | Contains | May depend on |
|---------|----------|---------------|
| `@microsentinel/contracts` | The envelope type; one payload class per event (with class-validator decorators); event-type string constants; enums that appear in payloads | `class-validator`, `class-transformer` only |
| `@microsentinel/messaging` | AMQP connection, topology, publisher, consumer wrapper, outbox relay, inbox, retry routing | `contracts`, `amqplib`, TypeORM, `observability` |
| `@microsentinel/auth-guard` | `JwtAuthGuard`, `RolesGuard`, `@Roles()`, `@CurrentUser()`, JWKS cache | `jose`, Nest |
| `@microsentinel/observability` | OpenTelemetry bootstrap, pino logger module, request-ID middleware, problem+json exception filter | OTel, pino, Nest |
| `@microsentinel/safe-fetch` | HTTP client with the SSRF guard, timeouts and body-size cap | `undici` |
| `@microsentinel/testing` | Testcontainers helpers, envelope builders, fixtures | dev-only |

### Rules for `contracts`

1. **Data shapes only.** No functions with logic, no entity classes, no database or Nest imports.
2. **Additive changes only** within a version: new *optional* fields. Removing, renaming or changing the meaning of a field requires a new event type (`….v2`) published alongside the old one until every consumer has moved ([versioning](../contracts/versioning.md)).
3. **Develop as if deploys were not atomic.** Even though one commit can change producer and consumer together, order the work as it would deploy: consumer tolerates the new shape first, then the producer emits it.
4. **Consumers are tolerant readers.** They validate only the fields they use and ignore the rest ([ADR-0005](0005-typeorm-and-class-validator-usage.md)).
5. **The package is the code form of the [event catalog](../contracts/event-catalog.md) and [asyncapi.yaml](../contracts/asyncapi.yaml).** A CI check validates every example payload in the catalog against the classes and the AsyncAPI schemas, so the three cannot drift.
6. Enums are **open**: a consumer that meets an unknown value must not crash (treat as "other" or dead-letter deliberately, never an unhandled exception in a `switch`).

### Rules for dependencies (enforced by a script in local CI)

- `apps/*` may import `packages/*`. **`apps/*` may never import another `apps/*`.**
- `packages/*` may never import `apps/*`.
- `contracts` imports nothing from the workspace.
- No package named `common`, `shared`, `core` or `utils`.

The check is a small script over each `package.json` and a lint rule on import paths; details in [repo-layout](../cross-cutting/repo-layout.md).

### The test for "may this be shared?"

> If service A changed this code for its own reasons, would service B have to change too?
> **Yes → it is domain code; do not share it.**
> **No, it changes for technical reasons that apply to everyone equally → share it.**

Example: `MonitorStatus` appears in `monitor.state.changed.v1`, so the *contract enum* lives in `contracts`. Monitor's internal state machine that decides transitions does not. Incident maps the contract enum to its own concept at its boundary (its anti-corruption layer) and never passes the contract type into its domain code.

## Consequences

- Positive: event drift is a compile error; plumbing is written once.
- Positive: a mechanical rule stops cross-service imports.
- Negative: a change in `contracts` or `messaging` rebuilds and redeploys every service. Shared technical libraries are still coupling; they are accepted because they change rarely and for everyone at once.
- Negative: the shared types hide a class of bug that only appears with independent deploys (old consumer, new producer). Contract tests ([ADR-0027](0027-testing-stack.md)) and rule 3 are the counterweight.
- Negative: mapping between contract types and internal types is boilerplate.
- Harder later: with a second language or separately released services, `contracts` must become generated code from AsyncAPI, published and versioned (option C).

## Revisit when

- A service in another language appears.
- Services start to be deployed on separate schedules by different people: publish `contracts` as a versioned package and pin versions per service.
- `contracts` gains anything with an `if` in it.

## Learning choice?

No. A shared contracts package in a monorepo is common practice. The strictness of the rules is what keeps it from going wrong.

## How to explain it in an interview

> "Event types live in one shared contracts package, so producer and consumers compile against the same definitions. The risk is a distributed monolith, so the package has rules: data shapes only, additive changes only, breaking changes mean a new versioned event, and consumers ignore fields they don't know. I also share technical plumbing like the RabbitMQ wrapper and the JWT guard, but never domain code, and a CI script fails if one app imports another. My test is: if one service changed this for its own reasons, would another have to change? If yes, it doesn't get shared."
