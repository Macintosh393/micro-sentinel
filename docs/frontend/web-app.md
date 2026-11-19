# Web app

The React single-page app in `apps/web`. Stack and reasons: [ADR-0024](../adr/0024-frontend-stack.md). It talks only to the gateway, on its own origin ([ADR-0016](../adr/0016-browser-session.md)).

The lesson this app carries: **what a frontend looks like in a microservices system**. One entry point, a token it must look after, data that arrives from several services and is only eventually consistent, and live updates that are hints rather than truth.

## Structure

```
apps/web/src/
  app/            router, providers (QueryClient, Mantine, Session), layout shell
  lib/
    api/          generated OpenAPI types, the typed client, token handling, problem parsing
    realtime/     SSE connection and query invalidation
  features/
    auth/         login, register, session context
    monitors/     api hooks, components, pages
    incidents/
    on-call/
    notifications/
    analytics/
    settings/
  components/     shared presentational pieces: StatusBadge, EmptyState, ErrorState, PageHeader
```

Feature folders own their query hooks (`useMonitors()`, `useAcknowledgeIncident()`), so a page never calls `fetch` directly.

## Routes and screens

Every screen lists where its data comes from. All paths are under `/api` at the gateway. "RT" = also invalidated by a realtime hint.

| Route | Screen | Phase | Reads | Writes |
|-------|--------|-------|-------|--------|
| `/login` | Login form | 1 | — | `POST /v1/auth/login` |
| `/register` | Register user + organization | 1 | — | `POST /v1/auth/register` |
| `/` | Redirect to `/dashboard` | 0 | — | — |
| `/dashboard` | Counts of monitors by status; unresolved incidents; (Phase 4) organization uptime, MTTA, MTTR | 1, 4 | `GET /v1/monitors` RT · `GET /v1/incidents?status=OPEN` and `?status=ACKNOWLEDGED` RT · `GET /v1/analytics/overview?window=` | — |
| `/monitors` | Table: name, URL, status badge, last checked, interval | 1 | `GET /v1/monitors` RT | `POST …/{id}/pause`, `…/resume`, `DELETE …/{id}` |
| `/monitors/new` | Create form | 1 | — | `POST /v1/monitors` |
| `/monitors/:id` | Definition, status, recent checks; (Phase 4) uptime and latency charts, summary | 1, 4 | `GET /v1/monitors/{id}` RT · `GET /v1/monitors/{id}/checks` · `GET /v1/incidents?monitorId=` RT · `GET /v1/analytics/monitors/{id}/summary` · `…/series` | pause, resume, delete |
| `/monitors/:id/edit` | Edit form | 1 | `GET /v1/monitors/{id}` | `PATCH /v1/monitors/{id}` |
| `/incidents` | List with status filter, "load more" | 1 (polling), 2 (live) | `GET /v1/incidents?status=&cursor=` RT | — |
| `/incidents/:id` | Header, acknowledge/resolve buttons; (Phase 2) timeline; (Phase 3) deliveries | 1, 2, 3 | `GET /v1/incidents/{id}` RT · `GET /v1/deliveries?incidentId=` | `POST …/acknowledge`, `POST …/resolve` |
| `/on-call` | Schedules list with who is on call now | 3 | `GET /v1/schedules` · `GET /v1/schedules/{id}/on-call` · `GET /v1/orgs/current/members` | `DELETE /v1/schedules/{id}` |
| `/on-call/schedules/new`, `/on-call/schedules/:id` | Schedule form: name, start, shift length, ordered members | 3 | `GET /v1/schedules/{id}` · `GET /v1/orgs/current/members` | `POST /v1/schedules`, `PUT /v1/schedules/{id}` |
| `/on-call/escalation` | Escalation policy editor: levels, delay, targets | 3 | `GET /v1/escalation-policy` · `GET /v1/schedules` · `GET /v1/orgs/current/members` | `PUT /v1/escalation-policy` |
| `/deliveries` | Delivery log with status filter; retry button on `DEAD` | 3 | `GET /v1/deliveries?status=&cursor=` | `POST /v1/deliveries/{id}/retry` |
| `/settings/profile` | Display name | 1 | `GET /v1/users/me` | `PATCH /v1/users/me` |
| `/settings/notifications` | My contact methods | 3 | `GET /v1/contact-methods` | `POST`, `PATCH`, `DELETE /v1/contact-methods…` |
| `/settings/channels` | Organization channels; test button | 2 | `GET /v1/channels` | `POST`, `PATCH`, `DELETE /v1/channels…`, `POST …/test` |
| `/settings/members` | Members and roles | 3 | `GET /v1/orgs/current/members` | `POST`, `PATCH`, `DELETE …/members…` |
| `/settings/api-keys` | API keys; the new key shown once | 3 | `GET /v1/api-keys` | `POST /v1/api-keys`, `DELETE /v1/api-keys/{id}` |
| `*` | Not found | 0 | — | — |

Realtime channel: `GET /v1/realtime/stream` from Phase 2 ([realtime](realtime.md)). Session: `POST /v1/auth/refresh`, `/logout` ([auth in the browser](auth-in-the-browser.md)). Contracts: [OpenAPI specs](../contracts/openapi/).

### Route tree

```
<RootLayout>                      QueryClientProvider, MantineProvider, SessionProvider
  /login, /register               public; redirect to /dashboard when already logged in
  <ProtectedLayout>               waits for the startup refresh; redirects to /login on 401
    <AppShell>                    nav, header with user menu and live-connection indicator
      /dashboard … /settings/*
```

Admin-only screens (members, API keys, channels, on-call editing) are hidden from the navigation for `MEMBER`, and their forms are disabled if reached directly. This is a convenience: the services enforce roles, and the UI handles a `403` like any other error.

## Data fetching and caching

Server state lives in TanStack Query only.

| Setting | Value | Why |
|---------|-------|-----|
| `staleTime` | 10 s default | Navigating back within 10 s shows cached data without a request |
| `refetchOnWindowFocus` | on | Returning to the tab refreshes; a cheap safety net for missed hints |
| `refetchInterval` | Phase 1: 5 s on monitor and incident queries. Phase 2+: off while the stream is connected, 5 s while it is not, plus a 60 s safety refetch | [realtime](realtime.md) |
| `retry` | 2 for `GET` on network errors and `502/503/504`; 0 for `4xx`; 0 for mutations | Do not retry what cannot succeed; never auto-retry a write |

### Query keys

| Key | Endpoint | Invalidated by |
|-----|----------|----------------|
| `['monitors', 'list', filters]` | `GET /v1/monitors` | monitor mutations; hint `monitor.status.changed` |
| `['monitors', 'detail', id]` | `GET /v1/monitors/{id}` | same, by ID |
| `['monitors', 'checks', id]` | `GET /v1/monitors/{id}/checks` | hint `monitor.status.changed` for that ID |
| `['incidents', 'list', filters]` | `GET /v1/incidents` (infinite query) | acknowledge/resolve; hint `incident.changed` |
| `['incidents', 'detail', id]` | `GET /v1/incidents/{id}` | same, by ID |
| `['deliveries', filters]` | `GET /v1/deliveries` | retry mutation; hint `incident.changed` for that incident |
| `['analytics', …]` | analytics endpoints | time only (`staleTime` 30 s) |
| `['schedules']`, `['escalation-policy']`, `['channels']`, `['contact-methods']`, `['members']`, `['api-keys']`, `['me']` | their endpoints | their own mutations |

Invalidating by prefix (`['incidents']`) refetches every active incident query; inactive ones refetch when next shown.

### Mutations

- On success: invalidate the affected keys; show a notification.
- **Acknowledge and resolve are optimistic**: the cache is updated immediately, rolled back on error, and reconciled by the refetch. The server's answer always wins. If the server returns `409 INCIDENT_ALREADY_RESOLVED`, the UI rolls back, shows "Already resolved", and refetches.
- Create and edit forms are **not** optimistic; they wait for the server and map validation errors onto fields.

### Eventual consistency in the UI

Three places where the user can see it, and what the UI does:

| Situation | Behaviour |
|-----------|-----------|
| A member was just created and is added to a schedule within a second → `422 UNKNOWN_RESPONDER` | Retry the request once after 1 s; if it fails again, show "This member is still being set up, try again in a moment" |
| A new organization opens the escalation page before its default policy exists → `404` | Show a loading state and retry (3 attempts, 1 s apart) |
| Analytics numbers lag the incident list | Show "as of HH:MM:SS" from `asOf`; never present analytics as live |

## State

| Kind | Where | Examples |
|------|-------|----------|
| Server state | TanStack Query cache | Everything from the API |
| Session | `SessionContext` + module-scoped token in the API client | User, role, access token |
| URL state | Router (path and search params) | Selected incident, status filter, chart window |
| Local UI state | `useState` in the component | Modal open, form input |

Filters live in the URL so a filtered list can be bookmarked and the back button works. There is no global store.

## The four states every screen implements

| State | Rule |
|-------|------|
| **Loading** | Skeleton rows shaped like the content (not a full-page spinner). Shown only on first load; background refetches keep the old data visible |
| **Error** | `ErrorState` with a human message from the problem's `title`, the `requestId` in small print (so it can be found in logs and traces), and a Retry button. `401` is handled by the API client, not the screen |
| **Empty** | `EmptyState` with one sentence and the primary action ("No monitors yet · Create monitor") |
| **Data** | The content |

A partial failure does not blank the page: on `/monitors/:id` in Phase 4, the charts can show an error state while the definition and recent checks render normally. Each query renders its own state. That is the UI consequence of data coming from independent services.

## Forms

React Hook Form + zod. Zod schemas mirror the OpenAPI constraints (for example `intervalSeconds` 30–3600). On `400 VALIDATION_FAILED`, each `errors[].field` is set on the matching input; anything unmatched is shown above the form. Submit buttons are disabled while a mutation is pending, which is the first defence against double submits; the server's idempotent commands are the second.

## Accessibility and UX baseline

- Status is never colour alone: badges have text (`DOWN`) and an icon.
- All actions reachable by keyboard; modals trap focus (Mantine does this).
- Times shown in the browser's time zone with the UTC value in a tooltip.
- Destructive actions (delete monitor, remove member, revoke key) need confirmation.

## Build and serve

- Dev: Vite dev server; the gateway proxies `/` to it, so the origin is the gateway's in every mode.
- Prod-style: `vite build` → static files in `nginx:1.30.6-alpine` with `try_files $uri /index.html` (client-side routes must all return the app) and long cache headers on hashed assets.
- The app has **no environment-specific build**: the API base is the relative path `/api`.

## Testing

Component tests with Testing Library + MSW for each screen's four states; Playwright journeys for the main flows ([testing strategy](../cross-cutting/testing-strategy.md)).

## Frontend slice per phase

| Phase | Added |
|-------|-------|
| 0 | Shell, router, providers, a "hello" page that calls one endpoint through the gateway |
| 1 | Login, register, session handling; monitors CRUD; incidents list and detail with polling; acknowledge/resolve; profile |
| 2 | SSE connection and invalidation; live indicator; incident timeline; channels settings |
| 3 | On-call schedules; escalation policy; contact methods; members; API keys; delivery log with retry |
| 4 | Dashboard analytics; monitor charts; Playwright suite |
