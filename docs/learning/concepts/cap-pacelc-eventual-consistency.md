# CAP, PACELC and eventual consistency

**TL;DR:** **CAP**: when a network partition happens, a distributed system must choose between staying **consistent** (refuse some requests) and staying **available** (answer with possibly stale data). **PACELC** adds: *else*, when there's no partition, you still trade **latency** against consistency. **Eventual consistency** means copies may disagree for a while but converge once updates stop arriving.

## The problem it solves

These are vocabulary for a trade-off you cannot avoid once data lives in more than one place. In MicroSentinel the same fact (a user's membership, an incident's status) exists in several databases and in browsers. You have to decide, per piece of data, what "correct" means while copies disagree.

## How it works

**CAP** (Brewer's conjecture, proved by Gilbert and Lynch):

- **C**onsistency here means *linearizability*: every read sees the latest write, as if there were one copy.
- **A**vailability: every request to a non-failing node gets a response.
- **P**artition tolerance: the system keeps working when messages between nodes are lost.

Partitions happen whether you like it or not, so the real choice is **CP or AP during a partition**.

**PACELC** (Abadi): **if P**, choose **A or C**; **else** (normal operation), choose **L**atency **or C**onsistency. Synchronous replication = consistent but slower; asynchronous replication = fast but readers may see old data.

Consistency models, strong to weak (simplified):

| Model | Guarantee |
|-------|-----------|
| Linearizable | Reads see the latest write, globally |
| Read-your-writes | You see your own writes; others may lag |
| Monotonic reads | You never see time go backwards |
| Causal | If A caused B, everyone sees A before B |
| Eventual | Copies converge if writes stop |

## In MicroSentinel

| Data | Choice | What a user can observe |
|------|--------|-------------------------|
| Inside one service (incident row + timeline + outbox) | Strong: one PostgreSQL transaction | Nothing inconsistent |
| `responders` replica in incident | Eventual (AP): incident keeps paging from its copy while auth is unreachable | A member added a second ago can't be put on a schedule yet: `422 UNKNOWN_RESPONDER`, UI retries ([ADR-0014](../../adr/0014-on-call-in-incident-and-local-replicas.md)) |
| Default escalation policy after registration | Eventual | `GET /v1/escalation-policy` may `404` for a moment; UI retries ([web app](../../frontend/web-app.md#eventual-consistency-in-the-ui)) |
| Analytics read model | Eventual, seconds behind | `asOf` shown on every panel |
| Browser vs server | Eventual, < 2 s | SSE hint then refetch |
| Acknowledge in the UI | Read-your-writes via optimistic update + server response | Immediate for the clicking user |
| Access token claims vs auth's database | Eventual, ≤ 15 min | A removed member keeps access until token expiry |

The system as a whole is **AP-leaning** between services: during a broker or service outage, every service keeps accepting writes and answering reads from its own data; copies converge when messages flow again. That is the right choice for paging (it must keep working) and the reason idempotency and versions are everywhere.

PACELC in normal operation: the outbox and async events choose **latency** (a write returns before downstream services know about it) over cross-service consistency.

## Trade-offs & when NOT to use it

- Eventual consistency pushes complexity to the edges: UIs that show staleness, APIs that return `422`/`404` for "not yet", consumers that tolerate reordering.
- Choose strong consistency where an invariant must never be violated: one unresolved incident per monitor lives in **one** database for that reason.
- Don't spread a single invariant across services; you'd need distributed coordination to keep it.

## Common misconceptions

- **"Pick two of three."** You can't opt out of partitions; the choice is what to sacrifice *when* one happens.
- **"CAP consistency = ACID consistency."** CAP's C is linearizability; ACID's C is "invariants hold".
- **"Eventual means slow."** Usually milliseconds; it means *no guarantee* of when, not *always late*.
- **"Microservices are eventually consistent, so data can be wrong."** Each service's own data is strongly consistent; only copies lag.

## Interview questions

1. **Explain CAP.**
   "When there's a network partition, a distributed system has to choose between consistency, meaning every read sees the latest write, and availability, meaning every request gets an answer. Partitions aren't optional, so it's really CP or AP during a partition. PACELC adds that even without partitions you trade latency against consistency."
2. **Where is your system eventually consistent, and how do users notice?**
   "Between services. Incident keeps a copy of members from auth's events, so a just-created member can't be scheduled for about a second; the API returns a specific error and the UI retries. The analytics dashboard shows an 'as of' time. Inside each service, writes are strongly consistent in one transaction."
3. **Why accept eventual consistency?**
   "So that incidents can open and pages go out while auth or notification is down. For a paging system, availability matters more than everyone seeing the same member list at the same instant."
4. **How do you keep a critical invariant then?**
   "Keep it inside one service's database. 'One unresolved incident per monitor' is a unique partial index in incident's database, not something two services coordinate."

## Further reading

- Eric Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed" (IEEE Computer, 2012).
- Seth Gilbert and Nancy Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" (ACM SIGACT News, 2002).
- Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (IEEE Computer, 2012): PACELC.
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 9; and "Please stop calling databases CP or AP" (martin.kleppmann.com, 2015).
- Werner Vogels, "Eventually Consistent" (ACM Queue, 2008).

Used in: [data-ownership](../../architecture/data-ownership.md), [ADR-0014](../../adr/0014-on-call-in-incident-and-local-replicas.md), [incident](../../services/incident.md), [analytics](../../services/analytics.md), [phase 2](../../roadmap/phase-2.md), [phase 4](../../roadmap/phase-4.md).
