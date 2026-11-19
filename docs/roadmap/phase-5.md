# Phase 5: Stretch

Optional. Pick by interest and by what target employers use. Each item is self-contained, has an effort estimate, and a check that proves it works. Overview: [phases](phases.md).

## Menu

| # | Item | Effort (h) | Teaches | Done when |
|---|------|-----------|---------|-----------|
| S1 | **Kubernetes with kind + Helm** | 20–30 | Pods, Deployments, Services, probes, ConfigMaps/Secrets, Ingress, Helm charts | `kind create cluster`; `helm install microsentinel ./deploy/helm`; the Phase 2 DoD #4 outage scenario passes against the cluster |
| S2 | **Load testing with k6** (`grafana/k6:1.8.3`) | 8–12 | Load profiles, thresholds, finding the first bottleneck | A k6 script ramps the read API to 200 req/s with p95 < 200 ms (or a written finding of what broke first); a second script registers 2 000 monitors at 30 s and the schedule-lag p95 stays < 15 s with N workers (find N) |
| S3 | **Multi-location checks** | 15–20 | Quorum confirmation, partial failures | Workers labelled `location=a|b|c`; a monitor goes `DOWN` only when ≥ 2 locations confirm; Toxiproxy cutting one location's egress opens no incident |
| S4 | **ClickHouse projection** | 12–16 | Column stores, MergeTree, materialized views | A second projector reads the same streams into ClickHouse; the series endpoint has a backend switch; both return the same numbers |
| S5 | **Per-service broker users with topic permissions** | 4–6 | RabbitMQ authorization | Each service has its own user; publishing an event of another service is refused (`ACCESS_REFUSED`) |
| S6 | **PostgreSQL row-level security** | 6–8 | Defence in depth for tenancy | Policies on tenant tables using a session variable set per transaction; the tenancy test suite passes; a deliberately unfiltered query returns 0 foreign rows |
| S7 | **Flap suppression** | 6–8 | Hysteresis beyond thresholds | Mode `flapping 120 s` for 20 min produces one incident with a "flapping" note, not five |
| S8 | **gRPC exercise** | 6–8 | Contract-first RPC, Protocol Buffers | [proto README](../contracts/proto/README.md) exercise completed; `grpcurl` call works |
| S9 | **Event-sourcing exercise** | 4–6 | Folding events into state | A pure `fold(timelineRows) → status` and a test asserting it equals the stored row for every incident ([ADR-0011](../adr/0011-incident-state-machine-not-event-sourcing.md)) |
| S10 | **Modular-monolith twin** | 20–30 | The comparison, concretely | One Nest app with the five domain modules, one database, in-process events; same e2e suite passes; a write-up comparing code size, memory and latency ([monolith guide](../learning/concepts/microservices-vs-modular-monolith.md)) |
| S11 | **Persist the last JWKS** | 2–3 | Startup resilience | A service restarted while auth is down becomes ready using the cached JWKS file |
| S12 | **Tail sampling** | 4–6 | Sampling strategies | Separate OTel Collector keeps 100 % of error traces and 10 % of the rest |

## Recommended picks for the target roles

For junior full-stack Node interviews, in order of value per hour: **S2** (load testing gives concrete numbers to talk about), **S10** (the monolith twin is the strongest possible answer to "would you use microservices again?"), **S1** (Kubernetes appears in many job adverts), **S9** (one afternoon, deepens the event-sourcing answer).

## S1 notes: Kubernetes mapping

| Compose | Kubernetes |
|---------|-----------|
| service | Deployment + Service |
| `*-migrate` with `service_completed_successfully` | Helm pre-install/pre-upgrade hook Job, or an init container |
| `healthcheck` + `/health/ready` | `readinessProbe` |
| `/health/live` | `livenessProbe` (with a `startupProbe` for slow starts) |
| `.env` | ConfigMap + Secret |
| `--scale monitor-worker=3` | `replicas: 3`; optionally a HorizontalPodAutoscaler on schedule lag |
| gateway on port 8080 | Ingress (the gateway stays as the in-cluster router, or is replaced by the Ingress controller per [ADR-0020](../adr/0020-nestjs-api-gateway.md)) |
| postgres, rabbitmq | StatefulSets, or the official operators |
| Compose DNS | Cluster DNS: `incident.microsentinel.svc.cluster.local` |

The single-active-consumer and lease designs need no change: they never assumed fixed hosts.

## S2 notes: what to measure

- Read path: `GET /api/v1/incidents` and `/monitors` at increasing rates; watch gateway and service p95, DB pool wait time, event-loop delay.
- Check path: monitors × interval vs worker count; watch schedule lag and in-flight checks; find where a single PostgreSQL claim table starts to hurt.
- Write down the first bottleneck and why. "The pool was exhausted at 180 req/s because the gateway's 10 s timeout let requests pile up" is a better interview story than any number.
