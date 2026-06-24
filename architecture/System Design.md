# System Design

A quick-reference guide covering how to design backend systems: reasoning about capacity, choosing service shapes, designing APIs, applying architectural patterns (microservices, event-driven, CQRS, hexagonal), and assembling the building blocks (load balancers, caches, queues, rate limiters) into systems that meet their requirements.

This document is the architectural lens. Lower-level mechanisms live in `Data Systems.md`, `Distributed Systems.md`, and `Networking Fundamentals.md`; this one is about putting them together.

---

## Table of Contents

1. The Approach to System Design
2. Requirements and Constraints
3. Back-of-the-Envelope Math
4. Service Shapes — Monolith, Modular Monolith, Microservices
5. Event-Driven Architecture
6. CQRS and Event Sourcing
7. Hexagonal / Clean Architecture
8. API Design (REST, gRPC, GraphQL)
9. Idempotency and API Robustness
10. Load Balancing
11. Caching as a Pattern
12. Rate Limiting and Backpressure
13. Building Blocks — Common Patterns
14. Scaling Strategies
15. Reliability and Resilience
16. Multi-Region and Geo
17. Cost as a Design Constraint
18. Evolving Systems

---

## 1. The Approach to System Design

System design is the practice of choosing how to assemble components — services, data stores, queues, caches, networks — into a system that meets functional and non-functional requirements within constraints.

It's almost always a sequence:

```
1. Clarify requirements      what is the system actually for?
2. Estimate scale            how many users, requests/sec (RPS), data?
3. Identify the hard parts   where will it break first?
4. Sketch the architecture   data flow, services, stores
5. Drill into bottlenecks    pick the right tool for each one
6. Discuss tradeoffs         what does this design optimize for, and at what cost?
```

The mistake most engineers make is starting at step 4. The right design depends on the requirements and the scale; without them, the result is a generic answer to a generic problem that probably does not fit.

### What Good System Design Optimizes For

Different systems optimize for different things; not everything can be had at once:

| Priority | Implies |
|----------|---------|
| Latency | Caching, CDNs, regional deployment, optimized data paths |
| Throughput | Horizontal scaling, async processing, batching |
| Availability | Redundancy, failover, graceful degradation |
| Consistency | Coordination, may sacrifice latency and availability |
| Cost | Avoid over-engineering, use managed services, scale to zero |
| Simplicity | Fewer moving parts, fewer failure modes, easier to operate |
| Developer velocity | Conventional patterns, good abstractions, fast feedback |

A real system has to balance all of these. Naming the priority order makes tradeoffs explicit.

### YAGNI

You aren't going to need it. The biggest source of system complexity is engineers building for imagined future scale. Stripe started on Rails+Postgres. Many billion-dollar systems still run on one big PostgreSQL.

**Build for current scale + 1 order of magnitude**, with the door open to evolve further when needed. Don't over-architect on day one.

---

## 2. Requirements and Constraints

Before any architecture decision, clarify:

### Functional Requirements

What the system must do, as user-visible behavior.

```
- Users can post a tweet of up to 280 chars.
- Users can follow other users.
- Users see a timeline of tweets from people they follow, newest first.
- Tweets support replies, retweets, likes.
```

### Non-Functional Requirements

How well the system must do it.

```
- Latency: timeline load p99 < 500 ms.
- Throughput: 10K tweets/sec at peak, 200K timeline reads/sec.
- Availability: 99.99% (~52 min/year of downtime tolerated).
- Consistency: read-your-writes for own posts; eventual for others.
- Data durability: tweets are permanent unless deleted.
- Compliance: GDPR right-to-erasure within 30 days.
- Budget: $X/month.
```

Non-functional requirements drive most architectural decisions. "What's the SLA?" is the first question.

### Constraints

External givens that bound the design space.

```
- Existing infrastructure (AWS, on-prem, hybrid).
- Team size and experience (3 engineers vs 30).
- Regulatory (data residency, encryption, audit).
- Timeline (need a minimum viable product (MVP) in 6 weeks vs months of design).
- Existing systems we must integrate with.
```

### Estimation Discipline

Concrete numbers turn debate into math. If "many users" means 1,000, the answer is "any database"; if "many users" means 100M, the answer is very different. Always push for numbers.

---

## 3. Back-of-the-Envelope Math

Quick mental estimation, to size systems before designing them.

### Powers of Two and Ten

```
2^10  = 1024              ~ 1 K
2^20  = 1,048,576         ~ 1 M
2^30  = 1,073,741,824     ~ 1 B
2^40                      ~ 1 T

1 day = 86,400 sec        ~ 10^5
1 year = 31,536,000 sec   ~ 3 * 10^7
```

### Latency Numbers 

```
L1 cache              ~ 1 ns
RAM                   ~ 100 ns
SSD random read       ~ 100 μs
HDD seek              ~ 10 ms
Datacenter RTT        ~ 0.5 ms
Cross-continent RTT   ~ 150 ms
```

### Typical Throughputs (Single Box, Rough)

```
Memcached / Redis      100,000 - 1,000,000 ops/sec
Postgres (mixed)       10,000 - 50,000 TPS (depending on workload)
Postgres (read-only,    100,000+ QPS
   in-memory)
Kafka broker           ~100 MB/s ingest (modest hardware)
NIC                    1 Gbps  ~ 125 MB/s
                       10 Gbps ~ 1.25 GB/s
SSD                    500 MB/s sequential, 100K IOPS random
NVMe                   3-7 GB/s sequential, 500K+ IOPS
```

RTT = round-trip time; TPS/QPS = transactions/queries per second; IOPS = I/O operations per second; NIC = network interface card; NVMe = non-volatile memory express.

### Example: Sizing a Twitter Timeline

Given: 200M daily users, average 5 timeline loads/day each, p99 latency 500 ms.

```
Reads/sec = 200M * 5 / 86400 sec
         ~ 11,500 reads/sec average
         peak could be 5-10x average -> ~100K reads/sec at peak

Per read: fetch ~100 tweets, each ~280 chars + metadata = ~500 bytes
         100 * 500 = 50 KB per timeline response

Bandwidth: 100K * 50 KB = 5 GB/sec at peak
           -- needs a content delivery network (CDN) or multiple regions, plus caching

Storage growth: 200M users * 2 tweets/day * 500 bytes
              = 200 GB/day = ~70 TB/year
              -- distributed storage / S3 archive

Memory for cache: 200M users * 50 KB timeline = 10 TB
                  -- can't fit in one box; cache shards across many
```

Exact numbers are not the goal; the goal is knowing whether the system is in the "single PG" world or the "needs Kafka and a cache layer" world.

---

## 4. Service Shapes — Monolith, Modular Monolith, Microservices

The choice that shapes everything else: how code is organized into services.

### Monolith

A single deployable unit. All features share a codebase, a process, a database.

```
+----------------------+
|     Single app       |
|  +---+ +---+ +---+   |
|  | A | | B | | C |   |
|  +---+ +---+ +---+   |
+----------+-----------+
           |
       Database
```

**Pros:**

- Simplest deployment, debugging, testing.
- Strong typing across module boundaries.
- Refactoring across boundaries is just a rename.
- Local function calls — no network in the hot path.

**Cons (at scale):**

- Single deployable: any change deploys everything.
- Single runtime: one team's bug crashes everyone.
- Hard to scale individual workloads independently.
- One team's slow build slows everyone.

**Verdict:** start here absent a clear reason not to. Most products never outgrow it.

### Modular Monolith

A monolith with strong module boundaries. Each module owns its data; cross-module communication is through explicit interfaces (function calls, not direct DB access).

```
+--------------------------------------+
|               Monolith               |
|  +--------+  +--------+  +--------+  |
|  | Auth   |  | Order  |  | Catalog|  |
|  | DB1    |  | DB2    |  | DB3    |  |
|  +---+----+  +---+----+  +---+----+  |
|      |           |           |       |
|      +-----------+-----------+       |
|        calls to public interfaces    |
+--------------------------------------+
```

The connecting line is the *only* allowed path between modules. Order calls Auth's published functions; it must never run SQL against Auth's tables (DB1). That one rule — talk through interfaces, never touch another module's database — is what keeps the boundaries real and lets a module be extracted into its own service later.


**Pros:**

- All the simplicity of a monolith.
- Module boundaries discipline coupling — easier to extract a service later if needed.
- Each module can have its own schema.

**Cons:**

- Requires discipline to maintain boundaries (no shortcut DB joins across modules).
- Still one deployment unit, one runtime.

**Verdict:** great middle ground. Many "microservices" stories should have stayed here.

### Microservices

Each service is a separately deployable unit, with its own database, communicating over the network.

```
+--------+   +---------+   +---------+   +-----------+
| Auth   |   | Orders  |   | Catalog |   | Payments  |
| DB     |   | DB      |   | DB      |   | DB        |
+--------+   +---------+   +---------+   +-----------+
    ^             ^             ^              ^
    +-------------+-------------+--------------+
                    HTTP/gRPC, events
```

**Pros:**

- Independent deployment, scaling, technology choices.
- Failure isolation (one service down ≠ system down, if designed well).
- Teams own their service end-to-end.
- Can scale teams: each team's service is small.

**Cons:**

- Distributed system from day one — partial failure, latency, retries, idempotency.
- Cross-service queries are joins over the network. No DB-level transactions.
- Operational complexity: deployment, monitoring, tracing, on-call for many services.
- Schema migrations span multiple services.
- Local function calls became RPCs.

**Verdict:** use when team scale or domain isolation demands it. Many "microservices" architectures are sized for hundreds of services without a hundred teams. The cost is real.

### When to Split

Reasonable triggers:

- A team wants to deploy/scale/test independently from the rest.
- The workload of one component differs sharply (CPU-bound vs IO-bound).
- A component has different uptime requirements.
- A component is written in a different language for good reason.

Bad triggers:

- "Microservices are the modern way" (no, they're a tool).
- "We might need to scale this someday."
- "Each entity gets its own service" (entity != service).

### Internal Communication Patterns

| Pattern | When |
|---------|------|
| Synchronous HTTP/gRPC | Request-response, latency-sensitive, simple cases |
| Async messaging (queue/stream) | Decoupled work, smoothing spikes, fan-out |
| Service mesh (Istio, Linkerd) | mutual TLS (mTLS), retries, observability without app changes |
| GraphQL gateway | Aggregating multiple backends for clients |
| Backend-for-Frontend (BFF) | Per-client-type aggregation layer |

---

## 5. Event-Driven Architecture

Services communicate primarily by emitting and consuming **events** — facts about things that happened — rather than by synchronously calling each other.

```
       order.created
+---------+    event    +-----------+
| Orders  | ----------> |  Kafka /  | -----+
+---------+             |  PubSub   |      |
                        +-----------+      |
                                           v
                              +----------------+----------------+
                              |                |                |
                          +----v----+    +-----v----+   +-------v-----+
                          | Email   |    | Analytics|   | Inventory   |
                          | service |    |          |   |             |
                          +---------+    +----------+   +-------------+
```

When `Orders` emits `order.created`, three downstream services react independently. Orders doesn't know they exist.

### Benefits

- **Loose coupling:** producers don't know consumers. New consumers can be added without changing producers.
- **Resilience:** if a consumer is down, events accumulate and are processed when it recovers.
- **Replayability:** events can be replayed to rebuild state or seed a new consumer.
- **Natural audit log:** every state change is an event.

### Costs

- **Eventual consistency:** consumers lag behind producers.
- **Debugging is harder:** the trace from cause to effect spans the queue.
- **Schema evolution:** events have a versioning problem (consumers across versions).
- **Operational overhead:** the message broker is another system to operate.

### Reliable Publishing

The hazard at the core of every event-driven flow: a service that writes to its database and then publishes an event performs two separate writes. A crash between them commits the state change but loses the event, and downstream consumers never learn it happened; publishing to the queue first has the symmetric failure. The fix is to make the event part of the same local transaction as the state change (transactional outbox) or to derive events from the database's replication log (change data capture). Delivery semantics and the outbox/CDC mechanics live in `Distributed Systems.md` and `Data Systems.md`.

### When To Use

Event-driven architecture is well-suited to:

- Fan-out workflows (one event triggers many parallel actions).
- Workflows that span long timescales or involve human steps.
- Integrations with external systems (where blocking on a synchronous response is not always possible).
- Audit trails and compliance.

Avoid for tightly-coupled request-response flows where the caller needs the result right now. Don't make a synchronous flow async just because async sounds better.

### Choreography vs Orchestration

- **Choreography:** each service reacts to events from others. Decentralized. Hard to see the overall flow.
- **Orchestration:** a coordinator drives the workflow. Centralized. Clearer to trace, but the coordinator is a critical component.

Tools: Temporal, AWS Step Functions, Cadence for orchestration. Plain Kafka/SQS for choreography.

A workflow that spans services — each owning its own data, with no shared transaction — is a **saga**: a sequence of local transactions in which a failure triggers compensating actions that undo the prior steps. Orchestration centralizes that logic in the coordinator; choreography distributes it across the reacting services. The saga model and its failure semantics are in `Distributed Systems.md`.

---

## 6. CQRS and Event Sourcing

Two related patterns often discussed together but separable.

### CQRS (Command Query Responsibility Segregation)

Separate the model for writes (commands) from the model for reads (queries). Different schemas, different stores, potentially different services.

```
Commands (writes)               Queries (reads)
       |                              |
       v                              v
+-------------+              +----------------+
| Write side  |              | Read side      |
| (normalized |              | (denormalized  |
|  store)     | ----events-->| projections)   |
+-------------+              +----------------+
```

The write side handles validation, business logic, transactions. The read side is shaped for the queries the UI needs — fast, denormalized, possibly multiple projections for different views.

**Benefits:**

- Read and write workloads scale independently.
- Read models can be denormalized without complicating writes.
- Multiple read models from the same writes (e.g., search index + UI cache + analytics).

**Costs:**

- More moving parts.
- Eventual consistency between writes and reads.
- More to keep in sync (every projection is a derivation of events).

**When to use:** read patterns differ greatly from write patterns; multiple very different views over the same data; performance requires reads that the write schema doesn't naturally support.

### Event Sourcing

The source of truth is not the current state — it's the **append-only log of events** that produced the state. Current state is derived by replaying events.

```
Events (the source of truth):
  - account_created    (account: 42)
  - deposit            (account: 42, amount: 100)
  - deposit            (account: 42, amount: 50)
  - withdraw           (account: 42, amount: 30)

Derived current state:
  account 42 has balance 120
```

**Benefits:**

- Perfect audit log by construction.
- Time-travel: query state at any past point.
- Multiple projections / read models from one event log.
- Replayable: rebuild any projection from scratch.

**Costs:**

- Mental model shift — current state is a view, not the truth.
- Migrating event schemas is hard (history can never be changed; only new versions can be added, with readers migrated).
- Snapshots needed for performance (replaying millions of events is slow).
- Frameworks tend to be heavy (Axon, EventStore, Marten).

**When to use:** strong audit requirements, complex derived state from many small events, financial / compliance domains.

**When not to use:** simple CRUD (create-read-update-delete) apps. Most apps don't need this; the overhead is significant.

### Immutability vs the Right to Erasure

The audit and replay benefits above depend on the log being append-only — but a system that must honor a deletion request (the GDPR right-to-erasure noted in Section 2) cannot rewrite or remove past events without destroying that guarantee. The two requirements are reconciled by **crypto-shredding**: store the personal data encrypted under a per-subject key, keep only ciphertext in the event log, and hold the keys in a separate mutable store. Erasing a subject means deleting their key; the events stay in place but become permanently undecryptable. The same technique applies to any append-only or long-retention store — event logs, message-broker topics, backups — where physical deletion is impractical.

---

## 7. Hexagonal / Clean Architecture

Aka "ports and adapters." A pattern for organizing application code so that the domain (business logic) is decoupled from external concerns (databases, APIs, UIs).

```
+----------------------------------------------------------+
|                                                          |
|      +--------------------------------------------+      |
|      |                                            |      |
|      |                   Domain                   |      |
|      |       (pure business logic, no I/O)        |      |
|      |                                            |      |
|      +--^------------^------------^------------^--+      |
|         |            |            |            |         |
|      (port)       (port)       (port)       (port)       |
|         |            |            |            |         |
|    +---------+  +---------+  +---------+  +---------+    |
|    |   HTTP  |  |   CLI   |  | Postgres|  |  Kafka  |    |
|    | adapter |  | adapter |  | adapter |  | adapter |    |
|    +---------+  +---------+  +---------+  +---------+    |
|                                                          |
+----------------------------------------------------------+
```

**Ports:** interfaces defined by the domain (e.g., `OrderRepository`, `EmailSender`, `PaymentGateway`).

**Adapters:** implementations of those ports (e.g., `PostgresOrderRepository`, `SesEmailSender`).

**Rule:** dependencies point inward. The domain doesn't know about HTTP, Postgres, or Kafka. Adapters depend on the domain's ports.

### Driving vs Driven Adapters

The single distinction that makes the pattern click: ports come in two kinds depending on *who calls whom*.

- **Driving (primary) adapters** are on the inbound side — they call *into* the domain. An HTTP handler, a CLI command, or a Kafka consumer receives an external request and invokes a domain operation. In the diagram, HTTP and CLI are driving.
- **Driven (secondary) adapters** are on the outbound side — the domain calls *out* to them. A Postgres repository, an email sender, a payment gateway. In the diagram, Postgres and Kafka are driven.

Both obey "dependencies point inward," but by opposite mechanisms. A driving adapter simply depends on the domain and calls it. A driven adapter is wired in by **dependency inversion**: the domain declares the interface it needs, and the adapter — in the outer layer — implements it. So the arrow of *control* points outward (the domain triggers a database write) while the arrow of *dependency* still points inward (the database code depends on the domain's interface, never the reverse).

```python
# Domain declares the outbound port — the interface it needs.
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...

# Domain logic depends only on that port, never on Postgres.
class PlaceOrder:
    def __init__(self, orders: OrderRepository):
        self.orders = orders

    def execute(self, cart: Cart) -> Order:
        order = Order.from_cart(cart)   # business rules live here
        self.orders.save(order)
        return order

# Driven adapter lives in the outer layer and implements the port.
class PostgresOrderRepository:          # satisfies OrderRepository
    def save(self, order: Order) -> None:
        ...                             # real SQL
```

Nothing in `PlaceOrder` imports Postgres. The concrete `PostgresOrderRepository` is constructed once at the **composition root** (`main`, or a DI container) and injected in — the only place that knows both the domain and the infrastructure.

**Benefits:**

- Domain logic testable without I/O (use in-memory adapters in tests).
- Swap implementations without changing domain (Postgres → DynamoDB).
- Clear separation between "what the system does" and "how it does it."

**Costs:**

- Indirection. More files, more interfaces.
- Easy to over-apply on simple CRUD apps where the domain is trivially "save and load."

**When to use:** complex domain logic that must evolve independently of infrastructure choices, long-lived systems with shifting tech stacks, code that must be testable without spinning up databases.

---

## 8. API Design (REST, gRPC, GraphQL)

The API is the contract between client and server. Choices here propagate through every consumer.

### REST

Resources and verbs over HTTP. The dominant style for public APIs.

```
GET    /orders            list orders
GET    /orders/42         get one order
POST   /orders            create
PUT    /orders/42         replace
PATCH  /orders/42         partial update
DELETE /orders/42         delete

Conventions:
  Plural nouns for collections.
  Nested for relationships: /users/42/orders
  Status codes carry meaning (200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500, 503).
```

**Pros:**

- Universal HTTP semantics, cacheable by intermediaries.
- Self-describing via URLs and HTTP methods.
- Browser-friendly, easy to debug with curl.

**Cons:**

- Verbose for highly nested data.
- Multiple round trips for complex compositions.
- "Over-fetching and under-fetching" — clients get either too much or have to make many calls.

### gRPC

RPC over HTTP/2 with Protobuf-encoded payloads. Strongly typed via .proto files.

```protobuf
service Orders {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc StreamOrders(StreamRequest) returns (stream Order);
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated Item items = 3;
}
```

**Pros:**

- Strongly typed, generated clients in many languages.
- Efficient binary protocol.
- Streaming (bidirectional) built in.
- Excellent for service-to-service.

**Cons:**

- Not browser-native (needs gRPC-Web proxy).
- Tooling and ecosystem smaller than REST.
- Schema evolution requires discipline (no removing fields, only deprecating).

**Common:** REST/JSON externally, gRPC internally.

### GraphQL

Clients query the exact shape of data they need from a typed schema.

```graphql
query {
  user(id: "42") {
    name
    orders(limit: 5) {
      id
      total
      items {
        name
      }
    }
  }
}
```

**Pros:**

- Clients fetch exactly what they need in one round trip.
- Single endpoint, single schema.
- Great for diverse clients (mobile, web, with different needs).

**Cons:**

- Server-side complexity: every field needs a resolver; N+1 queries are easy.
- Caching is harder (no HTTP-level cache key per resource).
- Authorization at field level is fiddly.
- Performance: clients can request expensive queries.

**Used by:** GitHub, Shopify, Facebook (which invented it). For consumer-facing apps with rich data needs.

### Common API Design Principles

**Versioning:**

- URL: `/v1/orders` (simple, but gives a resource multiple canonical URIs across versions, breaking permalinks and bookmarks).
- Header: `Accept: application/vnd.example.v1+json` (cleaner, less common).
- Add fields liberally (clients ignore unknown), never remove or repurpose (breaks old clients).

**Pagination:**

- **Offset/limit:** `?offset=20&limit=20` (or the equivalent `?page=2&limit=20`). Simple, but skewed by inserts (items shift pages), expensive at deep offsets.
- **Cursor:** `?after=xyz&limit=20`. Stable across inserts, efficient. Cursor encodes position.

**Errors:**

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance is too low",
    "details": { "available": 50, "requested": 100 },
    "request_id": "req-abc-123"
  }
}
```

Structured errors with stable codes. Don't leak internal exception messages or stack traces.

**Authentication:**

- API key in header for service-to-service.
- OAuth2 / OpenID Connect (OIDC) for user-facing.
- Always over TLS.
- Short-lived tokens + refresh, not long-lived bearer tokens.

**Authorization:**

- Authentication establishes identity; authorization decides what that identity may do. Keep the two separate.
- Coarse checks (is the caller a valid, active user or tenant?) belong at the gateway; fine-grained checks (may this user edit *this* record?) belong in the service that owns the data, since only it knows the resource.
- Model permissions with roles (RBAC) or attributes and policies (ABAC). Service-to-service calls authenticate with mTLS or signed tokens, not shared API keys alone.

Identity, token, and mTLS primitives are in `Cryptography.md`.

**Rate limiting headers:**

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1700000000
```

So clients can self-regulate without hitting the limit.

**Idempotency:** see next section.

---

## 9. Idempotency and API Robustness

A network call can fail in three ways: succeeds visibly, fails visibly, "I don't know." The third is the hard one — the client must safely retry.

**An operation is idempotent if applying it multiple times has the same effect as applying it once.**

```
GET, HEAD, PUT, DELETE   typically idempotent
POST                     typically not idempotent
PATCH                    depends
```

A pure GET is naturally idempotent. PUT is idempotent because it sets a resource to a value. POST creates new things — retry creates duplicates.

### Idempotency Keys

The standard pattern for making non-idempotent operations safe to retry.

```
POST /payments
Idempotency-Key: f4a73c2d-...
{ "amount": 100, "to": "user_42" }
```

The server records, against the key, a fingerprint of the request (so a reused key with a *different* body can be rejected) plus the stored response of the first call; retries with the same key return that same response without re-executing the side effect. Store keys with a TTL long enough to cover client retries (hours to a day). A subtlety is the concurrent in-flight race: two requests with the same key may arrive *simultaneously*, before the first has recorded its outcome. Guard against it by atomically reserving the key on first sight (an insert that fails if it exists) and returning `409 Conflict` (or blocking) while the original is still processing. Stripe popularized this; it's now standard for any API that takes money or has side effects.

### Optimistic Concurrency

```
PUT /orders/42
If-Match: "etag-v3"
{ ... }

Server:
  if current_etag == "etag-v3":
    apply update
  else:
    return 412 Precondition Failed
```

Clients fetch with the current version (ETag, version number), submit the update with the version. If another client wrote in between, the conditional fails and the client retries with the new version.

Prevents lost updates without holding locks.

### Pagination Stability

When iterating a long list, items may be inserted or deleted. Cursor pagination is generally stable; offset pagination is not. Document the contract.

### Long-Running Operations

If an operation takes longer than the client's timeout, switch to async:

```
POST /reports
-> 202 Accepted
   Location: /reports/abc-123

GET /reports/abc-123
-> 202 Accepted (still processing)
   ...
-> 200 OK + result (done)
-> 410 Gone (expired)
```

The Operation pattern: every long task becomes a resource with status, progress, result.

---

## 10. Load Balancing

A load balancer distributes incoming requests across multiple backend instances.

```
                 +-------------------+
Clients -------> |  Load Balancer    |
                 +---+-----+-----+---+
                     |     |     |
                  +--v-+ +-v--+ +v---+
                  |inst| |inst| |inst|
                  | 1  | | 2  | | 3  |
                  +----+ +----+ +----+
```

### Layers

- **L4 (transport):** balances TCP/UDP connections. Doesn't look at HTTP. Fast (just routing). Examples: AWS NLB, HAProxy in TCP mode.
- **L7 (application):** balances HTTP requests. Can route by path, host, header, method. Examples: NGINX, Envoy, HAProxy in HTTP mode, AWS ALB.

### Algorithms

- **Round-robin:** rotate through backends. Doesn't account for backend load.
- **Least connections:** new requests go to the backend with the fewest active connections. Better when backend processing time varies.
- **Weighted:** unequal capacities (heterogeneous instances).
- **Consistent hashing:** the same client/key always goes to the same backend. Used when state is partitioned (cache affinity).
- **Random with two choices:** pick two random backends, send to the less-loaded one. Nearly as good as tracking global load, at far lower cost.

### Health Checks

The LB periodically pings each backend; unhealthy ones are removed from rotation.

```
GET /healthz
-> 200 OK if healthy
-> 503 if not
```

Distinguish:

- **Liveness:** "is this process alive?" (kill if not).
- **Readiness:** "is this process ready to serve traffic?" (remove from LB if not, e.g., during startup or graceful shutdown).

A common bug: liveness checks that depend on downstream health — if a downstream is briefly down, the LB kills all backends.

### Sticky Sessions

The LB sends a given user's requests to the same backend (cookie or hash). Useful when backends keep session state in memory. Anti-pattern at scale — prefer stateless backends + external session store.

### Topology

Real systems usually have layers:

```
Internet -> CDN edge -> Regional load balancer -> Per-AZ load balancer -> Backend instances
```

Each layer hides the next, distributes load, and can apply policies (TLS termination, WAF, rate limiting, caching). The WAF is a web application firewall — it filters malicious HTTP requests before they reach the backend.

An **Availability Zone (AZ)** is one or more discrete, isolated datacenters within a cloud region, with their own power, cooling, and network; a region (e.g. `us-east-1`) is several AZs. The two load-balancing layers do different jobs: the **regional LB** chooses *which AZ* a request goes to (balancing across zones and routing around a failed one), and the **per-AZ LB** chooses *which backend instance within that zone*. Keeping a request inside one AZ once it lands avoids cross-AZ network hops, which add latency and are usually billed.

---

## 11. Caching as a Pattern

Caching at the architecture level (not just the data layer — covered in `Data Systems.md`).

### Where to Cache

```
Layer          Cache at this layer
-----          -------------------
Client      -> Browser cache
   |
CDN edge    -> CDN / edge cache
   |
LB / gateway-> Gateway (reverse-proxy) cache
   |
App         -> In-process cache, distributed cache (Redis)
   |
DB          -> DB internal cache (page cache, buffer pool)
```

Cache closer to the client = lower latency, less load downstream. But: shorter cache horizon (less staleness tolerance), more invalidation complexity.

### What to Cache Where

| Layer | Best for |
|-------|----------|
| Browser | Static assets, user-specific content with short TTL |
| CDN | Static or shared content (images, CSS, JS, public API responses) |
| Gateway | Rate-limit results, auth lookups |
| App in-process | Hot lookups (configuration, small directories) |
| Distributed (Redis) | Cross-instance shared state, session data |
| DB | Query results (rarely application-managed) |

### Invalidation Strategies (Recap)

- **TTL:** simple, accepts staleness.
- **Explicit invalidation:** write triggers cache delete.
- **Versioned keys:** new content, new key (immutable).
- **Stale-while-revalidate:** serve stale, refresh in background.

### Cache Stampede Prevention

When a hot key expires, thousands of requests miss the cache simultaneously and hit the backend.

- **Lock / single-flight:** only one request refreshes; others wait or serve stale.
- **Probabilistic early refresh:** refresh before TTL with some probability.
- **Always-warm:** background job refreshes hot keys before they expire.

### When NOT to Cache

- Personalized content for individual users (low cache hit rate).
- Data that must always be fresh (financial balances, inventory at checkout).
- Small, cheap-to-compute results (the cache lookup may cost as much as the computation).

---

## 12. Rate Limiting and Backpressure

### Why Rate Limit

- Protect from abuse (DDoS, scraping, credential stuffing).
- Fair allocation (one tenant doesn't starve others).
- Cost control (avoid runaway expensive operations).
- Capacity planning (cap the load on downstream services).

### Algorithms

**Fixed window:**

```
Time:   [0,60s)    [60s, 120s)   [120s, 180s)
Count:    100         100            100
```

Simple but can allow bursts at window boundaries.

**Sliding window log:**

Keep a log of timestamps; count entries in the last N seconds. Precise but memory-intensive.

**Sliding window counter:**

```
Estimated rate = current_window_count + 
                 prev_window_count * (1 - elapsed_in_current / window_size)
```

Smoother than fixed window; cheap. The estimate assumes the previous window's requests were spread uniformly, so it can misjudge a burst concentrated at the window's edge.

**Token bucket:**

```
Tokens accumulate at rate R (e.g., 10 per second), up to a max (burst).
Each request consumes 1 token.
If no tokens, reject or queue.
```

Allows bursts up to the bucket size; sustained rate = R.

**Leaky bucket:**

```
Requests queue up. Drain at rate R.
If queue full, reject.
```

Smooths bursts to a constant output rate.

### Where to Implement

- **Edge (CDN, gateway):** cheap rejection of bad traffic before it reaches the stack.
- **API gateway:** per-API-key, per-user limits.
- **Application:** business-logic rate limits (e.g., max emails per hour per user).
- **Downstream caller side:** backpressure to internal services.

### Communicating Rate Limits

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700000000
```

`Retry-After` tells clients when to try again. Good clients respect it; bad clients don't, in which case the rate limiter just keeps rejecting.

### Backpressure

When a downstream is slow, upstreams shouldn't pile up requests indefinitely.

- **Reject early:** under load, return 503 quickly rather than queueing.
- **Bounded queues:** never use unbounded queues. Set a max; drop or shed when full.
- **Circuit breakers:** when a downstream is failing, fail fast and recover gradually.
- **Adaptive concurrency limits:** measure success rate and latency, adjust concurrency dynamically (Netflix Concurrency Limits, Vegas algorithm).

---

## 13. Building Blocks — Common Patterns

The recurring micro-architectures common to system design.

### URL Shortener / ID Generator

Generate short, unique IDs at scale.

**Approaches:**

- **Database autoincrement:** one DB, won't scale to many writers.
- **UUID:** random, no coordination, but long.
- **Snowflake-style:** 64-bit ID = timestamp + machine ID + sequence. Coordinated only at machine ID level.

```
Snowflake (64 bits):
   1 bit : unused (sign bit kept 0 so the ID stays positive)
  41 bits: timestamp (ms since a custom epoch)
  10 bits: machine ID
  12 bits: sequence within ms

  2^41 ms  ~ 69 years before the timestamp overflows
  2^10     = 1024 machines generating concurrently
  2^12     = 4096 IDs per machine per ms (counter resets each ms;
             if exhausted, wait for the next ms)
```

The only thing machines must agree on is their machine ID, assigned once at startup. After that each generates IDs alone — local clock, local counter, no lock or network hop on the hot path. Because the timestamp sits in the high bits, larger ID ≈ newer, so IDs are roughly time-sortable (useful as DB primary keys; random UUIDs are not).

**From ID to short URL:** a 64-bit integer is long in decimal, but encoding it in **base62** (`0-9a-zA-Z`, 62 symbols per character) collapses it to at most 11 characters (a full 64-bit value), fewer for smaller IDs — e.g. a mid-range ID might encode as `dBszT2a`. `short.url/dBszT2a` decodes back to the integer, which is the lookup key. The generator produces the unique number; base62 makes it short and URL-safe.

### Rate Limiter (Detailed Design)

Token bucket per key, stored in Redis:

```
Each user/IP has a bucket: {tokens: int, last_refill: timestamp}

On request:
  Lua script in Redis (atomic):
    Read bucket
    Refill based on time since last_refill
    If tokens >= 1:
      tokens -= 1, allow
    Else:
      reject
    Save bucket
```

The Lua script ensures atomicity (read-modify-write in one Redis call).

### Notification System

Send emails / push / SMS reliably.

```
App publishes "send_email" event
       |
       v
Kafka topic "notifications"
       |
       v
+----------------+
| Worker pool    |    each worker pulls from topic
|  + retry queue |    on failure: backoff, re-enqueue
|  + DLQ         |    on permanent failure: dead-letter
+----------------+
       |
       v
SES / Sendgrid / FCM
```

Properties:

- Decouples producer from delivery (producer doesn't wait).
- At-least-once delivery (consumer idempotent).
- Multiple delivery channels can branch off the same event.

### Feed / Timeline

For each user, show recent posts from followed users.

**Pull (fan-out on read):** at read time, query each followee's recent posts. Cheap writes; expensive reads especially for users following many.

**Push (fan-out on write):** when a user posts, fan out to each follower's timeline cache. Cheap reads (a single lookup in the user's own cache); expensive writes especially for celebrities.

**Hybrid (Twitter's approach):** push for most users; pull for celebrities (so a celebrity post doesn't fan out to 100M caches).

```
On post:
  If poster.followers < THRESHOLD:
    push to each follower's timeline
  Else (celebrity):
    don't fan out — followers will pull on read

On timeline read:
  Get pushed posts from user's timeline cache
  Plus pull recent posts from celebrities they follow
  Merge by timestamp
```

### Search Autocomplete

Real-time suggestions generated as the user types.

```
+--------+    +-------+    +---------+    +--------+
| Client | -> | Edge  | -> | Suggest | -> | Trie / |
|        |    | cache |    | service |    | Redis  |
+--------+    +-------+    +---------+    +--------+
```

- **Trie or DAWG (directed acyclic word graph)** for prefix lookup.
- Pre-computed top-K suggestions per prefix (ranked by popularity / recency).
- Aggressive caching at the edge (suggestions are highly cacheable per prefix).

### Distributed Counter

Cassandra/DynamoDB counters, or many shards:

```
Counter "page_views_for_X" is sharded into 100 sub-counters.
On increment: pick a random shard, increment.
On read: sum all 100 shards.

Write contention reduced 100x; read cost up by 100x but still cheap.
```

### Leaderboard

Top-K by score, updated frequently.

A **sorted set** stores members each tagged with a numeric score and keeps them ordered by that score automatically on every write — exactly the structure a leaderboard needs.

```
Redis sorted set (Z* commands are the sorted-set family):
  ZADD leaderboard 4500 alice             -- add/update alice's score
  ZREVRANGE leaderboard 0 99 WITHSCORES   -- top 100 (REV = highest first)
  ZREVRANK leaderboard alice              -- alice's current rank

For millions of entries: shard by region or category;
periodic merge into a global leaderboard (ZUNIONSTORE).
```

Why not SQL `ORDER BY score LIMIT 100`: the set stays ordered as writes land, so reading the top-K is `O(log N + K)` and a score update is `O(log N)` — no re-sort or index scan per read. Redis holds it in memory, which is what makes frequent updates cheap.

### Distributed Job Scheduler

Cron at scale (e.g., "run every 5 min for every user").

```
- Time-sharded job table: shard by execution time.
- Workers poll their shard.
- Coordinated assignment (each job claimed by one worker via DB lock or stream).
- Idempotent jobs: safe to re-run on worker failure.
```

Tools: Kubernetes CronJobs (simple), Temporal/Cadence (workflows), Apache Airflow (DAGs). A DAG is a directed acyclic graph — here, tasks wired by their dependencies.

### Image / File Upload Pipeline

```
Client -> Get presigned URL from API -> PUT directly to S3
                                             |
S3 event notification              ----------+
        |
        v  (via a broker: SQS / SNS / EventBridge / Kafka)
Lambda / worker: resize, generate thumbnails, scan for malware,
                 update DB record, send notification
```

Direct uploads remove the app server from the bandwidth path, and routing the "object created" event through a broker rather than a direct call decouples upload from processing (the event-driven benefits of section 5: burst absorption, retries/DLQ, and free fan-out to extra subscribers like a thumbnailer or malware scanner).

### Distributed Locking

`Redis SET NX EX <ttl>`, ZooKeeper ephemeral node, etcd lease. Use carefully (see `Distributed Systems.md` for the fencing-token caveat).

### Service Discovery

DNS round-robin (simple, cached), Consul/etcd (stateful registry), Kubernetes Services (DNS + iptables/IPVS), service mesh (Istio).

---

## 14. Scaling Strategies

### Vertical Scaling (Scale Up)

Bigger machines. Simple. Limited by single-machine ceilings, but those ceilings are high — 128 cores and 1 TB RAM are routine.

**When:** the workload fits on one machine, simplicity is valuable, and the ceiling has not been reached. Often the right answer for years.

### Horizontal Scaling (Scale Out)

More machines. Requires the workload to be parallelizable.

**Stateless tier:** trivial to scale. Add more instances behind a load balancer. The default for stateless API/web servers.

**Stateful tier:** scale by sharding (Section 8 of `Data Systems.md`), replication, or both.

### Read Scaling vs Write Scaling

- **Read scaling:** read replicas. Each replica serves reads; the leader handles writes. Scales reads linearly with replicas.
- **Write scaling:** shard the dataset. Writes distributed across shards. Each shard a complete sub-system with its own replicas.

### Compute Scaling Patterns

- **Autoscaling:** instances added/removed based on CPU, RPS, queue depth. Standard cloud feature.
- **Scale to zero:** during idle, no instances run. Cold start when first request arrives. Used in serverless and edge functions.
- **Reserved capacity:** for known baseline; spot/on-demand for spikes.

---

## 15. Reliability and Resilience

### SLO / SLA / SLI

- **SLI (Indicator):** the measured quantity (e.g., latency, error rate).
- **SLO (Objective):** target value for the SLI (e.g., "99.9% of requests under 500ms").
- **SLA (Agreement):** external commitment, usually with consequences (refunds) if missed.

Internal SLOs are typically tighter than SLAs, to preserve headroom.

### Error Budget

If SLO is 99.9%, the error budget is 0.1% — about 43 min of downtime per month. Used to balance reliability vs feature velocity: if the budget is intact, ship features fast; if exhausted, focus on reliability.

### Failure Modes to Plan For

```
Single instance failure
Instance group / AZ failure
Region failure
Dependency failure (DB, downstream service)
Cascading failure (one slow thing slows everything)
Bad deploy
Data corruption
Bad config push
Misbehaving client
DDoS / traffic spike
```

For each, ask: how does the system behave? Is degradation graceful?

### Graceful Degradation

Better to deliver a worse experience than no experience.

```
Search engine slow?            Return cached results, mark them stale.
Recommendation service down?   Show generic popular items.
Logging downstream broken?     Buffer; drop oldest if buffer overflows.
Database read replica behind?  Serve from leader.
```

Don't fail the whole request when a non-critical dependency fails.

### Bulkheads

Isolate failures. Thread pool per downstream; connection pool per database. One slow dependency exhausts its own pool, not everything.

### Circuit Breakers

When a downstream is failing, stop sending requests to it for a window. Saves the downstream from being overwhelmed and saves the upstream from waiting on timeouts.

```
States:
  Closed:    requests pass through, errors counted
  Open:      requests rejected immediately for cooldown period
  Half-open: a few probe requests; if they succeed, close; else re-open
```

Hystrix popularized; libraries exist for every language (resilience4j, polly, gobreaker).

### Timeouts Everywhere

Every external call has a finite timeout. Default "wait forever" is the seed of cascading failure.

Choose timeouts shorter than the upstream's timeout — leave room for retries.

### Retries with Backoff and Jitter

```python
def with_retry(fn, attempts=5):
    for i in range(attempts):
        try: return fn()
        except RetriableError:
            if i == attempts - 1:           # final attempt: don't sleep, just fail
                raise
            sleep_for = min(30, 0.5 * 2**i * random.uniform(0.5, 1.5))
            sleep(sleep_for)
```

Critical: jitter prevents synchronized retry storms.

### Idempotency

Retries are only safe if operations are idempotent. See section 9.

### Tail Latency and Fan-Out

A request that fans out to many backends and waits for all of them is as slow as the slowest response, not the average. If each of N backends has a p99 of 10 ms, the chance that at least one lands in its slow 1% grows with N, so the *combined* latency drifts toward the backend tail — the slowest responses dominate. Scatter-gather reads, GraphQL resolvers, and per-shard queries all have this shape.

Mitigations: keep fan-out width bounded, return partial results when a laggard misses its deadline, and issue **hedged requests** — send a duplicate to a second replica once the first is slow and take whichever returns first. Hedging trades a little extra load for a much tighter tail. The mechanics are in `Performance Optimization.md`; measuring tail percentiles is in `Observability.md`.

### Chaos Engineering

Inject failures to verify resilience. Pioneered by Netflix (Chaos Monkey kills random instances). Modern tools: Gremlin, Litmus, Chaos Mesh.

Regular game days (planned chaos exercises) build confidence and surface gaps.

---

## 16. Multi-Region and Geo

### Why Multi-Region

- **Latency:** users in Tokyo shouldn't hit a US server. Latency matters more than throughput for UX.
- **Compliance:** data residency (GDPR, China, India).
- **Disaster recovery:** region-level outages happen.
- **Capacity:** spread load.

### Patterns

**Active-passive:** one primary region, secondary on standby. Failover requires DNS change or routing change. Recovery time objective (RTO) measured in minutes.

**Active-active:** both regions serve traffic. Hard problem: writes happening in both regions.

```
Active-active write paths:
  - Single writer per partition (route by key to home region; slow for the away region).
  - Multi-leader with conflict resolution (LWW or CRDTs or app-specific merge).
  - Synchronous cross-region writes (slow but consistent — Spanner).
```

**Geo-partitioning:** users assigned to a home region by location/contract. Writes happen in the home region; reads can happen locally (with replication lag) or cross-region (with latency).

### Cost of Cross-Region

```
Within region: < 1 ms RTT
Within continent: 20-100 ms RTT
Cross-continent: 100-300 ms RTT

Cross-region data transfer: billed per GB, a significant recurring cost
```

A synchronous request that crosses regions on every hot path is a non-starter for most workloads.

### Globally Distributed Stores

- **DynamoDB Global Tables:** multi-region, last-write-wins.
- **Spanner:** strongly consistent across regions (paying TrueTime commit-wait cost).
- **CockroachDB / YugabyteDB:** geo-partitioned with strong consistency where pinned, eventual where replicated.
- **Cassandra:** eventually consistent across DCs, tunable consistency per query.
- **Aurora Global Database:** cross-region read replicas with managed failover.

### Edge / CDN

Push as much as possible to the edge:

- Static assets via CDN.
- Cacheable API responses via edge caching.
- Edge compute (CloudFlare Workers, Lambda@Edge, Vercel) for request transformation.

The edge is the fastest tier; minimize what needs to traverse to the origin.

---

## 17. Cost as a Design Constraint

Cost shapes architecture more than is usually acknowledged. The "best" design is often the cheapest one that meets the requirements.

### Cost Categories

```
Compute           cores * time
Storage           GB * months (and tier — Standard, IA, Glacier)
Egress            GB transferred out of cloud
Inter-region/AZ   GB across regions or zones
API calls         per request to managed services (S3, DynamoDB, KMS)
Logs/metrics      ingested or stored data
Support and licensing
```

Egress is the line item teams most often underestimate: outbound data transfer is billed per GB, so a high-traffic service moving many TB per month accrues a large, easily overlooked bill.

### Patterns That Save Money

- **Use managed services for low-volume work** (Lambda, RDS, DynamoDB).
- **Use VMs for high-volume work** (Lambda gets expensive past a threshold).
- **Reserved/savings plans** for predictable baseline.
- **Spot instances** for fault-tolerant batch work.
- **Compression** before storing or transmitting.
- **Tiered storage:** hot data in fast tier, cold in archive.
- **Pre-aggregate** in analytics — don't scan billions of rows for every dashboard.
- **CDN for repeatable content** — cache hits at the edge are far cheaper per GB than origin egress.

### Patterns That Cost Money

- Cross-AZ chatter in microservices.
- High-cardinality metrics with unbounded retention.
- Excessive logging at INFO level.
- Encrypting and decrypting in KMS for every record (use envelope encryption).
- Synchronous replication across regions.
- Over-provisioned reserved capacity.

Tracking cost per feature or per tenant uncovers surprises.

---

## 18. Evolving Systems

Real systems are not designed; they evolve. The skill is shaping evolution.

### Strangler Fig Pattern

Replace a legacy system incrementally. Route some traffic to the new system; expand coverage over time; eventually remove the legacy.

```
Stage 1: 100% traffic to legacy
Stage 2: 1% to new (canary)
Stage 3: 50% to new
Stage 4: 100% to new
Stage 5: turn off legacy
```

Each stage is reversible. No big-bang migration.

### Decommissioning

When removing a feature/service:

- Identify all callers.
- Announce deprecation with timeline.
- Add metrics on remaining callers.
- Migrate or shut down callers.
- Final removal only when call count is zero.

### Schema Migration

Multi-deploy migrations (expand-then-contract):

1. Add the new schema/column (backward-compatible).
2. Write to both old and new in code.
3. Backfill the new from the old.
4. Switch reads to the new.
5. Stop writing to the old.
6. Drop the old.

Each step is independently deployable. See `Data Systems.md` section 9.

### Documentation as Architecture

The system that exists is not what the diagram shows. Keep:

- **System architecture diagram** (top-level boxes and arrows).
- **Service catalog** (what each service does, who owns it).
- **Runbooks** for common operations.
- **Postmortems** for past incidents.
- **ADRs (Architecture Decision Records):** the *why* of significant decisions.

Without these, every refactor starts from zero.

### Refactoring vs Rewriting

Greenfield rewrites are usually wrong. They:

- Lose tribal knowledge.
- Take longer than expected.
- Run in parallel with the legacy for months, paying double cost.
- Have new bugs the legacy didn't have.

Strangler-fig refactoring is usually right. Replace components incrementally; never have two competing systems for the same feature.

### The Hardest Part

Most systems are designed once, then evolve under different requirements than they started with. The hardest skill in system design is **not designing the perfect system today** — it is designing a system that can be changed tomorrow, when the unpredictable requirements finally arrive.

Choose boring technology. Keep moving parts to a minimum. Make boundaries explicit. Optimize for unpredictable change by minimizing the surface area of irreversible decisions.
