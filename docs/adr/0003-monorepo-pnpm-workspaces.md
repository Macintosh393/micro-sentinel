# ADR-0003: Use one repository with pnpm workspaces

- Status: Accepted
- Date: 2026-11-15
- Phase: 0

## Context

Seven Node services, one React app, two lab tools and a few shared packages need a home. Two questions: one repository or many, and which tool manages the workspace.

A **monorepo** is a single repository holding several independently buildable projects. A **polyrepo** setup gives each service its own repository. A **workspace** is a package manager feature that links local packages together so that `apps/incident` can depend on `packages/contracts` without publishing it.

## Decision drivers

- One developer; changes often span a producer, a consumer and a contract.
- Each service must still build into its own small Docker image.
- The tooling should be understandable, not magic.
- The frontend is Vite, not Nest; the tool must not assume everything is a Nest app.

## Options considered

### A. Polyrepo (one repository per service)
- Pro: the strongest independence; mirrors how separate teams work; forces versioned, published contracts.
- Con: a contract change needs coordinated commits in three repositories; shared packages need a registry (none available offline); eight sets of CI and config for one person.

### B. Nest CLI monorepo mode
- Pro: built into Nest; `nest generate app`; one `nest-cli.json`. Nest 12 uses Rspack as the default bundler for it.
- Con: one shared `package.json` and `node_modules` for all apps, so every service's image carries every service's dependencies unless bundled. Knows nothing about the React app or the Express tools. Hides how packages are linked.

### C. pnpm workspaces, plain (chosen)
- Pro: each app and package has its own `package.json`; dependencies are explicit; pnpm's strict `node_modules` layout stops an app from importing something it did not declare; `pnpm --filter` runs tasks for one project and its dependencies; `pnpm deploy` extracts one app with only its production dependencies for Docker.
- Pro: framework-agnostic: Nest, Vite and Express projects sit side by side.
- Con: no task caching or dependency-graph visualisation out of the box; more hand-written scripts.

### D. Nx (or pnpm workspaces + Turborepo)
- Pro: task graph, caching of builds and tests, "affected" commands, generators.
- Con: another layer of configuration and concepts to learn before writing any service. Caching pays off at a size this project will not reach.

## Decision

A single repository managed by **pnpm workspaces**, without Nx or Turborepo.

- `apps/*` for deployables, `packages/*` for shared libraries, `tools/*` for lab containers. Layout and rules: [repo-layout](../cross-cutting/repo-layout.md).
- Shared dependency versions are declared once in the pnpm **catalog** and referenced as `catalog:` in each `package.json`.
- Each Nest app is generated with the Nest CLI as a standalone ESM project inside `apps/`, so the CLI still works per app (`nest start --watch`, `nest generate`).
- Root scripts fan out with `pnpm -r` and `pnpm --filter`.

What a shared package may contain, and how to stop the monorepo from becoming a distributed monolith, is decided separately in [ADR-0022](0022-contracts-package-and-shared-libs.md).

## Consequences

- Positive: one clone, one install, one lockfile, atomic cross-service changes.
- Positive: small images: `pnpm deploy --filter @microsentinel/incident --prod` produces only what incident needs.
- Negative: a monorepo makes it *easy* to couple services through shared code and to make breaking contract changes in one commit. In a polyrepo those would hurt immediately. Discipline has to replace friction.
- Negative: without caching, `pnpm -r test` runs everything every time.
- Harder later: splitting into polyrepos means publishing `contracts` to a registry and versioning it properly.

## Revisit when

- A full local CI run exceeds ~5 minutes: add Turborepo (it layers on top of pnpm workspaces without restructuring).
- A second developer or team owns a service: consider extracting it and publishing contracts.

## Learning choice?

No. pnpm workspaces is a common, pragmatic choice for a small TypeScript monorepo. Many teams would add Turborepo or Nx from day one; skipping them is a simplicity choice, not a learning one.

## How to explain it in an interview

> "It's a pnpm workspace monorepo: apps, shared packages and a couple of lab tools, each with its own package.json. I chose it because one person changing a producer, a consumer and their contract in one commit is much less painful than three repos, and pnpm's strict resolution plus `pnpm deploy` still gives each service its own small image. The risk with a monorepo is that it makes coupling too easy, so I have explicit rules about what's allowed in shared packages."
