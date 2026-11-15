# ADR-0005: Use TypeORM with migrations only, and class-validator at every boundary

- Status: Accepted (tool choice fixed by the author; usage rules decided here)
- Date: 2026-11-15
- Phase: 1

## Context

TypeORM and class-validator are fixed. This ADR records the honest comparison with the alternatives and, more importantly, decides **how** to use them: schema management, query style, transactions across a domain write and its outbox row, and validation of consumed events.

An **ORM** (object-relational mapper) maps database rows to objects. A **migration** is a versioned script that changes the schema. A **DTO** (data transfer object) is a class describing the shape of data crossing a boundary.

## Decision drivers

- Junior Nest interviews assume TypeORM and class-validator; job adverts list them.
- Several core patterns here (outbox polling, leases) need SQL the ORM cannot express well.
- The outbox pattern is only correct if the domain write and the outbox insert share one transaction.
- Events from other services are untrusted input, like HTTP bodies.

## Options considered

### ORM

| | TypeORM 1.x (fixed) | Prisma | Drizzle | MikroORM |
|-|---------------------|--------|---------|----------|
| Model | Decorated entity classes; Active Record or Data Mapper | Schema file + generated client | SQL-like query builder with inferred types | Decorated entities, Unit of Work, identity map |
| Nest fit | First-party `@nestjs/typeorm` | Good, community patterns | Manual wiring | First-party-quality module |
| Type safety | Moderate: relations and partial selects are loosely typed | Strong | Strongest, close to SQL | Strong |
| Migrations | Generated from entity diff, hand-editable | Declarative, generated | Generated from schema | Generated |
| Raw SQL escape hatch | `query()` / `QueryRunner` | `$queryRaw` | It is the main API | `execute()` |
| Weak spots | Historic bugs and slow maintenance (1.0 arrived in 2026); `save()` semantics surprise people; easy to write N+1 queries | Separate schema language; interactive transactions historically awkward | More SQL knowledge needed; fewer conveniences | Smaller community; Unit of Work is a concept to learn |

Where an alternative would have been the better call: **Drizzle** for this project's SQL-heavy parts (outbox, leases, rollups), because they are raw SQL anyway and Drizzle types them; **Prisma** for the plain CRUD services, for its type safety and developer experience; **MikroORM** if strict Unit-of-Work semantics mattered. TypeORM wins on one criterion that matters here: it is what the target jobs use.

### Validation

| | class-validator + class-transformer (fixed) | zod |
|-|---------------------------------------------|-----|
| Style | Decorators on classes | Schema objects; types inferred |
| Nest fit | `ValidationPipe` is built around it; `@nestjs/swagger` reads the same classes | Supported in Nest 12 through `StandardSchemaValidationPipe` |
| Weak spots | Needs `reflect-metadata`; types and rules can drift; nested and union types are clumsy; slow release cadence | No decorator/Swagger integration without extra packages |

Where zod would have been better: validating event payloads and config, where a schema value is more convenient than a class. It is used in the browser for forms ([ADR-0024](0024-frontend-stack.md)), where there are no decorators.

## Decision

### Schema management
1. **Migrations only.** `synchronize` is `false` in every environment, including local dev. The only exception is throwaway in-memory experiments.
2. Migrations are generated with the TypeORM CLI from entity changes, then **read and edited by hand** before committing. Indexes with `WHERE` clauses, partitioned tables and `CHECK` constraints are written as raw SQL in the migration.
3. Migrations run in a one-shot `*-migrate` container before the service starts, never at service boot ([deployment-local](../architecture/deployment-local.md)).
4. Schema changes follow **expand → migrate → contract**: add the new column or table, deploy code that writes both, backfill, then remove the old one in a later migration.

### Query style

| Use | For |
|-----|-----|
| **Repository** (`find`, `findOne`, `save`, `insert`, `update`) | Simple CRUD by ID or a couple of equality filters |
| **QueryBuilder** | Dynamic filters, joins, keyset pagination, partial `UPDATE … WHERE status = …` |
| **Raw SQL** (`manager.query(sql, params)`) | `FOR UPDATE SKIP LOCKED` (lease claim, outbox relay, escalation timers), multi-row `INSERT … ON CONFLICT`, rollup `INSERT … SELECT … GROUP BY`, partition maintenance, anything with a CTE |

Rules: entities are **Data Mapper** style (no Active Record); never `save()` for an update where a conditional `UPDATE` is meant; always parameterised queries; no lazy relations.

### Transactions
One explicit style everywhere:

```
await dataSource.transaction(async (manager) => {
  // 1. inbox insert (if handling a message)
  // 2. domain changes through `manager`
  // 3. outbox insert through the same `manager`
});
```

The `EntityManager` is passed explicitly to every function that must take part. No hidden transaction propagation (no CLS-based `@Transactional()` decorator): it is convenient, but it hides exactly which statements share a transaction, which is the thing to learn. The shared `OutboxService.add(manager, event)` **requires** a manager, so it is impossible to write an outbox row outside a transaction.

### Validation
- HTTP: a global `ValidationPipe` with `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`. Unknown fields are rejected with `400`.
- Consumed events: the **same library, different classes, different strictness**. Event payload classes live in `@microsentinel/contracts`. Consumers run `plainToInstance` + `validate` with `whitelist: true` and **`forbidNonWhitelisted: false`**: unknown fields are dropped, not rejected, so producers can add fields without breaking consumers (the **tolerant reader** rule, [versioning](../contracts/versioning.md)). A payload that fails validation is a poison message and goes to the DLQ.
- HTTP DTOs are never reused as event payloads or entities. Three shapes, three classes, explicit mapping between them.

## Consequences

- Positive: the schema is always reproducible from migrations; the SQL that matters is visible as SQL.
- Positive: transaction boundaries are readable in the code.
- Negative: passing `manager` through call chains is verbose.
- Negative: three classes per concept (entity, DTO, event payload) is more typing than sharing one.
- Harder later: switching ORM means rewriting entities and repositories, but raw-SQL parts and migrations carry over.

## Revisit when

- A TypeORM bug blocks a feature for more than a sitting: use raw SQL for that part rather than fighting it.
- Explicit `manager` passing exceeds three levels deep in several places: reconsider a transaction context helper.

## Learning choice?

No for the tools (they match the target stack). The "explicit manager, no decorator" rule is a learning choice; many teams use `typeorm-transactional`.

## How to explain it in an interview

> "TypeORM with migrations only: `synchronize` is off everywhere because it can silently drop columns and it means your schema isn't versioned. I use repositories for simple CRUD, the query builder for dynamic filters, and raw SQL where the ORM gets in the way, like `SELECT … FOR UPDATE SKIP LOCKED` for the outbox and leases. Transactions are explicit: I pass the EntityManager so it's obvious the domain write and the outbox insert commit together. If I were choosing freely I'd look hard at Drizzle or Prisma for type safety, but TypeORM is what most Nest codebases use."
