# gRPC and Protocol Buffers

**There are no `.proto` files in this project, on purpose.**

The brief proposed one gRPC call: Incident asking Auth/Org "who is on call for organization 42?". [ADR-0014](../../adr/0014-on-call-in-incident-and-local-replicas.md) removed that call by moving on-call data into Incident and replacing lookups with local replicas built from events. After that decision no synchronous service-to-service call remains ([communication](../../architecture/communication.md)), so there is nothing for gRPC to carry.

Keeping an unused `.proto` "for completeness" would document an interface that does not exist.

## What you should still be able to say about gRPC

Interviewers ask about it. The short version (longer in [sync vs async communication](../../learning/concepts/sync-vs-async-communication.md)):

| | REST/JSON over HTTP | gRPC |
|-|---------------------|------|
| Contract | OpenAPI, optional, often written after the code | `.proto` file, required, written first |
| Encoding | JSON text | Protocol Buffers binary: smaller and faster to parse |
| Transport | HTTP/1.1 or HTTP/2 | HTTP/2 only |
| Streaming | SSE or WebSockets, separately | Built in: client, server and bidirectional streams |
| Browser | Native | Needs gRPC-Web and a proxy |
| Tooling | `curl`, any HTTP client | Generated clients; `grpcurl` |
| Typical use | Public APIs, browser-facing APIs | Internal service-to-service calls where latency and strict contracts matter |

gRPC is still **synchronous request/response**. It makes a call faster and more strictly typed; it does not remove temporal coupling. That is why switching the on-call lookup from REST to gRPC would not have solved the problem ADR-0014 addresses.

## If you want hands-on gRPC anyway (Phase 5 stretch)

A self-contained exercise that does not change the architecture:

1. Define a read-only service in `docs/contracts/proto/oncall.proto`:

   ```proto
   syntax = "proto3";
   package microsentinel.oncall.v1;

   service OnCallQuery {
     rpc WhoIsOnCall(WhoIsOnCallRequest) returns (WhoIsOnCallResponse);
   }

   message WhoIsOnCallRequest {
     string schedule_id = 1;
     string at = 2;            // RFC 3339; empty = now
   }

   message WhoIsOnCallResponse {
     string user_id = 1;
     string display_name = 2;
     string shift_ends_at = 3;
   }
   ```

2. Expose it from Incident with Nest's gRPC transport (`Transport.GRPC`) next to the existing REST endpoint `GET /v1/schedules/{id}/on-call`.
3. Call it from a small script with a generated client and with `grpcurl`.
4. Compare: payload size, latency, what the contract-first workflow feels like, what happens when a field is added (field numbers, unknown-field handling), and what you would need to expose it to a browser.

This sketch is an exercise description, not a contract of the system. If the exercise is done, move the file here and add it to the [README](../../../README.md) doc map.
