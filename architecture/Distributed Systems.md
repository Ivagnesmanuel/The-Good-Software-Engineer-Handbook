# Distributed Systems

A reference covering the mechanisms behind systems that span multiple machines: real systems and operational patterns (Dynamo, Spanner, Raft, ZooKeeper, CRDTs, sagas, idempotency) alongside the formal abstractions that underpin them (timing models, links, broadcast, ordered delivery, registers, consensus, Byzantine fault tolerance). 

---

## Table of Contents

1. Overview
2. System Models and Abstractions
3. The Eight Fallacies
4. Failure Models
5. Link Abstractions
6. Time, Clocks, and Ordering
7. Distributed Mutual Exclusion
8. Broadcast Abstractions
9. Ordered Delivery and Total-Order Broadcast
10. Consistency Models
11. CAP and PACELC
12. Replication and Partitioning
13. Quorums
14. Distributed Registers
15. Consensus (Paxos, Raft, ZAB)
16. Leader Election
17. Failure Detection and Gossip
18. Byzantine Fault Tolerance
19. Distributed Transactions
20. Exactly-Once and Idempotency
21. Distributed Coordination Primitives
22. Publish/Subscribe and Event Routing
23. Blockchain and Distributed Ledgers
24. Common Patterns and Anti-Patterns

---

## 1. Overview

A distributed system is one in which components on networked machines coordinate to appear, from the outside, as a single coherent system. The defining characteristic is not "many computers" but **independent failures**: each component can crash, restart, slow down, or partition from the others, and the system must continue functioning.

Leslie Lamport's definition captures it: *"A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable."*

The motivation is twofold: **performance** (aggregate the resources of many machines) and **dependability** (build services that survive the failure of individual components). A dependable service is characterized by availability (readiness for correct service), reliability (continuity of correct service), safety (absence of catastrophic consequences), integrity (absence of improper alteration), and maintainability.

### Why Distributed Systems Are Hard

A single-machine program has a single failure boundary: either it's running or it isn't. A distributed system has every component as a partial failure boundary, plus the network between them.

```
Single machine:           Two-machine system:
                          
    +--------+                +--------+      +--------+
    |  app   |                |  app A | ---> |  app B |
    +--------+                +--------+      +--------+
                                  ^             |
      running                     |             |
      OR                          +-------------+
      not running                       network
                              
                              A and B can each be:
                                running, slow, crashed, partitioned
                              The network can be:
                                fast, slow, lossy, partitioned, asymmetric
                              
                              State space explodes combinatorially.
```

From one side of a network, it is impossible to tell whether the other side has:

- Received the message but not replied yet (slow).
- Received it and replied, but the reply was lost (lost ack).
- Not received it (lost request).
- Received and processed it, but is now partitioned.
- Crashed before processing.
- Crashed after processing.

Whether the other side performed the requested action is, in general, **unknowable**.

This single fact — partial information under uncertainty — is the source of everything difficult about distributed systems.

### Two Goals: Availability and Consistency

Distributed systems are typically built for one or both of:

- **Availability:** the system keeps responding even when some nodes fail.
- **Consistency:** every client sees a coherent, agreed-upon view of state.

Under network partitions both cannot hold at full strength — see CAP. The task is choosing the right tradeoffs for the use case.

### Coordination Cost

Coordination — agreement between nodes — is expensive. Every "consistent" operation requires a round trip (often several). A system that needs strong consistency on every operation is bounded by network latency and the number of failures it must tolerate.

The fastest distributed systems coordinate as little as possible. Strong consistency exists in the parts that genuinely need it; everywhere else uses eventual consistency, idempotency, and locally-deterministic algorithms.

---

## 2. System Models and Abstractions

Reasoning about a distributed system requires fixing three things: what the **processes** can do, what the **links** between them can do, and what **timing** assumptions hold. The formal layer of this document is built on these choices; changing any one of them changes which guarantees are achievable.

### Processes, Events, and Histories

A system is a set of processes `P = {p1, ..., pn}`, each running on its own machine with no shared memory, communicating only by messages. Each process has a local state changed by **events**:

- **Internal events:** transform local state.
- **Send / receive events:** the only external interactions.

The sequence of events at one process is its **local history**; a prefix of it is a **partial local history**; the set of all local histories is the **global history**. Within a single process, events are totally ordered. Across processes, only causality (section 6) gives a partial order — there is no global clock to impose a total one.

### Safety and Liveness

Every abstraction is specified by two kinds of property:

- **Safety:** "nothing bad ever happens." Formally, if a safety property is violated in an execution, it is violated at some finite point, and no extension can repair it (a bad thing, once done, stays done). Examples: no two leaders; no read returns an uncommitted value.
- **Liveness:** "something good eventually happens." For any point in time, there remains hope the property is satisfied later. Examples: every request eventually receives a response; every correct process eventually decides.

The distinction matters because the two are traded against each other under failure. Most protocols **prioritize safety** — under bad conditions they stall (lose liveness) rather than return wrong answers (lose safety). Stalling is recoverable; corruption is not.

### Timing Assumptions

```
Model                  Assumption                        Consequence
-------------------    -------------------------------   --------------------------
Synchronous            Known bounds on message delay,    Easiest; timeouts are
                       processing time, and clock drift  reliable failure signals
Asynchronous           No timing bounds at all           Hardest; no timeout can
                                                         distinguish slow from dead
Partially synchronous  Bounds exist but are unknown,     Realistic; the model real
                       or hold only after some unknown   systems target
                       time t (eventual synchrony)
```

The synchronous model is the friendliest but rarely realizable on real networks. The asynchronous model is the most faithful but provably too weak for some problems (see FLP, section 15). **Partial synchrony** is the practical middle ground: the network misbehaves for a while, then settles long enough for a protocol to make progress. Real systems encode their timing assumptions either directly (wait `δ` before delivering) or in a separate **oracle** abstraction — a failure detector (section 17) or leader elector (section 16) — that hides the timing assumptions behind a clean interface.

### The Composition Model

Abstractions are layered. Each is a component reacting to events on its interface:

- **Request:** an upper layer asks this component for a service.
- **Indication:** this component delivers information (often a completed service) upward.
- **Confirmation:** this component confirms a requested service completed.

A component implements its properties by issuing requests to the component beneath it and handling that component's indications — for example, a perfect link is built on a stubborn link, which is built on a fair-loss link (section 5). Algorithms are written as **event handlers**:

```
upon <Component, Event | args> do
    ...local state updates...
    trigger <LowerComponent, Request | ...>
```

This is the notation used throughout the formal sections: `upon` an incoming event, update state and `trigger` outgoing events. A distributed algorithm is one such handler set per process; an execution is an interleaving of their steps.

---

## 3. The Eight Fallacies

Peter Deutsch's classic list of false assumptions distributed programmers make. Every one of these has caused production outages.

```
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.
```

Translating each into concrete consequences:

| Fallacy | Reality | Consequence |
|---------|---------|-------------|
| Network is reliable | Packets drop, connections die | Need timeouts, retries, idempotency |
| Latency is zero | Round-trip times (RTTs) are measured in ms, geo links in 100s of ms | Avoid chatty protocols, batch, cache |
| Bandwidth is infinite | Pipes are finite; congestion is real | Compress, paginate, avoid sending unnecessary data |
| Network is secure | Anyone on the path can read or modify | Encrypt everything; assume the network is hostile |
| Topology doesn't change | Hosts move, IPs change, DNS lies | Service discovery; don't cache addresses forever |
| One administrator | Many teams, many policies | Negotiate compatibility, version APIs |
| Transport cost is zero | Egress fees, cross-availability-zone (AZ) charges | Co-locate when latency or cost matters |
| Network is homogeneous | TCP behaves differently across paths | Test on real network conditions, not loopback |

These look obvious in a list. They are easy to violate one assumption at a time during design.

---

## 4. Failure Models

What can go wrong, in increasing severity:

```
Failure model     Node behavior                     Difficulty to handle
---------------   -------------------------------   ------------------------------
Crash-stop        Halts, never resumes              Easiest: failure is permanent
Crash-recovery    Halts, may restart with state     Log replay, re-join membership
Omission          Drops some messages               Silent; needs timeouts and acks
Performance       Keeps running but slow            Indistinguishable from a crash
Byzantine         Arbitrary, possibly malicious     Hardest: requires BFT protocols
```

**Crash-stop:** A node either works correctly or stops. Simplest model. Many algorithms designed for this.

**Crash-recovery:** A node may crash, restart, and rejoin with persisted state. Adds complications around log replay and re-establishing membership.

**Omission:** Messages are silently dropped. The sender doesn't know. Mitigated by timeouts and acks.

**Performance / partial failure:** A node continues running but processes slowly, intermittently, or with degraded throughput. From the outside, indistinguishable from network problems or partial crash. **This is the most common real-world failure** and the hardest to design for.

**Byzantine:** Nodes can do anything, including lie, send conflicting messages, or actively try to undermine consensus. Required for blockchains and fault tolerance in adversarial environments. Algorithms are dramatically more expensive (PBFT, HotStuff). Most non-adversarial distributed systems assume non-Byzantine.

Note the distinction between a **fault** (the underlying defect), an **error** (the resulting incorrect state), and a **failure** (the externally visible deviation from correct service): a fault activates into an error, which propagates into a failure, which may cause further faults elsewhere.

### What "Tolerating N Failures" Means

A protocol that "tolerates F failures" means: with F or fewer concurrent failures (of the given type), the protocol still satisfies its safety and liveness properties.

- **Safety:** something bad never happens (no two leaders, no inconsistent reads).
- **Liveness:** something good eventually happens (a request eventually completes).

Under enough failures, liveness can be lost (the system stops responding) without losing safety (the system never returns wrong answers). Most consensus protocols favor safety over liveness — stalling is preferred to corruption.

The failure type sets the quorum size. Crash faults need `N >= 2f + 1` (a majority survives). Byzantine faults need `N >= 3f + 1` (a majority of *correct* nodes survives even after the faulty ones lie) — see section 18.

---

## 5. Link Abstractions

The network is modeled as **links** connecting pairs of processes. Messages can be lost, delayed arbitrarily, or duplicated, and processes can crash. Stronger link abstractions are built on weaker ones, hiding these defects. Each link exposes two events: a **send** request and a **deliver** indication. Delivery is deliberately distinct from raw network *receipt*: a message is received into a buffer, then the link's algorithm runs before the message is *delivered* upward.

```
Authenticated Perfect Link   (adds: integrity under Byzantine senders)
        ^
Perfect Link                 (reliable delivery, no duplication, no creation)
        ^
Stubborn Link                (retransmits forever; reliable but duplicating)
        ^
Fair-Loss Link               (the raw network: may lose, may duplicate)
```

### Fair-Loss Link

The raw network, equivalent to UDP. Loses messages with some probability `p != 0`.

- **Fair loss:** if a correct `p` sends `m` to a correct `q` infinitely often, `q` delivers `m` infinitely often. (Persistent retransmission eventually gets through.)
- **Finite duplication:** if `p` sends `m` a finite number of times, `q` delivers it only a finite number of times.
- **No creation:** every delivered message was actually sent.

### Stubborn Link

Built on fair-loss to guarantee eventual delivery, by retransmitting every message forever.

- **Stubborn delivery:** if a correct `p` sends `m` once to a correct `q`, then `q` delivers `m` an infinite number of times.
- **No creation:** as above.

```
upon <StubbornLink, Send | q, m> do
    add m to sent
    trigger <FairLoss, Send | q, m>

upon <Timeout> do                       # fires periodically, forever
    forall m in sent do
        trigger <FairLoss, Send | q, m>
```

Correct but wasteful: it never stops retransmitting and delivers duplicates.

### Perfect Link

Built on the stubborn link by filtering duplicates. The abstraction most algorithms assume.

- **Reliable delivery:** if a correct `p` sends `m` to a correct `q`, then `q` eventually delivers `m`.
- **No duplication:** no message is delivered more than once.
- **No creation:** as above.

```
upon <PerfectLink, Send | q, m> do
    trigger <StubbornLink, Send | q, (id, m)>

upon <StubbornLink, Deliver | p, (id, m)> do
    if id not in delivered then
        add id to delivered
        trigger <PerfectLink, Deliver | p, m>
```

A naive perfect link still retransmits forever underneath. A **quiescent** implementation stops once it knows the message arrived (via an ack), which is what TCP approximates in practice.

### Authenticated Perfect Link

Adds cryptographic integrity so a Byzantine process on the path cannot forge or tamper with messages (a man-in-the-middle). Each message carries a MAC (Message Authentication Code) or signature the receiver verifies. This is the base layer for all Byzantine-tolerant abstractions (section 18). It defends message *integrity* only — a Byzantine *sender* can still sign and send malicious-but-well-formed content.

---

## 6. Time, Clocks, and Ordering

In a distributed system, the question "when did X happen?" has no globally consistent answer. Clocks drift, NTP (Network Time Protocol) is approximate, network delay is unpredictable.

### Wall Clocks (Time-of-Day)

A node's NTP-synced clock is typically accurate to a few milliseconds, possibly seconds. **Wall clocks can go backwards** (NTP correction, leap seconds). Code that assumes monotonic wall time is buggy.

**Useful for:** timestamps for humans, expiry checks with significant tolerance, debugging.

**Not useful for:** ordering events across machines, measuring durations, anything safety-critical.

### Monotonic Clocks

A clock that only moves forward, at approximately real-time speed. Available on every modern OS (`CLOCK_MONOTONIC` on Linux, `time.monotonic()` in Python, `Instant` in Rust). **Use for measuring elapsed time and for timeouts.**

Monotonic clocks are local — node A's monotonic clock can't be compared with node B's.

### Theory: Physical Clock Synchronization

A software clock approximates real time as `C(t) = a * H(t) + b`, where `H` is the hardware clock and `a, b` are correction factors chosen to keep it monotonic and close to UTC. Two parameters degrade it:

- **Skew:** the instantaneous difference between two clocks, `|Ci(t) - Cj(t)|`.
- **Drift:** the gradual divergence of once-synchronized clocks from accumulated rate error. Ordinary quartz drifts about 1 second per 12 days.

A hardware clock is *correct* if its drift is bounded: `1 - p <= dC/dt <= 1 + p`. Synchronization comes in two flavors against a bound `D`: **external** (each `Ci` tracks a UTC source `S`, so `|S(t) - Ci(t)| < D`) and **internal** (processes agree with each other, `|Ci(t) - Cj(t)| < D`). External synchronization implies internal synchronization with bound `2D`; the converse does not hold.

**Cristian's algorithm** (external). A process asks a time server `S`, which replies with timestamp `t`. The process measures round-trip time `RTT` and sets its clock to `t + RTT/2`, assuming the reply path took half the round trip. Accuracy is `±(RTT/2 - min)`, where `min` is the minimum one-way delay — so it is only as good as the network is fast and symmetric. It works probabilistically even in asynchronous systems, but the server is a single point of failure and trust.

**Berkeley algorithm** (internal, no UTC source). A master polls every node's clock, computes each offset (correcting for round-trip delay), averages the offsets while discarding outliers (faulty clocks), and sends each node a correction. Negative corrections are applied by *slowing* a clock rather than stepping it backward, to preserve monotonicity and causal order.

**NTP** is the internet-scale external-synchronization standard. It uses Cristian-style remote reads layered over a reconfigurable **stratum hierarchy** (primary servers on UTC sources, secondaries below them), plus filtering and clustering to reject bad samples. It operates in multicast, procedure-call, and symmetric modes.

The fundamental limit: **physical synchronization is meaningless in a truly asynchronous system**, because unbounded message delay makes the round-trip estimate useless. Different clocks can only be synchronized with a certain probability — which is exactly why logical clocks exist.

### Logical Clocks (Lamport)

A counter that respects causality without depending on physical time.

```
Each node maintains a counter L.

On any local event:    L = L + 1
On sending a message:  L = L + 1; attach L to the message
On receiving (m, L_m): L = max(L, L_m) + 1
```

**Property:** if event A happened before event B (in the causal sense), then `L(A) < L(B)`.

The converse is not true — `L(A) < L(B)` doesn't mean A happened before B. They might be concurrent. So a scalar Lamport clock can *order* events but cannot *detect concurrency*.

Used to define a **total order** over events: pick the one with the smaller Lamport timestamp; break ties by node ID. Useful for ordering, less useful for "is X causally before Y?"

### Vector Clocks

A vector with one entry per node. Tracks how many events each node has seen. Unlike scalar clocks, vector clocks capture concurrency exactly: `A -> B iff V(A) <= V(B)` componentwise with at least one strict inequality; otherwise A and B are concurrent.

```
Three nodes A, B, C. Each has a 3-entry vector.

Initial:   A: [0, 0, 0]   B: [0, 0, 0]   C: [0, 0, 0]

A does an event:        A: [1, 0, 0]
A sends msg to B:       msg carries [1, 0, 0]
B receives, increments: B: [1, 1, 0]

B does an event:        B: [1, 2, 0]
B sends to C:           msg carries [1, 2, 0]
C receives:             C: [1, 2, 1]
```

**Update rule:** increment your own entry on each local event; on receive, take the componentwise max with the message's vector, then increment your own entry. `Vi[i]` counts events produced by `pi`; `Vi[j]` counts events of `pj` that `pi` knows about.

**Comparing vector clocks:**

- V1 happened before V2  iff  every entry `V1[i] <= V2[i]` AND at least one `V1[i] < V2[i]`.
- V1 and V2 are concurrent if neither is "before" the other (some entry is greater in each).

Vector clocks **detect concurrent updates** — the fundamental requirement for conflict resolution in leaderless systems (Dynamo, Riak).

**Cost:** O(N) per timestamp where N = number of nodes. Doesn't scale to many writers.

### Hybrid Logical Clocks (HLC)

Combine physical and logical time. Like a Lamport clock that stays close to wall-clock time when possible.

```
HLC = (physical_time, logical_counter)

On event: 
  if physical_now > HLC.physical:
    HLC = (physical_now, 0)
  else:
    HLC.logical += 1

On receive (m_hlc):
  pick max of (physical_now, HLC, m_hlc), incrementing logical to preserve order.
```

Used by CockroachDB, YugabyteDB. The HLC value reads like a regular timestamp but obeys causality, so debugging is easier while still preserving ordering.

### TrueTime (Spanner)

Google Spanner uses GPS + atomic clocks to bound clock skew to a known interval (e.g., ±7 ms). Spanner exposes this as `TrueTime.now()` returning an interval `[earliest, latest]`.

To order transactions, Spanner **waits out the uncertainty interval** before committing — ensuring no two transactions overlap if their commit timestamps differ. This trades latency (waiting) for globally consistent timestamps without coordination.

### Happens-Before Relation

Lamport's foundational concept. Event A "happened before" event B (notation: A → B) if:

- A and B are events on the same node and A came first, OR
- A is a message send and B is its receive, OR
- There exists C with A → C and C → B (transitivity).

Events not related by → are **concurrent**. Concurrent events have no inherent order; they can be observed in different orders by different parts of the system. This is the formal basis for everything about causality in distributed systems, and the property every logical-clock scheme is engineered to track.

---

## 7. Distributed Mutual Exclusion

A direct application of logical clocks: letting processes with no shared memory take turns in a **critical section** (CS). The specification:

- **Mutual exclusion (safety):** at most one process is in the CS at any time.
- **No-deadlock (liveness):** if some process wants the CS, some process eventually enters it.
- **No-starvation (liveness):** every process that requests the CS eventually enters it. No-starvation implies no-deadlock.

### Lamport's Distributed Mutual Exclusion

Each process keeps a Lamport clock and a queue of pending requests ordered by `(timestamp, process-id)`.

```
To request the CS:
    increment clock; broadcast REQUEST(ts, self); enqueue own request

On receiving REQUEST(ts, pj):
    enqueue it; reply ACK to pj

Enter the CS when both hold:
    own request has the smallest (ts, id) in the local queue, AND
    an ACK (or later message) has been received from every other process

To release the CS:
    broadcast RELEASE; dequeue own request

On receiving RELEASE from pj:
    dequeue pj's request
```

Safety holds because all processes order requests by the same total order on timestamps (ties broken by id), so they agree on who goes first. The second entry condition — having heard from everyone with a later timestamp — guarantees no earlier request is still in flight. Cost: `3(N-1)` messages per CS entry (request, ack, release to each of the other `N-1` processes).

### Ricart-Agrawala Algorithm

An optimization that drops the explicit RELEASE message, cutting cost to `2(N-1)`. A process that wants the CS broadcasts a timestamped REQUEST and waits for a **reply from every other process**. A receiver replies immediately unless it is itself in the CS or wants it with an earlier `(timestamp, id)` — in which case it **defers** the reply until it exits the CS. Deferring is what enforces mutual exclusion; collecting all replies is what guarantees no earlier request is outstanding.

These algorithms are foundational rather than production tooling — at scale, mutual exclusion is obtained from a coordination service (section 21) with fencing tokens, not by all-to-all message exchange. But they show precisely how a consistent total order plus all-to-all acknowledgment yields a safety guarantee with no central lock.

---

## 8. Broadcast Abstractions

Broadcast generalizes point-to-point links to **one-to-all** communication: a process broadcasts a message and every process delivers it. The abstractions differ in what they guarantee when the **sender crashes mid-broadcast**, having reached some processes but not others. All are built over perfect links (section 5).

```
Uniform Reliable Broadcast   (even a faulty deliverer forces all to deliver)
        ^
Reliable Broadcast           (adds: all-or-none among correct processes)
        ^
Best-Effort Broadcast        (delivery only guaranteed if the sender is correct)
```

### Best-Effort Broadcast (BEB)

Send to each process over a perfect link. Guarantees:

- **Validity:** if a correct sender broadcasts `m`, every correct process delivers `m`.
- **No duplication / no creation:** inherited from the perfect link.

It gives no guarantee if the sender crashes partway: some correct processes may deliver `m`, others never will.

```
upon <BEB, Broadcast | m> do
    forall p in processes do
        trigger <PerfectLink, Send | p, m>

upon <PerfectLink, Deliver | src, m> do
    trigger <BEB, Deliver | src, m>
```

### Reliable Broadcast (RB)

Adds **agreement**: if *any* correct process delivers `m`, then *every* correct process delivers `m` — even if the original sender crashed after reaching only one of them. The standard implementation is "deliver-then-relay": the first time a process delivers a message, it re-broadcasts it, so the message survives the sender's death.

```
upon <RB, Broadcast | m> do
    trigger <BEB, Broadcast | (self, m)>

upon <BEB, Deliver | src, (origin, m)> do
    if (origin, m) not in delivered then
        add (origin, m) to delivered
        trigger <RB, Deliver | origin, m>
        trigger <BEB, Broadcast | (origin, m)>     # relay so it can't be lost
```

In a synchronous system a perfect failure detector lets a process relay only when it detects the sender crashed, saving messages; in an asynchronous system it must always relay.

### Uniform Reliable Broadcast (URB)

Reliable broadcast's agreement covers only correct processes — a faulty process may deliver a message that no correct process ever delivers. **Uniform agreement** strengthens this: if *any* process (correct or faulty) delivers `m`, then every correct process delivers `m`. This matters when a delivery has external side effects before the process crashes.

The implementation holds a message in a pending set and delivers it only once it knows a **majority** of processes have it (so the message is guaranteed to reach every correct process):

```
upon <BEB, Deliver | src, (origin, m)> do
    add src to ack[m]
    if (origin, m) not in pending then
        add (origin, m) to pending
        trigger <BEB, Broadcast | (origin, m)>     # ensure majority sees it

upon (origin, m) in pending and |ack[m]| > N/2 and not delivered[m] do
    delivered[m] = true
    trigger <URB, Deliver | origin, m>
```

With a perfect failure detector (synchronous), "majority" can be replaced by "all correct processes." URB is the building block for uniform total-order broadcast (section 9) and state-machine replication (section 12).

### Probabilistic / Gossip Broadcast

Reliable broadcast costs `O(N^2)` messages — for thousands of nodes, untenable. **Eager probabilistic broadcast** (gossip dissemination) trades certainty for scale: each process forwards a message to `k` randomly chosen peers, who forward to `k` more, for `r` rounds. Messages reach all nodes with high probability (e.g. 99%) in `O(log N)` rounds, with bounded per-node fan-out.

```
upon <Gossip, Broadcast | m> do
    gossip(m, round = 0)

procedure gossip(m, round):
    targets = pick k random processes
    forall p in targets do
        trigger <PerfectLink, Send | p, (m, round)>

upon <PerfectLink, Deliver | src, (m, round)> do
    if m not in delivered then
        add m to delivered
        trigger <Gossip, Deliver | m>
    if round < r then
        gossip(m, round + 1)
```

This is the abstraction behind the gossip failure detectors and anti-entropy in section 17 (Cassandra, Consul, Dynamo).

---

## 9. Ordered Delivery and Total-Order Broadcast

Reliable broadcast says nothing about the *order* in which messages are delivered. Many applications need ordering guarantees, in increasing strength:

```
FIFO order    < Causal order    < Total order
(per sender)    (causally          (all processes agree on one
                related msgs)       sequence, even unrelated msgs)
```

FIFO and causal order constrain *related* messages; total order constrains *all* messages, including concurrent ones from different senders. Total order is orthogonal to the other two — a valid total order could even deliver one sender's messages in reverse send order (though usually FIFO is also imposed).

### FIFO Broadcast

Messages from the *same* sender are delivered in the order that sender broadcast them. Implemented by tagging each message with a per-sender sequence number and buffering out-of-order arrivals until the gaps fill.

```
On broadcast: tag m with sn[self]; sn[self] += 1; RB-broadcast (m, sn)
On RB-deliver (m, sn) from p:
    buffer it; while the next expected seqno for p is buffered, deliver it in order
```

### Causal Order Broadcast

Delivery respects the happens-before relation: if `broadcast(m1) -> broadcast(m2)`, no process delivers `m2` before `m1`. Message `m1` may have caused `m2` if the same process broadcast `m1` before `m2`, or delivered `m1` before broadcasting `m2`, or via transitivity. Causal order = FIFO order + **local order** (if a process delivers `m` before sending `m'`, no correct process delivers `m'` before `m`).

Two implementations:

- **Waiting (vector-clock) causal broadcast:** each message carries the sender's vector clock; a receiver *waits* to deliver until every causally-preceding message has been delivered. Minimal message overhead, but delivery can stall behind missing dependencies.
- **Non-waiting causal broadcast:** each message piggybacks the full set of messages that causally precede it (its "past"). The receiver can deliver immediately because the dependencies travel with the message. No waiting, but message size grows with causal history.

### Total-Order (Atomic) Broadcast

All correct processes deliver all messages in the **same order**. This is the abstraction underneath state-machine replication: if every replica applies the same operations in the same order to the same start state, they stay identical. Its four properties parallel reliable broadcast — validity, integrity (no duplicates/spurious), agreement, and **order** (any two processes that deliver `m` and `m'` deliver them in the same relative order).

The canonical implementation reduces total order to **consensus** (section 15) plus reliable broadcast:

```
upon <TOB, Broadcast | m> do
    trigger <URB, Broadcast | m>

upon <URB, Deliver | src, m> do
    add m to unordered

upon unordered nonempty and no consensus running do
    round += 1
    propose(round, unordered)              # consensus decides the next batch + order

upon <Consensus, Decide | round, batch> do
    forall m in deterministic_order(batch) do
        trigger <TOB, Deliver | m>
        remove m from unordered
```

Each consensus instance agrees on the next batch of messages and their order; a deterministic rule (e.g. sort by id) orders within a batch. The uniformity of the result is inherited from its components: uniform consensus plus uniform reliable broadcast yields the strongest guarantee (uniform agreement, strong total order), and weakening either component weakens the result accordingly.

In practice this is exactly what ZAB (ZooKeeper) and Raft's log provide: a totally-ordered, replicated sequence of operations. "Total-order broadcast" and "consensus on a log" are two views of the same primitive.

---

## 10. Consistency Models

A consistency model is a contract between the storage system and the application: which orderings of operations are observable by which clients?

```
Strongest                                                           Weakest
                                                                   
Linearizable ---> Sequential ---> Causal ---> Read-your-writes ---> Eventual
  |                  |              |           |                     |
  Strong           Strong         Causal      Per-client            Loose
  (total order    (total order    ordering    ordering              ordering
  matches         respects        of dep-     for one user
  real time)      program         endent
                  order)          ops
```

### Linearizability (Strong Consistency)

Every operation appears to happen instantaneously at some point between its invocation and response, and this order matches real-time order across all clients.

```
Real time:           t1 ----- t2 -----  t3 -----  t4
                     |        |          |         |
Client A: write x=1 |--+-----| ack       |         |
Client B:                                 |---+---| read x (must return 1)
                                                       (since B started reading
                                                        AFTER A's write completed)
```

If client A's write commits at real time t2, and client B starts reading at real time t3 > t2, B must see x = 1 (or something later). Reads cannot return stale values once the write has acknowledged.

Formally (used in section 12): an execution is linearizable if there is a sequential order `S` of all operations such that `S` respects real-time precedence (`o1` finishes before `o2` starts implies `o1` precedes `o2` in `S`) and `S` is legal for each object's sequential specification.

**Cost:** every read coordinates with the leader / majority. Latency = round-trip; throughput limited by the slowest member.

**Examples:** etcd, Spanner, Postgres with synchronous commit to a quorum. ZooKeeper provides linearizable *writes*, but its reads are only sequentially consistent — a follower may serve stale (committed-in-the-past) data unless the client issues a `sync` before reading.

### Sequential Consistency

There exists some global order over operations that's consistent with each client's program order, but it doesn't have to match real time. A client's writes appear in order, but the absolute timing between clients is flexible.

Weaker than linearizable (no real-time guarantee), stronger than causal (every client sees the same total order).

### Causal Consistency

Operations that are causally related (one depends on another) appear in causal order to all clients. Operations that are concurrent can appear in different orders.

```
Alice posts: "I'm engaged!"
Bob comments on Alice's post: "Congrats!"

Causal: anyone who sees Bob's comment must also see Alice's post 
(because Bob's post causally depends on having seen Alice's).

Two unrelated posts can be observed in either order.
```

A pragmatic middle ground — strong enough to prevent confusing user-visible anomalies, weak enough to allow much higher availability than linearizable. It is delivered by the causal-order broadcast of section 9.

### Read-Your-Writes

A client sees its own writes immediately. It might not see other clients' writes immediately.

The minimum useful consistency level for interactive applications. Without it, users update their profile, refresh, and see old data — universally confusing.

### Monotonic Reads

A client never sees state move backwards. Once a client has seen value V_n, it will never see an earlier value V_m for the same data.

In leaderless or replicated systems, requests that hit different replicas can show different snapshots. Sticky sessions or read-replica-aware routing solve this.

### Monotonic Writes

A client's writes happen in the order they were issued. (Hard to imagine a useful system without this, but technically not free in some replicated systems.)

### Eventual Consistency

If writes stop, all replicas eventually converge. No guarantee about when, or about what intermediate states look like.

```
Writer:   write x=1  (replica A)
                     |
                     | propagates asynchronously
                     v
                  replicas B, C, D eventually receive x=1
```

In the meantime, a reader hitting B might see the old value, or the new value, or in some cases conflicting concurrent versions that need merging.

**Examples:** DynamoDB default reads, Cassandra default, S3 (until recently), DNS, CDN caches.

### Strong vs Eventual: When to Choose

Strong consistency is needed when:

- A read-then-write is computing a value (counters, account balances, inventory).
- A user action depends on previous user actions (auth, transactions).
- Cross-entity invariants exist (uniqueness, foreign keys).

Eventual consistency is fine when:

- Reads are independent of writes (feed timelines, search indexes, analytics).
- Application can tolerate or merge concurrent updates (CRDTs, last-write-wins).
- Cost of strong consistency (latency, availability) is too high.

Most real systems mix levels: strong for the critical path, eventual for derived data.

---

## 11. CAP and PACELC

### CAP Theorem

Brewer's theorem: a distributed system can offer at most **two of three** properties:

- **Consistency:** every read sees the most recent write or an error.
- **Availability:** every request gets a non-error response.
- **Partition tolerance:** the system continues operating despite network partitions.

In practice, **P is not optional** — networks partition, full stop. So the choice is between C and A *during a partition*:

| Choice | Behavior under partition |
|--------|-------------------------|
| CP | Refuse requests on the minority side (preserves consistency) |
| AP | Serve possibly-stale data on both sides (preserves availability) |

**Common misconception:** CAP is not "always pick 2 of 3." Outside of partitions, all three hold. CAP only constrains behavior *during partitions*.

### ACID and BASE

CAP frames two design philosophies that long predate it:

- **ACID** (Atomicity, Consistency, Isolation, Durability) chooses consistency over availability. The relational-database default: a transaction is all-or-nothing, preserves invariants, runs as if alone, and survives crashes once committed.
- **BASE** (Basically Available, Soft state, Eventually consistent) chooses availability over consistency. The NoSQL default: the system stays available, replicas hold possibly-stale "soft" state, and convergence happens eventually.

Neither is universally correct; they are the two ends of the CAP tradeoff applied to whole system designs.

### PACELC (More Useful)

Daniel Abadi's refinement of CAP. The name is two if/else clauses joined together — read it as **PAC / ELC**:

```
if (Partition)  -> choose A (Availability) or C (Consistency)   <- this is CAP
else            -> choose L (Latency)      or C (Consistency)
```

CAP only says what happens *during* a partition. PACELC adds the more common case — the **Else**, when the network is healthy — and points out there is *still* a tradeoff: to answer a read or write faster, a replica can respond from local state (low latency, possibly stale = **L**), or it can coordinate with other replicas first to guarantee freshness (higher latency = **C**).

A system is classified by picking one letter from each clause, giving a four-letter label. The table below shows only the two chosen letters:

- First letter — partition behavior: **PA** (stay available) or **PC** (stay consistent).
- Second letter — no-partition behavior: **EL** (favor latency) or **EC** (favor consistency).

| System | Under partition | When healthy |
|--------|----------------|--------------|
| Spanner | **PC** — refuse rather than serve stale | **EC** — pays coordination latency to stay consistent |
| Postgres (single leader) | **PC** | **EC** |
| DynamoDB | **PA** — stays available | **EL** — answers locally for low latency |
| Cassandra | **PA** | **EL** |

So Spanner is a "PC/EC" system and Cassandra is "PA/EL". The takeaway: even with a perfectly healthy network, every replicated operation has a consistency-vs-latency knob — CAP's tradeoff never fully goes away, it just changes from "availability vs consistency" to "latency vs consistency".

### Partition Management

Partition detection is not global — one side may notice the partition while the other does not, so different processes simultaneously run in **partition mode** and **normal mode**. A system entering partition mode must decide which operations to allow (to bound the inconsistency it will have to repair) and typically **logs operations** for later reconciliation. Recovery then takes one of two routes:

- **Roll back and re-apply** operations in a proper order, using version vectors to detect which updates were concurrent (and therefore conflicting).
- **Restrict operations** to those that provably commute, so no conflict can arise — the CRDT approach (section 21).

Leftover conflicts that neither route resolves are surfaced for application- or human-level **compensation**.

---

## 12. Replication and Partitioning

Replication keeps copies of data on multiple nodes for availability and fault tolerance. If a single object fails with probability `p`, replicating it across `n` independent nodes drops the unavailability to `p^n` (if any one replica suffices to serve the request) — the basic reason replication exists. The mechanisms differ by where writes go and how they propagate.

### Leader-Based (Single Master)

One node is the leader. All writes go to the leader; the leader propagates to followers.

```
                Writes
                  |
                  v
      +------------------------+
      |         Leader         |
      +------------------------+
        |         |          |
        v         v          v
  +--------+  +--------+  +--------+
  |Follower|  |Follower|  |Follower|
  +--------+  +--------+  +--------+
```

**Pros:** simple, easy to reason about, simple ordering (leader assigns the order).

**Cons:** leader is a single point of write failure. Failover requires consensus on who the new leader is.

**Replication mechanism:**

- **Statement-based:** ship SQL statements. Breaks on non-deterministic statements (`NOW()`, autoincrement, random).
- **Write-ahead log (WAL) shipping:** ship low-level page changes. Tight version coupling — replicas must be same major version.
- **Logical replication:** ship a stream of row-level changes (`INSERT order=42`, `UPDATE`...). Version-tolerant.

### Multi-Leader

Multiple nodes accept writes. Each propagates to the others.

```
             Writes
               |
          +----+----+
          |         |
          v         v
      +------+   +------+
      |  L1  |<->|  L2  |
      +------+   +------+
          |         |
          v         v
     replicas     replicas
```

**Used for:**

- Multi-region active-active (low write latency in every region).
- Offline-first clients (laptops/phones writing without connectivity).
- Calendar sync, collaborative editing, mobile apps.

**Hard part: conflict resolution.** When two leaders write to the same key concurrently, which wins?

- **Last-write-wins (LWW):** highest timestamp wins. Simple, lossy.
- **Application-defined merge:** the application defines how to merge concurrent versions (e.g., shopping carts merge by union).
- **CRDTs:** data structures with built-in commutative merge (section 21).
- **Conflict surfacing:** present the conflict to a human / application for resolution.

Multi-leader is operationally expensive. Most systems avoid it by routing writes to a single leader per partition.

### Leaderless (Dynamo-Style)

Every node accepts writes. There is no leader. Writes go to multiple nodes in parallel.

```
              Writes from client to multiple nodes
                       |
              +--------+--------+
              |        |        |
              v        v        v
          +-----+  +-----+  +-----+
          | N1  |  | N2  |  | N3  |
          +-----+  +-----+  +-----+
              |        |        |
              +--------+--------+
                       |
                       v
             Background: anti-entropy
             (Merkle trees, read repair, hinted handoff)
```

**Used by:** Dynamo, Cassandra, Riak.

**Read and write to multiple replicas** for tunable consistency (see Quorums, section 13).

**Conflict resolution:** same problem as multi-leader. Cassandra uses LWW timestamps; Riak supports CRDTs and application-defined merging.

### Theory: Passive and Active Replication

The two replication topologies above map onto two formal models of how a replicated object stays consistent. Both aim for **linearizability** (section 10), whose sufficient condition is that replicas agree on *which* invocations they handle (atomicity) and on the *order* in which they handle them (ordering).

**Passive replication (primary-backup).** One primary orders all operations; backups apply them. The primary, before replying to a client, sends the update to all backups and waits for their acks — so the operation order the primary chose is the order every replica applies (linearizability). On primary failure a new primary is elected from the backups; the protocol must handle a crash in each window:

- Primary crashes *after* the client got its reply: the new primary recognizes the already-applied request (by id) and re-sends the result without re-applying it.
- Primary crashes *before* sending any update: the client times out and retries; the new primary treats the request as fresh.
- Primary crashes *mid-update* (some backups acked, some not): the new primary must make the update atomic — applied by all or none — before resuming.

Because it relies on correct leader election, passive replication cannot be implemented in fully asynchronous systems.

**Active replication (state-machine replication).** No primary; every replica is a deterministic state machine that receives the *same* operations in the *same* order via **total-order broadcast** (section 9) and therefore produces identical outputs. The client accepts the first reply. There is no failover logic — a crashed replica simply stops contributing — but the dependence on total-order broadcast means it, too, is impossible in a fully asynchronous system (it needs eventual synchrony). This is the model behind Raft/Paxos replicated logs: agree on the order, apply deterministically, stay identical.

### Synchronous vs Asynchronous

Orthogonal to leader topology. How long does the leader (or coordinator) wait for replicas before acknowledging?

| Mode | Latency | Durability |
|------|---------|------------|
| Async | Lowest | Data lost if leader fails before replication |
| Sync to one replica | Medium | Survives one failure |
| Sync to majority | Higher | Survives minority failures |
| Sync to all | Highest | Survives N-1 failures, but slowest replica dictates speed |

In practice, **semi-sync** (wait for one or a quorum, not all) is the standard compromise.

### Partitioning (Sharding)

Replication puts copies of the *same* data on many nodes; **partitioning** splits *different* data across nodes so the dataset and its write throughput can exceed one machine. The two are orthogonal and almost always combined: each partition is replicated, so a node is leader for some partitions and follower for others. The core design question is the **partition function** — the map from a key to the node(s) that own it — and what happens to that map when nodes join or leave.

**Key-range partitioning.** Keys are assigned in sorted contiguous ranges (`a–f` on N1, `g–m` on N2, ...). Range scans are efficient because adjacent keys are co-located, but sequential keys (timestamps, monotonic IDs) drive all writes to one partition — a **hot partition**. Used by HBase, Bigtable, Spanner, CockroachDB.

**Hash partitioning.** Hash the key and partition by hash value. This spreads load evenly and removes hot spots from sequential keys, but destroys range-scan locality. Used by Cassandra (within a partition key), DynamoDB.

**Hot keys.** Even a perfect hash cannot split a *single* key that is itself hot (a celebrity user ID, a viral product). Hashing distributes keys, not the load within one key. Mitigations: append a random suffix to fan the key across partitions (fanning reads back in), or cache it in front.

### Consistent Hashing

Naive hash partitioning computes `node = hash(key) mod N`. The flaw is `N`: changing the node count remaps *almost every* key, forcing a near-total reshuffle of data — unacceptable during routine scaling or a single node failure. **Consistent hashing** removes the dependence on `N`.

Hash both keys and nodes onto the same circular space (the "ring"). A key is owned by the first node found walking clockwise from the key's position. Adding or removing a node only remaps the keys between it and its predecessor — `O(K/N)` keys move instead of nearly all of them.

```
              key k1
                |
          N3 --*--- N1
         /            \
        *    ring      *
         \            /
          N4 ------- N2
           \         |
        (added)    key k2

  Clockwise order: N1 -> N2 -> N4 -> N3 -> N1.
  k1 sits just before N1, so walking clockwise it is owned by N1.
  Adding N4 between N2 and N3 reassigns only the arc (N2, N4]
  (those keys move from N3 to N4); every other key stays put.
```

**Virtual nodes.** One ring position per physical node gives uneven load (random gaps) and dumps a departing node's whole range onto its single successor. Real systems assign each physical node many **virtual nodes** (often hundreds) scattered around the ring, which smooths the load and spreads a departing node's keys across many successors. Used by Dynamo, Cassandra, Riak.

**Interplay with replication.** The N replicas of a key are typically the next N *distinct physical* nodes clockwise from the key's ring position — Dynamo calls this the **preference list**. This is the bridge to the next section: the quorum parameters W and R are counted over exactly those N nodes.

### Rebalancing and Routing

**Rebalancing** moves partitions as the cluster grows or shrinks. The key rule is to *not* tie the partition count to the node count (the `mod N` trap): instead fix a large partition count up front and move whole partitions between nodes (Cassandra's vnodes, Elasticsearch's fixed shard count), or split partitions dynamically as they grow (HBase, Spanner). Rebalancing should move the minimum data and keep the system serving throughout.

**Request routing** — how a client reaches the node owning a key — takes one of three forms:

- Client contacts any node, which forwards or redirects to the owner.
- A partition-aware routing tier / load balancer sits in front.
- The client is partition-aware and contacts the owner directly.

The routing layer's partition map is itself cluster metadata, kept consistent through a coordination service (section 21, ZooKeeper/etcd) or propagated by gossip (section 17).

---

## 13. Quorums

A quorum is a subset of replicas large enough that any two such subsets always share at least one member. A write commits to a write quorum and a read consults a read quorum; the guaranteed overlap is what lets a read observe the most recent write.

### The Overlap Rule

For N replicas, let W be the write quorum (replicas a write must reach before it is acknowledged) and R the read quorum (replicas a read must consult). The single rule that makes reads observe writes is:

```
W + R > N
```

This is the pigeonhole principle. A write set of size W and a read set of size R drawn from N replicas cannot be disjoint when W + R > N — they must share at least one replica, and that replica holds the latest write. Each replica tags its value with a version (timestamp or logical counter), so the read picks the newest value among the R responses.

```
N = 3, W = 2, R = 2          W + R = 4 > 3  ->  sets must overlap

replicas:   [A] [B] [C]
write hits:  A   B           (any 2 of 3)
read hits:       B   C       (any 2 of 3)
overlap:         B           <- carries the latest write; read selects it by version
```

When `W + R <= N` the two sets can miss each other entirely, so a read may return only stale replicas. This is the line between "reads observe the latest write" and eventual consistency.

### Tunable Consistency

Fixing N, the choice of W and R trades read freshness and operation latency against fault tolerance. An operation must wait for its quorum to respond, so a larger quorum is slower; an operation cannot proceed unless its quorum is reachable, so a larger quorum is also less available.

| Configuration | `W+R>N` | Reads see latest | Write availability | Read availability |
|---------------|---------|------------------|--------------------|-------------------|
| W=N, R=1 | yes | yes | none — one dead replica blocks writes | survives N-1 failures |
| W=1, R=N | yes | yes | survives N-1 failures | none — one dead replica blocks reads |
| W=majority, R=majority | yes | yes | survives ⌊(N-1)/2⌋ failures | survives ⌊(N-1)/2⌋ failures |
| W=1, R=1 | no | no (eventual) | survives N-1 failures | survives N-1 failures |

The first three rows satisfy the overlap rule and therefore return the latest write; they differ only in where the cost falls. `W=N, R=1` makes reads cheap and fault-tolerant but writes both slow and fragile, since every replica must acknowledge. `W=1, R=N` is the mirror image. Majority quorums balance the two and are the configuration consensus protocols use, because a majority is the smallest quorum that overlaps another majority while tolerating a minority of failures.

The last row, `W=1, R=1`, deliberately breaks the overlap rule: with W + R = 2 ≤ 3 the write and read sets can be disjoint, so reads may be stale. This is the fastest and most available setting and the reason it is only eventually consistent.

Cassandra exposes these levels per query as `ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`, and `EACH_QUORUM`.

### Availability and Sloppy Quorums

The general statement of the availability column above: under `W + R > N`, an operation needs at least its own quorum reachable, so writes tolerate `N - W` failures and reads tolerate `N - R` failures. Shrinking W speeds up and hardens writes at the cost of read freshness; shrinking R does the reverse.

**Sloppy quorums:** when the designated replicas for a key are unreachable, accept the write on whatever replicas are available (hinted handoff) and hand the data back to the proper replicas once they recover. This preserves write availability during partitions but suspends the overlap guarantee — a read against the designated replicas may not yet see such a write — so it trades strict quorum semantics for availability.

### Quorums Are Not Consensus

Quorum reads/writes provide eventual consistency with bounded staleness. They do **not** provide linearizability — concurrent operations can still produce inconsistencies that need application-level resolution.

True linearizability requires consensus (Paxos/Raft), where every operation is sequenced through the leader and confirmed by a majority. The next section shows how quorums become linearizable registers once a timestamp discipline and a write-back rule are added.

---

## 14. Distributed Registers

A **register** is the simplest shared object: a single variable with `read()` and `write(v)`. It is the formal bridge between quorums (section 13) and full consensus (section 15) — a register provides linearizable single-value storage *without* solving consensus, which is why it can be implemented in a fully asynchronous system where consensus cannot.

Notation `(X, Y)` means X writers and Y readers, fixed in advance. Two operations are **concurrent** if neither's response precedes the other's invocation. The semantics strengthen in three steps:

- **Safe:** a read concurrent with a write may return any value; a read not concurrent with any write returns the last written value.
- **Regular:** safe, plus a concurrent read returns either the last value or the concurrently-written one (no arbitrary garbage).
- **Atomic (linearizable):** regular, plus **ordering** — once a read returns `v2`, no later read may return an *older* `v1`. Reads cannot go backward.

### Regular Register: Read-One Write-All (fail-stop)

Assumes a perfect failure detector. Each process stores a local copy. A write updates every non-crashed process and waits for their acks; a read returns the local copy.

```
write(v):  beb-broadcast (WRITE, v); wait for ACK from every correct process
read():    return local value
```

Cost: read 0 messages (local), write at most `2N`. **It breaks if the failure detector is imperfect** — a falsely-suspected process is dropped from the write set and silently goes stale.

### Regular Register: Majority Voting (fail-silent)

Removes the perfect-failure-detector assumption; assumes only a majority of correct processes. Each stored value carries a **timestamp**. A write increments the timestamp and pushes `(ts, v)` to all; a read collects a quorum and takes the highest-timestamped value.

```
write(v):
    ts += 1
    broadcast (WRITE, ts, v)
    each receiver: if ts > local ts, adopt (ts, v); reply ACK
    wait for ACK from a majority (> N/2)        # write complete

read():
    rid += 1
    broadcast (READ, rid)
    each receiver: reply (rid, ts, v)
    wait for a majority of replies
    return value with the highest ts            # read complete
```

Quorum intersection (`W + R > N`) guarantees the read quorum sees the latest completed write. Cost: at most `2N` messages each for read and write. This is a *regular* register — concurrent reads can still observe values out of order.

### Atomic Register: Read-Impose Write-Majority

To reach atomic (linearizable) semantics, **the read writes back**: after picking the highest-timestamped value, the reader imposes it on a majority before returning. This blocks a later read from a lagging quorum from returning an older value.

```
read():
    query a majority; pick (ts, v) with the highest ts
    broadcast (WRITE-BACK, ts, v); wait for majority ACK     # impose before returning
    return v
```

The crash-tolerant variant with a perfect failure detector is **read-impose write-all** (impose on every process instead of a majority). Cost rises to ~`4N` for a read (one round to read the quorum, one to write back). This write-back rule is precisely what turns a stale-tolerant quorum store into a linearizable one — and it is the mechanism Dynamo-style read-repair approximates.

A register stops short of consensus: it linearizes operations on a *single* value, but multiple writers can still overwrite each other (last-timestamp-wins) rather than agreeing. Agreeing on a *sequence* of values across contending proposers is the consensus problem.

---

## 15. Consensus (Paxos, Raft, ZAB)

Consensus is the problem of getting a group of nodes to agree on a single value (or a sequence of values), despite some of them failing.

### The Problem

Imagine three nodes deciding which of two leaders to follow, or four nodes deciding the order of two concurrent writes. They need to agree exactly. Wrong answers (two leaders, inconsistent order) corrupt the system. Stuck answers stall progress.

A consensus algorithm must satisfy:

- **Termination (liveness):** every non-faulty node eventually decides.
- **Validity:** if a value is decided, it was proposed by some node.
- **Integrity:** each node decides at most one value.
- **Agreement:** no two nodes decide differently. **Uniform agreement** additionally forbids a faulty node (before it crashes) from deciding differently from the correct ones — required when a decision has external side effects.

### Theory: Flooding Consensus

In a *synchronous* system with crash faults, consensus is straightforward by **flooding**. The algorithm runs in rounds; in each round every process broadcasts the set of proposals it has seen. After at most `f + 1` rounds (with up to `f` crashes), all correct processes hold the same proposal set and apply the same deterministic choice (e.g. the minimum). Agreement holds because the sets converge; termination holds because the round count is bounded.

**Uniform** agreement needs more care: a process that decides and then crashes before sharing its decision can leave others to decide differently. The fix is to decide only in the final round, once all surviving processes provably hold the same set. This synchronous algorithm is the conceptual baseline — the difficulty is doing the same thing *asynchronously*.

### FLP Impossibility

The Fischer-Lynch-Paterson result (1985): in a fully asynchronous system with even one possible crash failure, **no deterministic consensus algorithm can guarantee both safety and liveness.** With no timing bounds, a slow process is indistinguishable from a crashed one, so any algorithm can be forced to wait forever or decide wrongly.

The escape is to assume **partial synchrony** (timeouts, eventual delivery), which is what real systems use. FLP says consensus can stall during certain failure patterns, but a correct protocol never returns a wrong answer — safety is preserved, only liveness is at risk.

### Paxos

The original consensus protocol (Lamport, 1989). Famously hard to explain. It guarantees safety always and makes progress only when the network behaves well for long enough (partial synchrony), exactly as FLP requires.

**Why it is built the way it is.** A single acceptor would be simplest but is a single point of failure, so a value is "chosen" only when a **majority** of acceptors accept it. That forces acceptors to accept multiple proposals over time, which raises the danger of two different values being chosen. Paxos rules this out with an invariant, derived in stages:

- **P1:** an acceptor accepts the first proposal it receives — but competing proposers can then deadlock with no majority.
- **P2c (the real rule):** every proposal carries a unique number `n`; before issuing value `v` at number `n`, a proposer must learn, from a majority, that either no acceptor in that majority has accepted anything below `n`, or `v` is the value of the highest-numbered proposal any of them already accepted. This guarantees: once a value is chosen, every higher-numbered proposal carries the *same* value.

**The two phases** implement P2c:

```
Phase 1 (prepare):
  Proposer picks a unique, increasing number N, sends Prepare(N) to acceptors.
  Acceptor: if N > highest seen, promise never to accept anything < N,
            and return the highest-numbered (n', v') it has already accepted.
            else reply NACK.

Phase 2 (accept):
  Proposer collects promises from a majority.
  Sets value = the highest-numbered v' returned (or its own if none returned).
  Sends Accept(N, value) to acceptors.
  Acceptor: accept unless it has already promised to a higher number.

A value is chosen once a majority accepts. Learners observe the majority
of Accepts and decide; they propagate Decide(v) to the rest.
```

**Multi-Paxos:** elect a stable leader once, then skip Phase 1 for subsequent values. The leader runs many Phase 2s in parallel for a stream of values — this is what a real log-replication system uses. To avoid dueling proposers (each forcing the other to restart Phase 1), proposers are effectively elected via leader election (section 16).

Paxos is correct but operationally complex. Modern systems usually use Raft.

### Raft

Raft keeps a set of replicas in agreement by electing one **leader** that takes all writes and copies them to everyone else. The whole protocol answers two questions: how to pick a leader, and how the leader keeps the followers in sync. It gives the same guarantees as Paxos but is designed to be understood. Used by etcd, Consul, CockroachDB, TiKV, and MongoDB.

A node is a **follower**, a **candidate**, or the **leader**. The leader sends periodic heartbeats; as long as they arrive, everyone stays a follower.

```
            no heartbeat (election timeout)
  Follower ---------------------------------> Candidate
     ^                                            | wins majority vote
     |  hears from a valid leader                 v
     +------------------------------------------ Leader --> heartbeats
```

**Electing a leader.** If a follower hears no heartbeat for a (randomized) timeout, it assumes the leader is gone, becomes a candidate, and asks the others for their vote. Whoever collects votes from a majority becomes the new leader. Each node votes for only one candidate, so there can never be two leaders at once. The timeouts are randomized so that nodes rarely campaign at the same instant — otherwise they keep splitting the vote and no one wins.

Each leadership period is stamped with a **term**: a counter that increases by one at every election. It counts *how many leaders the cluster has been through*, not time. Every message carries the sender's term, and the rule is simple — a node that sees a higher term than its own is out of date and steps down to follower. This is what stops a partitioned-off old leader from causing trouble when it returns: its messages carry a stale term, so everyone ignores them and it demotes itself. No central referee needed.

**Replicating the log.** The leader stores client commands in an ordered **log** and forwards each new entry to the followers. Once a majority have stored an entry, it is **committed** — locked in for good — and every replica applies committed entries to its state machine in the same order. Because everyone replays the same commands in the same sequence, every replica ends up in the same state.

The log below is one replica's. Each column is one entry: its **index** (position, applied in this order), the **term** of the leader that wrote it, and the **command** to run. The term jumps 1 → 2 → 3 mark where leadership changed hands — entries 1–2 were written by the term-1 leader, 3–4 by its successor, and so on.

```
Index:    1     2     3     4     5     6
Term:     1     1     2     2     3     3      <- leader changed at each jump
Command:  x=1   x=2   y=5   z=3   x=4   y=7
                                  ^
                                  committed up to here: a majority has stored
                                  entries 1-6, so they are permanent. Anything
                                  after this is written but still tentative.
```

If a follower's log has fallen behind or diverged, the leader simply rewinds to the last point where they agree and overwrites the rest with its own copy. Followers always conform to the leader; the leader never edits its own log.

**Why it stays correct.** A node only grants its vote to a candidate whose log is at least as up-to-date as its own. A committed entry already lives on a majority, and a winner also needs a majority, and two majorities always share a node — so the new leader is guaranteed to already have every committed entry. Nothing committed is ever lost.

**Properties:**

- Tolerates F failures with 2F+1 nodes (it needs a majority to function).
- All writes go through the leader — simple to reason about, but the leader is a throughput bottleneck.
- Reads are linearizable only if served by the leader after it confirms it is still leader (a stale, partitioned-off leader would otherwise return old data).

Raft's replicated log is total-order broadcast (section 9) in production form: agree on one order of commands, then apply it the same way on every replica (section 12, active replication).

### ZAB (Zookeeper Atomic Broadcast)

ZooKeeper's consensus protocol. Similar structure to Raft (leader-based, term-numbered, log-replicated). Predates Raft. As its name says, it is literally an atomic- (total-order-) broadcast protocol. ZooKeeper itself is the coordination primitive layer for many other distributed systems.

### Comparing Consensus Protocols

| Protocol | Used by | Notes |
|----------|---------|-------|
| Paxos | Chubby, Spanner, FoundationDB | Original; many variants (Multi-Paxos, Fast Paxos, Cheap Paxos) |
| Raft | etcd, Consul, CockroachDB, MongoDB, TiKV | Easier to understand and implement |
| ZAB | ZooKeeper | Predates Raft, similar mechanism |
| Viewstamped Replication | (precursor, less used in production) | |
| EPaxos | (research) | Leaderless multi-Paxos variant |
| PBFT, HotStuff | Tendermint/CometBFT, Diem (HotStuff) | Byzantine fault tolerant (section 18) |

**For practical engineering:** Raft is the default mental model. Consensus is typically obtained from an existing implementation (etcd, Consul) rather than implemented from scratch.

### Cost of Consensus

Every consensus operation requires at least one round trip to a majority of nodes. Throughput is limited by the slowest member of the quorum and by network round-trips. Strong consistency at scale is expensive — minimize how much state goes through consensus.

```
Typical pattern:
  Metadata / coordination: through consensus (etcd, ZooKeeper)
  Bulk data:               in higher-throughput, lower-consistency stores
```

Spanner pushes consensus into the data path by partitioning aggressively (one Raft group per data range), keeping each group small enough for fast consensus.

---

## 16. Leader Election

Choosing a single leader from a group of equal nodes. The classic problem solved by consensus.

### Approaches

**Consensus-based** (Raft/Paxos): formal majority vote with term numbers. Guaranteed at most one leader per term, even under partition.

**Lease-based:** a leader holds a lease (TTL) granted by a coordinator (ZooKeeper, etcd). Renew before expiry. If renewal fails, lease expires, another node can acquire. Liveness depends on clocks and lease length.

**Bully algorithm:** highest-ID node wins. Simple but doesn't handle partitions well.

### Theory: Leader Election as an Oracle

Formally, leader election is a thin layer over a failure detector (section 17) — instead of asking "who is dead?", it asks "who is the one alive process to trust?".

- **Perfect leader election** is built on a *perfect* failure detector: it eventually elects the same correct process at every node (e.g. the lowest-id non-crashed process). If the underlying detector is imperfect, it may falsely demote a healthy leader.
- **Eventual leader election** (the `Ω` oracle) is built on an *eventually perfect* detector: leadership may flap and several processes may briefly believe they lead, but after some unknown time a single correct leader is elected and **stabilizes**. `Ω` is the weakest oracle that makes asynchronous consensus solvable, which is why Multi-Paxos and Raft are, at heart, "elect a stable leader, then let it drive."

### Split-Brain

Two nodes both believe they are the leader. Catastrophic — both accept writes, system diverges.

**Causes:**

- A leader is partitioned from the majority but doesn't realize it.
- A leader experiences a long GC pause; the cluster elects a new leader; the old leader resumes and acts as if it's still leader.
- Lease expiration without proper fencing.

**Prevention:**

- **Fencing tokens:** each leadership change gets a monotonically increasing token. Downstream systems reject operations with stale tokens.
- **Majority quorum requirements:** a leader must continuously prove it has a majority. If it loses contact with the majority, it steps down.
- **STONITH ("shoot the other node in the head"):** when a new leader is elected, the old leader is forcibly killed.

```
Without fencing:
  Leader A writes "do X" to storage.
  Leader A pauses (long GC).
  Cluster elects Leader B.
  Leader B writes "do Y" to storage (overwriting X).
  Leader A wakes up and writes "do X" — overwriting Y!

With fencing tokens (incrementing per leader):
  Leader A gets token 5.
  Leader A writes "do X with token 5".
  Storage records: last_token = 5.
  Cluster elects Leader B (token 6).
  Leader B writes "do Y with token 6".
  Storage records: last_token = 6.
  Leader A wakes up, tries to write with token 5.
  Storage rejects: 5 < 6.
```

---

## 17. Failure Detection and Gossip

How do nodes know which other nodes are alive?

### Naive Approach: Heartbeats

Each node periodically sends "I'm alive" to a coordinator. If N missed heartbeats, declare dead.

**Problem:** heartbeat through a single coordinator doesn't scale. The coordinator becomes a bottleneck and a single point of failure.

### Theory: Failure Detector Abstractions

A **failure detector** is an oracle that encapsulates a system's timing assumptions and answers "which processes have crashed?" Its quality is two properties:

- **Completeness:** every crashed process is eventually suspected.
- **Accuracy:** correct processes are not wrongly suspected.

```
Detector                  Completeness   Accuracy           Requires
-----------------------   ------------   ----------------   ----------------------
Perfect (P)               strong         strong             synchronous system
Eventually Perfect (<>P)  strong         eventual (after t) partial synchrony
```

The **perfect detector (P)** makes no mistakes — feasible only in a synchronous system, where a missed reply within a known timeout *must* mean a crash. The **eventually perfect detector (◇P)** tolerates partial synchrony: before the unknown stabilization time it may *suspect* a slow process wrongly, but when a belated reply arrives it un-suspects and lengthens its timeout, so after some point it stops making mistakes. ◇P is what real systems implement, and (via the `Ω` leader elector of section 16) it is exactly what lifts asynchronous consensus past FLP.

### Gossip (Epidemic) Protocols

Each node periodically picks a random peer and exchanges "what I know about who's alive." Information spreads exponentially across the cluster — O(log N) rounds to reach everyone. This is the failure-detection use of the gossip broadcast in section 8.

```
Round 1:  Node A talks to random node B. They exchange known-states.
Round 2:  A talks to C; B talks to D. Now four nodes have the latest info.
Round 3:  Eight nodes have it...
```

**Used by:** Cassandra, Consul, Serf, AWS Dynamo, many service meshes.

**Properties:**

- Scalable to thousands of nodes.
- No single point of failure.
- Eventual consistency about who's alive.
- Robust to message loss.

### SWIM Protocol

A more efficient failure detector. SWIM (Scalable Weakly-consistent Infection-style process group Membership) avoids all-to-all pinging:

1. **Direct ping** to a randomly chosen target.
2. **Indirect ping** through K random intermediaries if the direct ping fails.
3. **Suspicion mechanism** before declaring death — gives the target a chance to refute.

Used by HashiCorp Serf, Consul, and others.

### The Hard Part: Partial Failures

A node that's slow but not dead is harder to handle than a clean crash. Some systems quarantine slow nodes (move work away from them, reduce their priority); others escalate to "treat as dead" after thresholds.

```
Latency-based exclusion patterns:
  - p99 > 10x cluster median for sustained period -> remove from rotation
  - Connection error rate > X% -> circuit-break
  - Health check failure with TTL -> demote/restart
```

---

## 18. Byzantine Fault Tolerance

A **Byzantine** process may deviate arbitrarily: crash, lie, send conflicting messages to different peers, or actively coordinate with other faulty processes to break the protocol. This is the failure model for open, adversarial, or trustless systems (blockchains, inter-organization consensus, safety-critical avionics). Everything in the earlier sections assumed crash faults; Byzantine tolerance rebuilds the same stack against an adversary, and it costs more on every axis.

The two structural changes versus the crash model:

- **Authenticated channels are mandatory.** Cryptographic signatures or MACs (the authenticated perfect link of section 5) stop an adversary from forging or tampering with messages in transit. They do *not* stop a Byzantine *sender* from signing malicious-but-well-formed content — that requires voting.
- **The quorum grows from `N > 2f` to `N > 3f`.** With up to `f` liars, a process needs `2f + 1` matching messages to be sure a *majority of the matches came from correct processes* (`f` could be lies). This `N >= 3f + 1` bound recurs throughout Byzantine protocols.

### Byzantine Broadcast

Reliable broadcast (section 8) must be rebuilt so a Byzantine sender cannot make correct processes deliver different messages. Both variants use an **echo** phase: on first seeing the sender's message, each process echoes it to everyone, and a process only acts once it has collected enough matching echoes that a majority must have come from correct nodes.

- **Byzantine consistent broadcast:** deliver once *strictly more than* `(N + f) / 2` matching echoes arrive. Guarantees no two correct processes deliver *different* messages — but a correct process may deliver while another delivers *nothing* (no agreement). A uniform version is impossible.
- **Byzantine reliable broadcast:** adds a second **ready** phase. After the echo quorum, processes broadcast `READY`; a process also sends `READY` once it sees `f + 1` readys (proof that at least one correct process is ready), and **delivers** once it sees `2f + 1` readys. The ready amplification gives all-or-nothing agreement among correct processes.

### Byzantine Consensus and the Generals Problem

Lamport's **Byzantine Generals Problem** frames it: loyal and traitorous generals must agree on attack-or-retreat; the army wins only if all loyal generals choose the same action, and a few traitors must not be able to swing them to a bad one. The problem reduces to: how does one general reliably communicate its order to all loyal generals?

The **Oral Messages** algorithm `OM(f)` solves it for `N >= 3f + 1` by recursion:

```
OM(0):  commander sends its value to every lieutenant;
        each lieutenant uses the value received (or RETREAT if none).

OM(f), f > 0:
   1. commander sends its value to every lieutenant.
   2. each lieutenant i, with the value it received, acts as commander in
      OM(f-1) to relay that value to the other N-2 lieutenants.
   3. each lieutenant decides the MAJORITY of the values it has collected
      (its own from the commander, plus those relayed by the others).
```

The recursion is why the bound is `3f + 1`: with 3 generals and 1 traitor a loyal lieutenant cannot tell whether the commander or its peer is lying, and majority breaks down. The cost is brutal — message complexity is exponential in `f`. **Authenticated** (signed) messages collapse the requirement: unforgeable signatures let a recipient trace a value's provenance, so a chain of signatures `v:0:j1:...:jk` proves the path, and consensus becomes achievable with only `f + 1` connectivity. The trade is a dependency on a key-distribution authority (itself a single point of trust).

### PBFT (Practical Byzantine Fault Tolerance)

`OM(f)`'s exponential cost makes it impractical; **PBFT** (Castro & Liskov) is the polynomial-time protocol that made BFT deployable, and its lineage underpins modern BFT state-machine-replication systems such as Tendermint/CometBFT (Cosmos). It is a primary-backup state machine (like passive replication, section 12) hardened against a Byzantine primary, tolerating `f` faults with `N = 3f + 1` replicas. A request passes three all-to-all phases:

```
PRE-PREPARE:  primary assigns a sequence number to the request, multicasts it.
PREPARE:      each backup multicasts agreement; a replica is "prepared" once it
              has 2f matching PREPAREs (+ the pre-prepare). Fixes the order.
COMMIT:       replicas multicast COMMIT; once a replica has 2f+1 COMMITs it
              executes the request and replies to the client.

The client accepts the result once it holds f+1 matching replies from
different replicas (so at least one came from a correct replica).
```

The two voting rounds (prepare fixes *order*, commit fixes *durability of that order* across enough correct replicas) are what defeat a primary that tries to tell different replicas different things. A separate **view-change** protocol replaces a primary that stalls. Safety holds in full asynchrony; liveness needs partial synchrony — FLP again.

### Byzantine Registers and Multi-Hop (brief)

The register stack (section 14) also rebuilds under Byzantine faults: a **safe register** needs a *masking quorum* `N > 4f` (quorum size `> (N + 2f)/2`) because a read must out-vote both stale and lying replicas; signatures shrink the requirement again. The same picture extends to multi-hop networks (Dolev's and the CPA algorithms for partially connected, possibly malicious relays), but that regime matters for sensor/mesh networks more than typical datacenter systems. The takeaway is uniform: every crash-tolerant abstraction has a Byzantine analogue, uniformly more expensive.

---

## 19. Distributed Transactions

A transaction that spans multiple data stores or nodes. Atomic across all participants: either all commit or none do.

### Two-Phase Commit (2PC)

The classical protocol. A coordinator and N participants.

```
Phase 1 (Prepare):
  Coordinator -> all participants: "PREPARE: can you commit?"
  Each participant: writes "ready" to its local WAL, replies "yes" or "no"

Phase 2 (Commit/Abort):
  If all said yes:
    Coordinator -> all participants: "COMMIT"
    Each participant commits locally, replies "done"
  If any said no (or any didn't reply):
    Coordinator -> all participants: "ABORT"
```

**Atomicity:** if all participants prepare successfully, the coordinator's commit decision is durable and all participants will eventually commit (even after crashes — they replay the WAL).

**Problem: blocking on coordinator failure.** If the coordinator crashes after Phase 1 but before sending Phase 2, participants are stuck. They've prepared, can't commit (might be aborted), can't abort (might be committed). They hold locks until the coordinator recovers.

3PC (Three-Phase Commit) adds a precommit phase to reduce blocking, but assumes synchronous networks (rarely true) and is rarely used in practice.

**Modern systems mostly avoid distributed transactions:**

- Co-locate data that must be transactional (single-shard transactions).
- Use sagas (next section) for multi-system workflows.
- Use idempotent operations and reconciliation instead of atomic multi-system updates.

### Sagas

A saga is a sequence of local transactions, each on a single service. If any step fails, **compensating transactions** undo the previous steps.

```
Booking workflow:
  T1: Reserve flight     (compensation: cancel flight)
  T2: Reserve hotel      (compensation: cancel hotel)
  T3: Charge credit card (compensation: refund)

If T3 fails, run compensations T2-comp and T1-comp in reverse.
```

**Properties:**

- No global lock; no 2PC overhead.
- Eventually consistent (the system is briefly in an intermediate state).
- Compensations must be idempotent (retries are unavoidable).
- "Semantic" rollback — the compensation doesn't always restore the exact prior state (refund != reversal).

**Orchestration vs choreography:**

- **Orchestration:** a central saga coordinator drives the sequence and triggers compensations on failure. Easier to reason about, but the coordinator is critical.
- **Choreography:** each service reacts to events from others. More decentralized, harder to trace.

### Spanner-Style: Distributed ACID Without 2PC's Pain

Spanner uses 2PC plus Paxos: each partition (shard) is a Paxos group, and 2PC coordinates across Paxos groups. TrueTime provides external consistency (linearizable across the entire system).

Cost: latency dominated by 2PC + commit-wait for TrueTime. But achieves true distributed ACID without the blocking pathology, because each "participant" is a fault-tolerant Paxos group rather than a single node.

Spanner-derivatives (CockroachDB, YugabyteDB) follow the same pattern.

---

## 20. Exactly-Once and Idempotency

### Why Exactly-Once Is Hard

In a distributed system, the available guarantees are:

- **At most once:** drop on failure (simple, lossy).
- **At least once:** retry on failure (simple, duplicates possible).
- **Exactly once:** at least once + dedup (idempotency) — what we usually want.

"Exactly once delivery" at the network/messaging layer is a marketing claim. What's actually achievable is **effectively-once processing**: at-least-once delivery combined with idempotent processing.

### Idempotency Patterns

**Idempotent operations:** the same operation can be applied multiple times with the same effect as applying it once.

- `SET x = 5` is idempotent.
- `x = x + 1` is not.
- `INSERT ... ON CONFLICT DO NOTHING` is idempotent.
- `INSERT` alone is not (duplicates accumulate).

**Idempotency keys:** the client generates a unique key per logical operation. The server records (key, result) on first execution. Subsequent requests with the same key return the cached result.

```
POST /payments
Idempotency-Key: abc-123
{"amount": 100, ...}

First request: processes, stores (abc-123 -> payment_id 42)
Second request (retry): returns payment_id 42 without re-charging
```

Stripe's API is the canonical example.

**Deduplication windows:** for streams, dedupe by event ID over a TTL window. Beyond the window, duplicates may slip through, but the cost is bounded.

### Transactional Outbox + Idempotent Consumer

The standard way to achieve effectively-once semantics across a database and a message queue: the producer writes its business change and an outbox row in the *same* transaction, a relay publishes outbox rows at-least-once, and the consumer dedupes by idempotency key. The atomic local write closes the gap that two separate writes (DB then queue) would leave open.

The concrete implementation — outbox schema, the relay, Change Data Capture as the log-based alternative, and idempotent-consumer mechanics — lives in `Data Systems.md`. This section owns the *why* (delivery semantics, idempotency); that one owns the *how*.

---

## 21. Distributed Coordination Primitives

Cross-node coordination — locks, leader election, configuration, service discovery — typically uses a coordination service rather than a hand-rolled consensus implementation.

### Coordination Services

| Service | Algorithm | Used for |
|---------|-----------|----------|
| ZooKeeper | ZAB | Kafka metadata (pre-KRaft), HBase, Hadoop, many Apache projects |
| etcd | Raft | Kubernetes, Patroni (Postgres HA), CoreDNS |
| Consul | Raft | HashiCorp stack: service discovery, config, KV |
| Apache Curator | wraps ZooKeeper | Higher-level recipes |

All offer a similar abstraction: a hierarchical key-value store with watches, ephemeral nodes (auto-deleted on session loss), and linearizable operations.

### Distributed Locks

A lock that prevents two nodes from doing the same exclusive operation simultaneously.

```
def critical_section():
    lease = etcd.acquire_lock("my-job", ttl=30s)
    try:
        do_work()
    finally:
        etcd.release(lease)
```

**Critical caveat: fencing.** A distributed lock with TTL has a fundamental race: the lock holder might pause (GC, slow I/O), the lock might expire, another node acquires it — and the first node wakes up still believing it holds the lock.

Without fencing tokens, both holders can execute the critical section. The downstream resource must enforce: every operation includes a token (monotonically increasing); resource refuses operations with old tokens.

```
Without fencing:
  Node A acquires lock.
  Node A pauses (GC).
  Lock TTL expires.
  Node B acquires lock.
  Node B writes "v1" to DB.
  Node A wakes up, writes "v0" to DB (now corrupt).

With fencing tokens:
  Node A acquires lock with token=5.
  Node A pauses.
  Lock TTL expires.
  Node B acquires lock with token=6.
  Node B writes "v1" with token=6. DB records token=6.
  Node A wakes up, tries to write "v0" with token=5.
  DB refuses: 5 < 6.
```

The famous Martin Kleppmann critique of Redis's Redlock highlighted this. This is the same fencing mechanism as the leader-election tokens in section 16 — distributed locking and leader election are the same problem at different granularity.

### Service Discovery

How services find each other in a dynamic environment where instances come and go.

```
Service registers itself: "I am order-service, instance i-abc, healthy, at 10.0.1.5:8080"

Clients query: "where is order-service?"
              -> [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]
```

**Mechanisms:**

- **DNS-based:** simplest. CoreDNS, AWS Route53, K8s built-in DNS. Cache TTLs introduce staleness.
- **Service registry:** Consul, etcd, ZooKeeper. Strong consistency, watches for changes.
- **Service mesh:** Istio, Linkerd. Sidecars handle discovery + load balancing + mutual TLS (mTLS).
- **Cloud-native:** K8s Service abstractions, AWS Cloud Map, GCP Service Directory.

Health checks are essential — registered instances must be regularly verified, unhealthy ones removed.

### CRDTs (Conflict-Free Replicated Data Types)

Data structures designed so that concurrent updates from multiple replicas always converge to the same value, without coordination. They are the constructive answer to the partition-recovery problem of section 11: if every operation provably commutes, a partition needs no reconciliation.

**Two flavors:**

- **State-based (CvRDT):** replicas exchange full state; merge operation is commutative, associative, idempotent.
- **Operation-based (CmRDT):** replicas exchange operations; operations commute.

**Examples:**

```
G-Counter (grow-only counter):
  Each node has its own count. Increment locally.
  Merge: take max of each node's count, sum.
  Query: sum across nodes.

PN-Counter (positive-negative):
  Two G-Counters: one for increments, one for decrements.
  Value = increments - decrements.

OR-Set (observed-remove set):
  Each add gets a unique tag.
  Remove requires knowing the tag.
  Merge: union of all adds and removes; element present iff at least one
         (element, tag) pair has no corresponding remove.

LWW-Register (last-write-wins register):
  Each value has a timestamp. Higher timestamp wins.
  Lossy: concurrent writes lose the loser.

Sequence CRDTs (Logoot, RGA):
  For collaborative text editing. Each character has a position identifier
  that maintains order under concurrent insertions.
```

**Used by:** Riak (CRDTs as first-class types), Redis (CRDTs via Redis Enterprise), Yjs / Automerge (collaborative editing), Figma's multiplayer engine.

CRDTs trade flexibility for coordination-freeness. Not every data type has a useful CRDT (the merge semantics may not match application requirements), but where they fit, they are powerful.

---

## 22. Publish/Subscribe and Event Routing

Publish/subscribe is a messaging paradigm that decouples the producers of events from their consumers. **Publishers** emit events; **subscribers** declare interests; an **Event Notification Service (ENS)** delivers each event to every subscriber whose interest it matches. No party addresses another directly.

It provides four kinds of decoupling, which together make it the backbone of event-driven architectures:

- **Space decoupling:** parties do not need to know each other; routing is by content, not address.
- **Time decoupling:** parties need not be online simultaneously; the ENS mediates.
- **Synchronization decoupling:** producing does not block on consumers.
- **Many-to-many:** each event can reach many consumers; each consumer hears many producers.

An **event** is a set of attribute values over a fixed, shared schema; a **subscription** is a constraint over that schema. Geometrically, the schema defines an n-dimensional space, each event is a point, each subscription is a subspace, and an event matches a subscription when the point falls inside the subspace. The matching model defines the "flavor":

| Flavor | Subscriptions select on | Notes |
|--------|--------------------------|-------|
| Topic-based | a flat topic tag on each event | Simplest; the Kafka/MQTT model |
| Hierarchy-based | topics in a containment tree | Subscribing to a topic also yields its subtree |
| Content-based | predicates over event attributes | Most expressive; ENS filters before delivery, costliest to route |

Real brokers (Kafka, RabbitMQ, NATS, Google Pub/Sub) sit on this model, layering in durability, ordering, and delivery guarantees (the at-least-once / effectively-once machinery of section 20). Content-based routing is the most powerful and the hardest to scale, because the broker must evaluate every subscription's predicate against every event.

---

## 23. Blockchain and Distributed Ledgers

A **blockchain** is a fully-replicated, append-only ledger maintained over a trustless peer-to-peer network: transactions are grouped into blocks, each block cryptographically chained to its predecessor (it contains the hash of the previous block), so altering a past transaction would require re-writing every block after it. The result is public, immutable, and non-repudiable shared state without a central authority. It is, in distributed-systems terms, **total-order broadcast (section 9) in a Byzantine, open-membership setting** — and its consensus mechanism is determined by whether membership is controlled.

### Permissioned (closed membership)

Participants are known and admitted, and known to each other's public keys. Agreement on the next block uses a classical **Byzantine consensus** protocol of the PBFT lineage (section 18), tolerating `f` Byzantine nodes with `N = 3f + 1`. This gives immediate finality and high throughput, at the cost of a fixed, vetted membership. Tendermint/CometBFT (Cosmos) is a canonical example. Not every permissioned ledger runs BFT, though: Hyperledger Fabric's default ordering service is Raft, which is only *crash* fault-tolerant — it assumes a non-adversarial ordering tier and pushes trust onto endorsement policies instead.

### Permissionless (open membership)

Anyone may join, so classical BFT (which needs a known node set to count votes) does not apply. Instead, the right to append a block is won through a **randomized leader election** that is costly to win and cheap to verify — a Sybil-resistance mechanism (one adversary must not be able to forge many identities and outvote everyone):

- **Proof of Work (Bitcoin):** miners race to find a nonce with `hash(block) < target`. The first to solve broadcasts the block; others verify the hash trivially and extend it. Forks are resolved by converging on the chain with the **most accumulated work** (loosely, the "longest chain") — these diverge when difficulty changes, since a shorter chain of harder blocks can outweigh a longer chain of easier ones. Rewriting history requires out-mining the rest of the network — economically infeasible beyond a few confirmations (the "wait 6 blocks" rule), which is why finality is *probabilistic*, not absolute. Security comes at the cost of enormous energy use. Throughput is bounded by the block-size / block-interval tradeoff: larger blocks raise throughput but propagate slower, increasing fork rate.
- **Proof of Stake:** the next proposer is chosen with probability proportional to staked wealth, eliminating mining's energy cost. Attacking requires acquiring a large fraction of the stake, which is self-defeating (it devalues the very asset staked).

For a backend engineer the takeaway is the framing: blockchains solve the same agreement problem as Raft or PBFT, but trade latency and throughput for open membership and Byzantine, adversarial trustlessness. When membership *can* be controlled, a classical consensus protocol is dramatically cheaper.

---

## 24. Common Patterns and Anti-Patterns

### Patterns

**Idempotent retries with exponential backoff and jitter:**

```python
def call_with_retry(fn, max_attempts=5, base_delay=0.5):
    for attempt in range(max_attempts):
        try:
            return fn()
        except RetriableError:
            delay = base_delay * (2 ** attempt) * random.uniform(0.5, 1.5)
            time.sleep(min(delay, 30))
    raise
```

**Backoff** grows the wait after each failure (here roughly 0.5s, 1s, 2s, 4s, 8s, capped at 30s) so a struggling service gets room to recover instead of being hammered. **Jitter** multiplies each wait by a random factor so clients don't synchronize: without it, a thousand clients that fail at the same instant would all retry at the same instant, fail together, and keep marching in lockstep — a self-inflicted spike called a thundering herd.

This only works on **idempotent** operations — ones that give the same result whether run once or many times (`set balance = 100`, not `add 100 to balance`). Retries are needed because a request may have actually succeeded on the server while its acknowledgement was lost in transit; you then retry and it runs again. If the operation is idempotent that second run is harmless; if not, you double-charge the account.

**Circuit breakers:** When a downstream is failing, stop sending requests for a window so it can recover. Track error rates; trip the breaker when error rate exceeds threshold.

```
States:
  Closed:    pass requests, count failures
  Open:      reject immediately for cooldown period
  Half-open: allow a few probe requests; if they succeed, close; else re-open
```

**Bulkheads:** Isolate failures by partitioning resources (thread pools, connection pools) per downstream. One slow dependency exhausts its own pool, not the entire process.

**Timeouts everywhere:** Every network call has a finite timeout. The default "wait forever" is the source of cascading failures — when one service slows down, every caller hangs.

**Idempotency keys:** see section 20.

**Read-repair:** Replicated reads check consistency; if replicas disagree, fix in the background. (The production form of the register write-back in section 14.)

**Hinted handoff:** When a replica is down, accept the write on another node with a "hint" to deliver when the target recovers. Trades strict replica placement for availability.

### Anti-Patterns

**Hidden distributed systems:** treating remote calls like local function calls. Latency, partial failure, and timeouts ambush code that ignores them. Always treat network calls as fallible and slow.

**Per-request fan-out without limits:** "for each item in the list, call the downstream service" — but the list is unbounded. One request triggers thousands of downstream calls.

**Synchronous chains:** A->B->C->D, each blocking on the next. Latency adds; one failure cascades. Decouple with async messaging where possible.

**Distributed transactions for performance-critical paths:** 2PC has high latency. Sagas or single-shard transactions are usually better.

**Reinventing consensus:** writing custom leader election or "distributed locking" code on top of inconsistent storage. Use etcd / ZooKeeper / Consul; they have already been debugged for decades of edge cases.

**Trusting wall-clock comparisons across machines:** comparing timestamps from different nodes to decide ordering. Clocks drift; use logical clocks or version vectors.

**Assuming the network is reliable:** see the eight fallacies. Code that doesn't retry, doesn't time out, doesn't handle duplicates, will fail in production.

**Tight coupling between services via shared databases:** two services writing to the same database create a hidden distributed transaction. Each service owning its own data and communicating via APIs/events is messier but safer.

### A Final Heuristic

The cost of distribution is real. The best distributed system is the one that proves unnecessary.

- If a single large database can handle the load, use one.
- If a single region with replicas can handle it, use that.
- Active-active multi-region is rarely actually required. Most products do not need it.
- Exactly-once across systems is rarely actually required. Idempotent processing is usually enough.

Distribute when scale, latency, or availability requires it. Not because the architecture diagram looks better.
