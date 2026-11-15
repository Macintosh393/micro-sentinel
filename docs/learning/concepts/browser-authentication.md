# Browser authentication: cookies vs tokens, CORS, CSRF

**TL;DR:** Where the browser keeps credentials decides which attacks matter. Anything JavaScript can read, **XSS** can steal. Anything the browser attaches automatically (cookies), **CSRF** can abuse. **CORS** is the browser's rule about reading responses across origins; it is a relaxation of the same-origin policy, not a server-side security control. A common robust design: short-lived access token in memory, long-lived refresh token in an httpOnly `SameSite` cookie, one origin for app and API.

## The problem it solves

A single-page app must stay logged in across reloads and send credentials on API calls. Every storage option has a failure mode:

- `localStorage`: one XSS bug exfiltrates the token to an attacker who can use it from anywhere for its whole lifetime.
- A plain session cookie: any site the user visits can make the browser send authenticated requests to your API (CSRF).
- Separate origins for app and API: CORS configuration everywhere, preflights, and cookies that must be `SameSite=None`.

## How it works

**Origins:** scheme + host + port. `http://localhost:8080` and `http://localhost:5173` are different origins.

**Same-origin policy:** a page may *send* many cross-origin requests (forms, images, simple `fetch`) but may not *read* the responses unless the server allows it with CORS headers. Requests with custom headers or a JSON content type trigger a **preflight** (`OPTIONS`) first.

**Cookie attributes:**

| Attribute | Effect |
|-----------|--------|
| `HttpOnly` | JavaScript cannot read it (`document.cookie`) |
| `Secure` | Sent only over HTTPS |
| `SameSite=Strict` | Not sent on any cross-site request |
| `SameSite=Lax` | Sent on top-level navigations (GET links) from other sites, not on cross-site POSTs or subresources |
| `SameSite=None` | Sent cross-site (requires `Secure`) |
| `Path` | Sent only for matching paths |

**Attacks and defences:**

| Attack | Needs | Defences |
|--------|-------|----------|
| **XSS** | Script injection into your page | Output encoding (React escapes), CSP, no `dangerouslySetInnerHTML`, no third-party scripts; keep long-lived secrets out of JS reach |
| **CSRF** | Credentials attached automatically (cookies) | `SameSite`, CSRF tokens or custom headers, `Origin` checks; or don't authenticate state-changing requests with cookies |
| **Token theft at rest** | Token in persistent storage | Don't persist access tokens; httpOnly cookies for refresh |

Designs compared:

| Design | XSS impact | CSRF exposure | Server state | CORS |
|--------|-----------|---------------|--------------|------|
| Tokens in `localStorage` | Steal tokens, use anywhere | None | None | If cross-origin |
| **Access token in memory + refresh in httpOnly cookie** | Act as user while page open; cannot take refresh token | Only refresh/logout endpoints | Refresh token store | None if one origin |
| Server session cookie (or BFF holding tokens) | Act as user while page open | All state-changing endpoints | Session store | None if one origin |

## In MicroSentinel

- Access token (15 min JWT) in a module variable; refresh token in `ms_refresh`: `HttpOnly; SameSite=Strict; Path=/api/v1/auth` (+ `Secure` with HTTPS), rotated on every use with reuse detection ([ADR-0016](../../adr/0016-browser-session.md)).
- **One origin**: the gateway serves the app and `/api`; the gateway sends **no** CORS headers. Vite's dev proxy keeps one origin when developing on the host.
- API calls use `Authorization: Bearer`, which browsers never add automatically, so they are not CSRF-able. Cookie endpoints require `X-MS-CSRF: 1` and pass an `Origin` check.
- XSS defences: React escaping, lint ban on `dangerouslySetInnerHTML`, user URLs rendered as links only for `http(s)`, strict CSP from the gateway ([security](../../cross-cutting/security.md#browser-security-headers-set-by-the-gateway)).
- Silent refresh on startup, single-flight refresh, one replay on `401`, `BroadcastChannel` between tabs, cache cleared on logout ([auth in the browser](../../frontend/auth-in-the-browser.md)).

## Trade-offs & when NOT to use it

- The memory + cookie design needs careful client code (refresh races, multiple tabs).
- XSS can still act as the user while the page is open: no browser design fixes XSS; only preventing it does.
- A BFF with server-side sessions keeps all tokens off the browser and allows instant revocation; current OAuth guidance for browser apps leans that way. It costs a session store and a stateful edge.
- If the frontend must live on another origin, you need CORS with an explicit origin allow-list, `credentials: 'include'`, and `SameSite=None; Secure` cookies.

## Common misconceptions

- **"CORS protects my API."** It protects *users' browsers* from pages reading responses; `curl` ignores it. The API still needs authentication.
- **"httpOnly cookies make XSS harmless."** They stop theft of that cookie, not actions taken from the compromised page.
- **"`SameSite=Lax` stops all CSRF."** Not GET requests that change state (don't have those) or same-site attackers (a compromised subdomain).
- **"JWTs must go in localStorage."** They can live in memory, with a cookie-based refresh.
- **"CSRF tokens are needed for bearer-token APIs."** Not when the browser can't attach the token by itself.

## Interview questions

1. **Where do you store a JWT in the browser?**
   "The access token only in memory, never in localStorage, because XSS could read it. The refresh token is an opaque value in an httpOnly, SameSite=Strict cookie scoped to the auth path. On reload the app calls refresh to get a new access token."
2. **What is CSRF and how did you prevent it?**
   "Another site making the user's browser send a request to my API with their cookies attached. My API calls authenticate with an Authorization header, which the browser never adds by itself, so they're not exposed. Only refresh and logout use the cookie, and those need SameSite=Strict, a custom header and an Origin check."
3. **What is CORS?**
   "A browser mechanism that lets a server allow pages from other origins to read its responses. It's a relaxation of the same-origin policy, not protection for the server. I avoided it by serving the app and the API from the same origin through the gateway."
4. **Cookie sessions vs tokens?**
   "Server sessions are simple and instantly revocable but need a session store and CSRF protection on every write. Tokens let services verify statelessly. I combined them: a short-lived token in memory for API calls and a revocable refresh cookie."
5. **What does refresh-token rotation buy you?**
   "Each refresh token works once. If a stolen one is used after the real user has refreshed, the server sees a reused token and revokes the whole family, so theft becomes detectable and both sessions end."

## Further reading

- OWASP Cheat Sheet Series: "Cross-Site Request Forgery Prevention", "Cross Site Scripting Prevention", "Session Management", "HTML5 Security" (local storage).
- MDN Web Docs: "Cross-Origin Resource Sharing (CORS)", "Using HTTP cookies", "Same-origin policy".
- IETF draft "OAuth 2.0 for Browser-Based Applications" (OAuth working group; BFF and token-handling patterns).

Used in: [ADR-0016](../../adr/0016-browser-session.md), [auth in the browser](../../frontend/auth-in-the-browser.md), [security](../../cross-cutting/security.md), [auth-org](../../services/auth-org.md), [phase 1](../../roadmap/phase-1.md).
