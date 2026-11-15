# C4 level 1: system context

The C4 model (Simon Brown) describes a system at four zoom levels: Context, Containers, Components, Code. Level 1 shows the system as one box, the people who use it, and the outside systems it touches. Nothing inside the box is shown yet.

```mermaid
C4Context
  title MicroSentinel — system context

  Person(admin, "Owner / Admin", "Configures monitors, schedules, channels")
  Person(member, "Member / on-call responder", "Receives pages, acknowledges and resolves incidents")
  System_Ext(script, "API client", "Script or CI job using an API key")

  System(ms, "MicroSentinel", "Checks URLs, opens incidents, pages responders, reports uptime")

  System_Ext(target, "Monitored targets", "HTTP(S) endpoints registered by users. Untrusted.")
  System_Ext(mail, "Email server", "Local stand-in: Mailpit")
  System_Ext(sms, "SMS gateway", "Local stand-in: dev-stubs")
  System_Ext(hook, "Webhook and chat receivers", "Local stand-in: dev-stubs")

  Rel(admin, ms, "Manages", "HTTPS, browser")
  Rel(member, ms, "Views, acknowledges, resolves", "HTTPS, browser")
  Rel(script, ms, "Manages monitors", "REST + API key")
  Rel(ms, target, "Checks", "HTTP GET/HEAD")
  Rel(ms, mail, "Sends email", "SMTP")
  Rel(ms, sms, "Sends SMS", "HTTP")
  Rel(ms, hook, "Posts incident events", "HTTP POST")
  Rel(mail, member, "Delivers page")
  Rel(sms, member, "Delivers page")
```

If your Markdown viewer does not render `C4Context` (it is a newer Mermaid diagram type), this plain flowchart shows the same thing:

```mermaid
flowchart LR
  admin([Owner / Admin]) -->|browser| ms[MicroSentinel]
  member([Member / responder]) -->|browser| ms
  script([API client]) -->|REST + API key| ms
  ms -->|HTTP checks| target([Monitored targets])
  ms -->|SMTP| mail([Email server])
  ms -->|HTTP| sms([SMS gateway])
  ms -->|HTTP POST| hook([Webhook / chat receivers])
  mail & sms -.-> member
```

## What crosses the boundary

| Boundary crossing | Direction | Trust | Notes |
|-------------------|-----------|-------|-------|
| Browser ↔ MicroSentinel | in | Authenticated user | One origin; see [auth in the browser](../frontend/auth-in-the-browser.md) |
| API client → MicroSentinel | in | API key exchanged for a token | [auth-org](../services/auth-org.md) |
| MicroSentinel → targets | out | **Untrusted** | We call URLs that users typed. This is the main SSRF risk: [security](../cross-cutting/security.md) |
| MicroSentinel → email / SMS / webhooks | out | Webhook URLs are **untrusted** user input | Same SSRF guard; all three are local stand-ins by default ([ADR-0028](../adr/0028-local-stand-ins.md)) |

## Things deliberately outside the system

- **Real email, SMS and chat providers.** The system speaks SMTP and plain HTTP, so a real provider is a configuration change, but none is required and none is on by default.
- **An identity provider.** auth is our own. A real product would probably use Keycloak or a hosted IdP; building it is a [learning choice](../adr/0015-jwt-rs256-jwks-validate-everywhere.md).
- **The flaky target.** `flaky-target` is a lab tool that plays the role of a monitored target. It is part of the repository but not part of the system.

Next: [C4 level 2: containers](c4-containers.md).
