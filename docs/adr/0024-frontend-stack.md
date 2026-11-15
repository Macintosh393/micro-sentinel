# ADR-0024: Build the web app with Vite, React Router, TanStack Query and Mantine

- Status: Accepted
- Date: 2026-11-15
- Phase: 0

## Context

React is fixed. Build tool, routing, data fetching, forms and UI library are open. The app is an authenticated dashboard: lists, detail pages, forms, a few charts, live updates. It has no public pages and no SEO needs. It talks to one origin through the gateway ([ADR-0016](0016-browser-session.md)).

## Decision drivers

- The frontend must grow every phase without becoming the project's time sink.
- Choices should match what junior React interviews discuss: hooks, server state vs client state, routing, forms, testing.
- The API contract is OpenAPI; the frontend should be typed from it, not from hand-written interfaces.
- One container, static files, no server-side rendering infrastructure.

## Options considered

### Build tool / framework

| Option | Pros | Cons |
|--------|------|------|
| **Vite (SPA)** (chosen) | Fast dev server; static output served by nginx; the default for new React apps; tests run on the same toolchain with Vitest | No server rendering (not needed) |
| Next.js | Server components, routing, data loading built in; common in job adverts | Needs a Node server; adds a second backend-like layer in front of a system that already has a gateway and services; its data model competes with the lesson "browser → gateway → services" |
| React Router framework mode | SSR and loaders with the same router | Same objection as Next.js at this size |

### Routing

| Option | Pros | Cons |
|--------|------|------|
| **React Router (library/data mode, client only)** (chosen) | The most widely known router; nested routes and layouts; route-level error boundaries | Its loaders overlap with TanStack Query; we use loaders only for auth gating |
| TanStack Router | Fully type-safe routes and search params | Less common in existing codebases and interviews |

### Server state

| Option | Pros | Cons |
|--------|------|------|
| `useEffect` + `useState` | No dependency; worth doing once to understand | Hand-written loading/error states, caching, deduplication, race conditions, refetching |
| **TanStack Query** (chosen) | Caching, deduplication, background refetch, invalidation, polling (`refetchInterval`), retries; invalidation is exactly what the SSE hints need ([ADR-0019](0019-sse-realtime-service.md)) | A library to learn; cache keys need discipline |
| Redux Toolkit + RTK Query | One store for everything; strong devtools; still common in enterprise codebases | More ceremony than this app's client state justifies |
| SWR | Small, simple | Fewer features around mutations and invalidation |

### Client state
Almost none exists: the session (access token, user) and a few UI toggles. A React context for the session and component state for the rest. No Redux, no Zustand, until something real needs it.

### Forms and validation

| Option | Pros | Cons |
|--------|------|------|
| **React Hook Form + zod** (chosen) | Small re-render footprint; schema-based validation with inferred types; the common pairing | Two libraries; rules are duplicated with the backend's class-validator DTOs |
| Mantine's `@mantine/form` | One fewer dependency | Less transferable |
| Sharing class-validator DTOs with the browser | One definition | Pulls decorators and `reflect-metadata` into the bundle and couples the UI to backend classes; rejected by [ADR-0022](0022-contracts-package-and-shared-libs.md) |

Browser validation is a convenience for the user. The server validates everything again and is the authority; the UI maps `400` problem details back onto fields.

### UI library

| Option | Pros | Cons |
|--------|------|------|
| **Mantine** (chosen) | Complete set (tables, modals, forms, notifications, date inputs) with good defaults and accessibility; little CSS to write | Its styling approach is its own; less CSS practice |
| Tailwind + shadcn/ui | Very popular; components are copied into the repo and fully owned | More assembly and styling decisions per screen |
| MUI | Huge, well known | Heavier; theming is a subject in itself |
| Hand-written CSS | Maximum learning of CSS | Slowest path to a usable dashboard |

### API client
`openapi-typescript` generates types from the six [OpenAPI specs](../contracts/openapi/); `openapi-fetch` gives a typed `fetch` wrapper. A request to a path that does not exist, or with the wrong body, fails to compile.

## Decision

Vite 8 · React 19 · React Router 8 (client-side only) · TanStack Query 5 · Mantine 9 · React Hook Form + zod 4 · Recharts 3 · `openapi-typescript` + `openapi-fetch` · `eventsource-parser`. Tests: Vitest, Testing Library, MSW, Playwright ([ADR-0027](0027-testing-stack.md)). Versions: [tech-stack](../architecture/tech-stack.md).

Structure and conventions are in [web-app](../frontend/web-app.md). The essentials:

- **Server state lives only in TanStack Query.** Never copied into `useState` or context.
- **One API client module** owns the access token, attaches the header, refreshes on `401` and replays once.
- **Query keys follow the URL**: `['incidents', { status }]`, `['incidents', id]`, `['monitors']`. SSE hints invalidate by key prefix.
- **Every screen implements four states**: loading, error (with retry), empty, data.
- **Mutations** invalidate affected queries on success. Acknowledge and resolve use optimistic updates with rollback.
- **Feature folders**, not type folders: `features/incidents/{api,components,pages}`.

## Consequences

- Positive: little custom infrastructure; most time goes to screens and to understanding data flow.
- Positive: the generated client ties the frontend to the documented contract; a backend change that breaks the UI fails the build.
- Positive: polling in Phase 1 and SSE-driven invalidation in Phase 2 use the same query hooks.
- Negative: no Next.js experience from this project, though it appears in many job adverts.
- Negative: Mantine does much of the CSS work, so the project builds little CSS skill.
- Negative: validation rules exist twice (zod in the browser, class-validator on the server).
- Harder later: adding server rendering means adopting a framework and rethinking auth.

## Revisit when

- A public, SEO-relevant page is needed (a status page): a framework with server rendering for that part.
- Client-only state grows past the session and a few toggles: add Zustand.
- React Router 8's API differs from what [web-app](../frontend/web-app.md) assumes: follow the router's docs; the doc's intent (client-side nested routes, auth-gated layout) is what matters.

## Learning choice?

No. This is a mainstream SPA stack.

## How to explain it in an interview

> "It's a Vite single-page app. Server state is all in TanStack Query: it handles caching, loading and error states, and invalidation, which is how live updates work: an SSE hint just invalidates a query key. The only client state is the session, in a context. Routing is React Router with a protected layout. Forms use React Hook Form with zod for instant feedback, but the server is the authority and I map its validation errors back onto the fields. API types are generated from the OpenAPI specs, so if the backend contract changes the frontend stops compiling. I didn't use Next.js because there's nothing to server-render in an authenticated dashboard and I already have a gateway and services behind it."
