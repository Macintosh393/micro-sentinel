# ADR-0028: Use local stand-ins for every external dependency

- Status: Accepted
- Date: 2026-11-15
- Phase: 1 (flaky-target), 2 (Mailpit, dev-stubs)

## Context

The product's edges touch the outside world: targets to check, an email server, an SMS provider, chat and webhook receivers. The hosting constraint is strict: everything self-hosted, no SaaS, no accounts, no API keys, and `docker compose up` must work offline once images are pulled.

Depending on real services would also make demos and tests unreliable (a real website's uptime is not ours to control) and would make failure injection impossible (nobody can ask a real SMS provider to return `503` for the next three calls).

## Decision drivers

- Zero external accounts or keys.
- Deterministic demos and tests: failures on demand.
- Deliveries must be *visible*: a human should see the email, the SMS, the webhook body.
- Real providers must remain possible later without redesign.

## Options considered

### Email

| Option | Pros | Cons |
|--------|------|------|
| Log the email to stdout | Nothing to run | Not SMTP; nothing to look at; cannot test connection failures |
| MailHog | Well known | Unmaintained for years |
| **Mailpit** (chosen) | Maintained; single small binary; SMTP on 1025; web inbox on 8025; REST API that tests can query; can simulate SMTP failures | One more container (~40 MB) |
| A real provider's sandbox | Realistic | Account and key |

### SMS, webhooks, chat

| Option | Pros | Cons |
|--------|------|------|
| Public request-inspection sites | No code | Online only; sends data to a third party |
| WireMock or MockServer | Powerful stubbing | JVM container, large; configuration language to learn |
| **A small Express app in the repo, `dev-stubs`** (chosen) | ~150 lines; exactly the endpoints needed; an inbox page; failure switches; teaches what a webhook receiver looks like from the other side | We maintain it |

### Targets

| Option | Pros | Cons |
|--------|------|------|
| Real public websites | Realistic | Needs network; cannot be made to fail; impolite to hammer |
| httpbin-style images | Ready-made status and delay endpoints | Behaviour is per-URL, not switchable over time: cannot go down *now* and recover *later* without changing the monitor |
| **A small Express app in the repo, `flaky-target`** (chosen) | Mode switchable at runtime while monitors keep checking the same URL | We maintain it |
| Toxiproxy in front of a target | Network-level faults (latency, resets) | Used additionally for the break-it lab; not a target by itself |

## Decision

Three stand-ins, all in the core Compose profile, all without authentication (lab tools on a private network).

### `mailpit` (`axllent/mailpit:v1.33.1`)
Notification sends SMTP to `mailpit:1025`. Inbox UI at `http://localhost:8025`. Tests read messages through Mailpit's HTTP API.

### `dev-stubs` (Express 5, `tools/dev-stubs`, UI at `http://localhost:8090`)

| Endpoint | Stands in for | Behaviour |
|----------|---------------|-----------|
| `POST /webhooks/:inbox` | A customer's generic webhook receiver | Stores headers and body; verifies and displays the `X-MS-Signature` HMAC result; honours `Idempotency-Key` by flagging repeats |
| `POST /chat/:inbox` | A chat platform's incoming webhook | Accepts the Discord-style body `{ "content": "…", "embeds": […] }`; returns `204` |
| `POST /sms/messages` | An SMS gateway's REST API | Accepts `{ "to": "+…", "body": "…" }`; returns `202` with a message ID |
| `GET /` | — | HTML page listing everything received, newest first, per inbox |
| `GET /api/messages?inbox=` | — | JSON for tests |
| `PUT /control` | — | Failure injection per endpoint kind: `{ "target": "sms", "mode": "ok" \| "fail" \| "slow" \| "flaky", "status": 503, "delayMs": 8000, "failCount": 3 }` |
| `DELETE /api/messages` | — | Clear inboxes |

Storage is in memory, capped at 500 messages.

### `flaky-target` (Express 5, `tools/flaky-target`, control at `http://localhost:8091`)

Workers check `http://flaky-target:3000/site/<name>`. Each `<name>` is an independent virtual site, so one container serves many monitors.

| Mode | Behaviour of `GET /site/<name>` |
|------|---------------------------------|
| `up` (default) | `200`, small body, ~20 ms |
| `down` | `503` |
| `timeout` | Accepts the connection, never responds |
| `slow` | `200` after `delayMs` (default 5 000) |
| `flapping` | Alternates `up` and `down` every `periodSeconds` (default 15) |
| `error` | Closes the socket without a response |
| `huge-body` | `200` and streams `sizeMb` of data (default 500) |

Control API: `PUT /control/<name>` with `{ "mode": "down" }` (plus mode-specific fields); `GET /control` lists all sites and modes; `POST /control/reset`. A one-page UI has a row of buttons per site. Because these hostnames resolve to private addresses, they are listed in `SSRF_ALLOW_HOSTS` in the local `.env` ([ADR-0026](0026-pinger-safety.md)).

### Real providers
Notification's senders are small adapters behind one interface (`send(delivery) → result`), selected by channel type and configured by environment variables:

| Channel | Default (local) | To use a real provider |
|---------|-----------------|------------------------|
| `EMAIL` | `SMTP_HOST=mailpit`, port 1025, no auth | Point SMTP settings at any SMTP server |
| `SMS` | `SMS_GATEWAY_URL=http://dev-stubs:3000/sms/messages` | Write an adapter for the provider's API; off by default |
| `WEBHOOK` | URLs entered by users; in the lab they point at `dev-stubs` | Any HTTPS URL passing the SSRF guard |
| `CHAT_WEBHOOK` | Same; body shape is Discord-compatible | A real Discord or Slack-compatible incoming-webhook URL |

No real adapter is built, enabled or required in any phase.

## Consequences

- Positive: the whole product, including every notification channel and every failure mode, works on a plane with no network.
- Positive: demos are scripted: flip `flaky-target` to `down`, watch the incident appear, flip `dev-stubs` SMS to `fail`, watch retries and the DLQ.
- Positive: end-to-end tests assert on real deliveries by querying Mailpit and dev-stubs.
- Negative: two small apps to maintain; they are not the real providers, so provider-specific behaviour (rate limits, bounce handling, delivery receipts) is never exercised.
- Negative: the SSRF allow-list must contain the stand-ins, which is a deliberate hole in a security control, confined to local configuration.
- Harder later: real SMS needs a provider adapter, credentials handling and delivery-receipt webhooks.

## Revisit when

- The system is ever run for real users: add one real email path and treat provider errors as first-class.

## Learning choice?

No. Local stand-ins for external dependencies are standard practice for development and testing. Writing them instead of configuring a general mock server is a simplicity choice.

## How to explain it in an interview

> "Nothing in the project needs an account or an API key. Email goes over SMTP to Mailpit, which is a local mail catcher with an inbox UI. For SMS, webhooks and chat I wrote a small Express receiver that stores what it gets and can be told to fail or respond slowly, so I can demonstrate retries and dead-lettering on demand. And instead of monitoring real websites there's a fake target I can switch between up, down, slow and flapping. The senders are adapters behind one interface, so a real provider is a configuration change, but the demo never depends on one."
