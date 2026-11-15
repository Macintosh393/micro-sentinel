# Repository layout

One repository, pnpm workspaces ([ADR-0003](../adr/0003-monorepo-pnpm-workspaces.md)). What may be shared and the coupling risk of sharing: [ADR-0022](../adr/0022-contracts-package-and-shared-libs.md).

## Tree

```
micro-sentinel/
├─ README.md
├─ package.json                 # root scripts only: ci:local, test, test:int, e2e, seed, dlq:redrive
├─ pnpm-workspace.yaml          # packages: apps/*, packages/*, tools/*; catalog of shared versions
├─ pnpm-lock.yaml
├─ .npmrc                       # save-exact=true
├─ tsconfig.base.json           # strict, ES2023, module nodenext, experimentalDecorators, emitDecoratorMetadata
├─ .oxlintrc.json, .prettierrc
├─ .gitattributes               # eol=lf
├─ .env.example
├─ compose.yaml
├─ compose.dev.yaml
├─ docs/                        # this documentation
├─ prompts/                     # the brief this documentation was written from
├─ infra/
│  ├─ postgres/init/
│  ├─ rabbitmq/                 # rabbitmq.conf, definitions.json
│  ├─ lgtm/                     # dashboards, alert rules
│  └─ toxiproxy/
├─ pacts/                       # Pact files written by consumer tests
├─ scripts/                     # check-deps.mjs, dlq-redrive.mjs, wait-healthy.mjs
├─ apps/
│  ├─ gateway/
│  ├─ auth/
│  ├─ monitor/                  # one app, MONITOR_ROLE=api|worker
│  ├─ incident/
│  ├─ notification/
│  ├─ analytics/
│  ├─ realtime/
│  └─ web/                      # React + Vite
├─ packages/
│  ├─ contracts/
│  ├─ messaging/
│  ├─ auth-guard/
│  ├─ observability/
│  ├─ safe-fetch/
│  └─ testing/
└─ tools/
   ├─ flaky-target/             # Express
   ├─ dev-stubs/                # Express
   └─ seed/
```

## Inside a Nest service

```
apps/incident/
├─ package.json                # name @microsentinel/incident; deps use catalog: versions
├─ nest-cli.json               # builder: swc
├─ tsconfig.json               # extends ../../tsconfig.base.json
├─ vitest.config.ts            # unplugin-swc for decorator metadata
├─ Dockerfile                  # multi-stage: base, deps, build, prune, runtime, dev
├─ src/
│  ├─ main.ts                  # bootstrap: logger, ValidationPipe, filters, shutdown hooks
│  ├─ instrumentation.ts       # OTel SDK, preloaded with --import
│  ├─ migrate.ts               # entry point of the incident-migrate job
│  ├─ config.ts                # config schema
│  ├─ app.module.ts
│  ├─ incidents/               # one folder per Nest module
│  │  ├─ incidents.module.ts
│  │  ├─ incidents.controller.ts
│  │  ├─ incident.service.ts           # applyTransition
│  │  ├─ incident-state-machine.ts     # pure
│  │  ├─ incident.entity.ts, incident-event.entity.ts
│  │  ├─ dto/                          # HTTP DTOs (class-validator)
│  │  └─ *.spec.ts                     # unit tests next to the code
│  ├─ on-call/
│  ├─ responders/
│  ├─ escalation/
│  ├─ consumers/               # message handlers; ACL mapping from contracts to domain
│  ├─ health/
│  └─ migrations/
└─ test/
   ├─ api/*.api.spec.ts        # Supertest + Testcontainers
   └─ int/*.int.spec.ts        # messaging integration
```

Conventions: one Nest module per folder; entities and DTOs live inside the module that owns them; message handlers are in `consumers/` and convert contract types to domain inputs at the top of the handler (the service's anti-corruption layer); `*.spec.ts` = unit, `*.api.spec.ts` / `*.int.spec.ts` = needs Docker.

## Inside the web app

See [web app](../frontend/web-app.md#structure).

## Package names and dependency rules

| Workspace | Package name | May depend on |
|-----------|--------------|---------------|
| `apps/<svc>` | `@microsentinel/<svc>` | any `packages/*` |
| `apps/web` | `@microsentinel/web` | none of the backend packages (it uses generated OpenAPI types from `docs/contracts/openapi/`) |
| `packages/contracts` | `@microsentinel/contracts` | `class-validator`, `class-transformer` only |
| `packages/messaging` | `@microsentinel/messaging` | `contracts`, `observability` |
| `packages/auth-guard` | `@microsentinel/auth-guard` | `observability` |
| `packages/observability` | `@microsentinel/observability` | — |
| `packages/safe-fetch` | `@microsentinel/safe-fetch` | — |
| `packages/testing` | `@microsentinel/testing` | any package (dev dependency only) |
| `tools/*` | `@microsentinel/<tool>` | nothing from the workspace (fixtures stay independent) |

Enforced by `scripts/check-deps.mjs` in `pnpm ci:local`, which reads every `package.json` and fails if:

- any `apps/*` depends on another `apps/*`;
- any `packages/*` depends on an `apps/*`;
- `contracts` depends on anything in the workspace;
- a package is named `common`, `shared`, `core` or `utils`;
- a workspace imports a path inside another workspace (`../../apps/…`) instead of a package name (checked with a lint rule on import specifiers).

## Shared-library policy

The test for sharing ([ADR-0022](../adr/0022-contracts-package-and-shared-libs.md)): *if one service changed this for its own reasons, would another have to change too?* If yes, it is domain code and is not shared.

| Shared | Not shared |
|--------|-----------|
| Envelope and event payload classes | Entities |
| AMQP connection, outbox, inbox, retry routing | Repositories, services, business rules |
| JWT guard and decorators | Role → permission mapping (each service decides) |
| Logger, tracing bootstrap, problem+json filter | Error codes (each service owns its own) |
| SSRF-safe HTTP client | HTTP DTOs |
| Test helpers | Seed data |

**The coupling risk, stated plainly.** Every service depends on `messaging` and `observability`. A breaking change there rebuilds and redeploys all of them: that is coupling, accepted because those packages change rarely and for everyone at once. If a shared package starts changing for one service's sake, the change belongs in that service. Watch the commit history of `packages/`: frequent single-consumer changes are the early sign of a distributed monolith.

## Build outputs and images

- Packages build to `dist/` with `tsc` (they publish types to the apps); apps build with Nest's SWC builder.
- `pnpm --filter @microsentinel/incident... build` builds incident and everything it depends on (`...` suffix).
- Docker: each app's `Dockerfile` copies the lockfile first, runs `pnpm fetch`, copies sources, builds only that app and its dependencies, and uses `pnpm deploy --filter @microsentinel/incident --prod /out` to produce a minimal folder for the runtime stage ([deployment-local](../architecture/deployment-local.md#images)). The build context is the repository root, with a `.dockerignore` that excludes `node_modules`, `dist`, `docs` and `.git`.

## Why not one `package.json`

Nest CLI monorepo mode shares one `package.json` and `node_modules` across apps ([ADR-0003](../adr/0003-monorepo-pnpm-workspaces.md)). Separate manifests make each service's dependencies explicit, so "which services use `nodemailer`?" has a precise answer, and pnpm's strict layout stops a service from importing a package it never declared.

## Polyrepo, for the record

If services were owned by different teams, the layout would split into one repository per service, with `contracts` generated from AsyncAPI and published to a registry with semantic versions, and Pact files exchanged through a Pact Broker. The dependency rules above are designed so that this split would not require untangling code.
