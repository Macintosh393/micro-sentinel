# ADR-0018: Run one PostgreSQL server with a database per service, and fit the stack into 8 GB

- Status: Accepted
- Date: 2026-11-15
- Phase: 0

## Context

The brief's stack (Kafka, ClickHouse, Jaeger or Tempo, several PostgreSQL servers, Redis, seven Node services, a web dev server) does not fit comfortably in ~8 GB next to an editor and a browser. Some things must be shared, replaced or made optional. The risk is that a shortcut taken to save memory quietly removes the lesson: for example, sharing one database schema would save nothing and destroy "database per service".

## Decision drivers

- ~8 GB for the whole stack; the core loop should run in about half of that.
- No shortcut may let one service read or write another's tables.
- Each shortcut must be reversible by configuration, not by rewriting code.
- Be explicit about what each shortcut gives up.

## Options considered

### PostgreSQL layout

| Option | Memory | Isolation | Verdict |
|--------|--------|-----------|---------|
| A. One server per service (5 containers) | ~5 × 150–250 MB | Full: separate processes, versions, failure domains | Truest to the pattern; ~1 GB for nothing this project can observe |
| B. **One server, one database and one role per service** (chosen) | ~350 MB | Logical: no cross-database queries in PostgreSQL, and each role is granted only its own database. Shared CPU, memory, disk, version and failure domain | Keeps the rule that matters |
| C. One database, one schema per service | Same as B | Weak: cross-schema joins work unless grants are perfect; one migration history is tempting | Too easy to cheat |
| D. One database, shared tables | Lowest | None | The anti-pattern |

### Everything else

| Component in the brief | Lighter choice | What is given up |
|------------------------|----------------|------------------|
| Kafka (~1 GB+) | RabbitMQ (~400 MB), fixed by the author; streams for replay | Native partitions and compaction ([ADR-0017](0017-ordering-and-replay.md)) |
| ClickHouse (~1 GB) | PostgreSQL in the same server ([ADR-0021](0021-postgres-analytics-store.md)) | Columnar speed at scale |
| Jaeger/Tempo + Prometheus + Loki + Grafana + Collector as five containers | `grafana/otel-lgtm`, one container, optional ([ADR-0025](0025-observability-stack.md)) | Persistence and realistic operations of each tool |
| Redis for Monitor's locks | Removed; leases are in PostgreSQL ([ADR-0010](0010-postgres-leases-for-scheduling.md)) | Nothing |
| Separate Redis per use | One Redis, used only by the gateway | Nothing yet |
| Vite dev server in a container (~300 MB) | nginx with the built app (~20 MB) in prod-style runs | Hot reload, in that mode only |

## Decision

**B**, plus the lighter choices above, plus Compose profiles for optional components.

**PostgreSQL.** One `postgres:18.7` container. An init script creates:

```sql
CREATE ROLE auth_svc LOGIN PASSWORD '…';
CREATE DATABASE auth_db OWNER auth_svc;
REVOKE CONNECT ON DATABASE auth_db FROM PUBLIC;
-- repeated for monitor, incident, notification, analytics
```

Each service receives only its own `DATABASE_URL`. A service that tried to open another service's database would fail authentication. Server settings sized for the budget: `shared_buffers=128MB`, `max_connections=100`; each service's pool is capped at 10 connections (monitor-worker: 5), which totals well under the limit and makes pool exhaustion something the [break-it lab](../roadmap/break-it-lab.md) can demonstrate.

**Which shortcuts break "database per service" in spirit, and which do not**

| Shortcut | Breaks the spirit? | Reason |
|----------|--------------------|--------|
| One server, separate databases and roles | **No** | The pattern is about ownership and coupling through data. Nobody can reach anyone else's tables; moving a service to its own server is a connection-string change |
| …but: shared resources | Partly | A runaway query in analytics can slow incident. A server restart is an outage for all five. Real independence of failure and scaling is lost. Say this when asked |
| One PostgreSQL version for everyone | Slightly | Services cannot choose their own storage technology or version. Analytics using PostgreSQL instead of a column store is a consequence |
| One RabbitMQ for all services | No | A broker is shared infrastructure by nature |
| One Redis (gateway only) | No | Single user |
| Shared schema, cross-database views, `postgres_fdw`, or a "reporting" role that can read everything | **Yes** | Each one recreates the hidden coupling. None is used |
| Analytics reading other services' databases directly "just for reports" | **Yes** | The most common real-world violation. Analytics is fed by events only |

**Profiles and limits.** Optional components sit behind Compose profiles (`analytics`, `obs`, `chaos`); every container has a `mem_limit`. Estimated totals: core ≈ 2.5–2.8 GB, everything ≈ 4–4.5 GB ([deployment-local](../architecture/deployment-local.md)). Estimates are replaced by `docker stats` measurements in the Definition of Done of Phase 0 and each later phase.

**If memory is still tight** (in order): run one `monitor-worker` instead of two; use the prod-style `web` image; stop `lgtm` when not tracing; use host-run services for the one being edited ([deployment-local](../architecture/deployment-local.md), option D).

## Consequences

- Positive: the full system, including tracing, fits with room to spare.
- Positive: the ownership rule is enforced by PostgreSQL permissions, not by convention.
- Negative: one database server is a single point of failure and a shared resource; the system demonstrates logical, not operational, isolation.
- Negative: pool sizes must be coordinated against one `max_connections`.
- Negative: experiments about "one service's database is down" need Toxiproxy in front of a single service's connection, because stopping the server stops everyone.
- Harder later: nothing; splitting servers is configuration.

## Revisit when

- An experiment needs independent database failure for two services: run a second PostgreSQL container for one of them (a `DATABASE_URL` change).
- The machine has 16 GB or more: option A becomes affordable and is the more faithful setup.

## Learning choice?

No. A team with one small machine would do the same, and many production systems start with one cluster and a database per service.

## How to explain it in an interview

> "Locally it's one Postgres server with a separate database and login role per service. Each service can only connect to its own database, so there are no cross-service joins and nobody depends on someone else's schema, which is what database-per-service is really about. What I give up is operational isolation: a heavy analytics query can affect the others and the server is one failure domain. Moving a service to its own server is just a connection string. The thing I'd never do is let a 'reporting' service read everyone's tables, because that's shared-database coupling by the back door."
