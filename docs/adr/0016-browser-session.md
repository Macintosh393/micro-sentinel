# ADR-0016: Keep the access token in memory and the refresh token in an httpOnly cookie, on a single origin

- Status: Accepted
- Date: 2026-11-15
- Phase: 1

## Context

The React app must stay logged in across page reloads and call the API with a token that every service can verify ([ADR-0015](0015-jwt-rs256-jwks-validate-everywhere.md)). Where the browser keeps credentials decides which attacks matter:

- **XSS** (cross-site scripting): attacker-controlled JavaScript runs in our page. It can read anything JavaScript can read (`localStorage`, variables) and make any request the page can make.
- **CSRF** (cross-site request forgery): another site causes the browser to send a request to us, and the browser **automatically attaches our cookies**. The attacker cannot read the response but can trigger the action.
- **CORS** (cross-origin resource sharing): the browser rule that stops a page on one **origin** (scheme + host + port) from reading responses from another, unless the server opts in with headers.

## Decision drivers

- A stolen long-lived credential is much worse than a stolen 15-minute one.
- Services verify a bearer JWT; whatever the browser does must end in an `Authorization` header by the time a request reaches a service.
- Fewer moving parts: ideally no CORS configuration at all.
- The author must be able to explain cookies vs tokens, CORS and CSRF from this project.

## Options considered

### A. Access and refresh tokens in `localStorage`
- Pro: trivial; survives reloads; no CSRF (nothing is attached automatically).
- Con: any XSS reads both tokens and can use the refresh token for days, from anywhere. The worst option.

### B. Server-side session: opaque session ID in an httpOnly cookie; gateway (or a BFF) holds the tokens and injects the JWT upstream
- Pro: no token is ever visible to JavaScript; instant revocation by deleting the session; the pattern recommended for browser apps by current OAuth guidance.
- Con: needs a session store (Redis) and a stateful gateway/BFF that translates cookie → token on every request. Every state-changing request is cookie-authenticated, so CSRF protection is needed everywhere. More infrastructure and more to build in Phase 1.

### C. Access token in an httpOnly cookie
- Pro: JavaScript cannot read it.
- Con: services expect a bearer header, so the gateway must translate; CSRF protection needed on every write; the SSE and fetch code cannot inspect expiry.

### D. Access token in memory, refresh token in an httpOnly cookie (chosen)
- Pro: the long-lived secret is unreadable by JavaScript. The short-lived one is readable only while the page is open and is never persisted.
- Pro: API requests use an `Authorization` header, which browsers never attach automatically, so **API calls are not CSRF-able**. CSRF matters only for the two cookie-authenticated endpoints.
- Pro: no session store; the gateway stays stateless.
- Con: XSS can still act as the user while the page is open (true of every option) and can call the refresh endpoint to obtain a fresh access token. It cannot exfiltrate the refresh token itself.
- Con: a page reload loses the access token; the app must refresh silently on startup.
- Con: several tabs each hold their own access token and race to rotate the same refresh cookie.

### Origin layout

| Option | Pros | Cons |
|--------|------|------|
| E. Web app and API on different origins (`app.example` / `api.example`) | Independent hosting | CORS on every API route; preflight requests; cookies need `SameSite=None; Secure` and `credentials: 'include'`; much more to get wrong |
| F. **One origin**: the gateway serves the app at `/` and the API at `/api` (chosen) | No CORS at all; cookies can be `SameSite=Strict`; one URL to remember | The gateway proxies static files (fine locally; a CDN or web server in front would do it in production) |

## Decision

**D + F.**

**Single origin.** `http://localhost:8080` serves both. In the Compose dev setup the gateway proxies `/` to the Vite dev server in the `web` container, so the origin is the same in dev and prod-style runs. When running Vite directly on the host (`pnpm dev`, port 5173), Vite's dev proxy forwards `/api` to the gateway, again avoiding CORS. **The gateway sends no CORS headers**; cross-origin browser calls are simply not allowed.

**Tokens**

| | Access token | Refresh token |
|-|--------------|---------------|
| Format | RS256 JWT | 256 random bits, base64url; opaque |
| Lifetime | 15 min | 7 days, **rotated on every use** |
| Stored in browser | A JavaScript variable (module scope in the API client). Never `localStorage` or `sessionStorage` | Cookie `ms_refresh`: `HttpOnly; SameSite=Strict; Path=/api/v1/auth; Max-Age=604800`, plus `Secure` when `COOKIE_SECURE=true` |
| Stored on server | Not stored | SHA-256 hash, with a **family ID** |
| Sent as | `Authorization: Bearer …` | Automatically, only to `/api/v1/auth/*` |

**Flows** (sequence diagrams in [auth in the browser](../frontend/auth-in-the-browser.md)):

- **Login:** `POST /api/v1/auth/login` → body has the access token and user; response sets the cookie.
- **Startup / reload:** the app calls `POST /api/v1/auth/refresh` before rendering protected routes. `200` → logged in; `401` → show login.
- **Expiry:** the client refreshes ~60 s before `exp`, and on any `401` it refreshes once and replays the request. Concurrent refreshes share one in-flight promise.
- **Rotation and reuse detection:** each refresh invalidates the presented token and issues a new one in the same family. Presenting an already-used token means it was copied: the whole family is revoked and the user must log in again.
- **Several tabs:** a `BroadcastChannel` shares the new access token and "logged out" between tabs. If two tabs still race, the loser's request carries a just-rotated token; auth accepts a token that was rotated less than 10 s ago and returns the same successor instead of treating it as theft.
- **Logout:** `POST /api/v1/auth/logout` revokes the family and clears the cookie.

**CSRF.** Only `refresh` and `logout` are authenticated by cookie. Protection is layered:

1. `SameSite=Strict`: the browser does not send the cookie on any cross-site request.
2. Both endpoints require `Content-Type: application/json` and the custom header `X-MS-CSRF: 1`. A cross-site HTML form cannot set either, and a cross-site `fetch` with a custom header needs a CORS preflight, which the gateway never approves.
3. The gateway rejects these two requests if the `Origin` header is present and is not the app's own origin.

A CSRF on `refresh` would in any case gain nothing: the attacker cannot read the response.

**SSE.** The native `EventSource` API cannot send an `Authorization` header. The app reads the stream with `fetch` instead ([ADR-0019](0019-sse-realtime-service.md)), so realtime uses the same bearer token and no cookie.

**XSS is the real threat** for this design, so the defences live there: React's default escaping, no `dangerouslySetInnerHTML`, a Content-Security-Policy header from the gateway (`default-src 'self'`), and no third-party scripts ([security](../cross-cutting/security.md)).

## Consequences

- Positive: no CORS; minimal CSRF surface; the long-lived credential is out of JavaScript's reach; no session store.
- Positive: services see only bearer tokens; browser specifics end at auth and the web client.
- Negative: client complexity: silent refresh, a single-flight refresh promise, replay on `401`, cross-tab coordination.
- Negative: no instant revocation of an access token.
- Negative: single origin ties the app's URL to the gateway.
- Harder later: a separately hosted frontend needs CORS and `SameSite=None` cookies, or a move to option B.

## Revisit when

- The frontend must be served from another origin or a native mobile client appears.
- Instant session revocation becomes a requirement: move to option B (BFF with server-side sessions).

## Learning choice?

No, but note the alternative: current security guidance for browser apps increasingly prefers option B (a BFF that keeps tokens server-side). Option D is widespread, simpler to run, and exercises refresh, rotation, CORS and CSRF concepts directly, which is why it is used here.

## How to explain it in an interview

> "The access token is a 15-minute JWT that only lives in memory, and the refresh token is an opaque value in an httpOnly, SameSite=Strict cookie scoped to the auth path. JavaScript can't read the refresh token, so XSS can't steal the long-lived credential, and since API calls use an Authorization header rather than a cookie, they aren't vulnerable to CSRF. Only refresh and logout use the cookie, and those need a custom header on top of SameSite. I serve the app and API from one origin through the gateway, so there's no CORS at all. Refresh tokens rotate on every use with reuse detection. If I needed instant revocation I'd move to a BFF with server-side sessions."
