# 04 — Distributed data, consistency and coordination

What changes about **data and agreement** once state is spread across services and nodes:
no single transaction, no single clock, no single source of truth unless you design one.
This file covers how each service owns data, how multi-step work stays correct (saga, outbox),
how reads are served across services (CQRS, event sourcing), how data is spread (replication,
sharding) and how nodes agree (leader election, consensus).

Part of the [design patterns guide](README.md). Previous: [architecture styles and decomposition](03-architecture-styles-and-decomposition.md). Next: [distributed communication, resilience and scaling](05-distributed-communication-resilience-and-scaling.md).

> **Status: preview — not yet read.** Underlying theory: [CAP theorem](../architecture/cap-theorem.md), [distributed systems notes](../notes/distributed-systems-interview-notes.md), [data architecture](../data/data-architecture.md).

**Running example:** booking a flight now spans `Inventory`, `Booking`, `Payment` and `Ticketing`,
each with its own database.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Ownership and cross-service consistency](#part-a--ownership-and-cross-service-consistency)
3. [Part B — Reads across services](#part-b--reads-across-services)
4. [Part C — Spreading data: replication and partitioning](#part-c--spreading-data-replication-and-partitioning)
5. [Part D — Coordination](#part-d--coordination)
6. [Part E — Consistency models](#part-e--consistency-models)
7. [Level 3 — deep dive](#level-3--deep-dive)
8. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

Splitting data across services and machines breaks three comfortable assumptions:

| You used to have… | Now you have… | Pattern family |
|---|---|---|
| One database transaction around the whole operation | Several local transactions with no shared commit | Saga, outbox, idempotency |
| Joins across all your data | Data owned by different services | API composition, CQRS views, event-carried state |
| One machine deciding who's in charge, and one clock | Many nodes that must agree despite failures and clock drift | Leader election, quorum, consensus, logical clocks |

The core rule: **each service owns its data; nobody reads another service's tables.** Everything
else here is how to live with that rule. Idempotency — safely repeating an operation — is the
glue that makes most of these patterns work; its home is the
[Idempotent Receiver](06-integration-patterns.md#idempotent-receiver) in the integration file.

**Check yourself (L1):** (1) Why can't a normal DB transaction cover *reserve seat → charge card → issue ticket*? (2) What is the one rule that creates all these problems?

---

## Part A — Ownership and cross-service consistency

### Database per service
Each service has a private store (can be a different technology — polyglot persistence). Gains: independent evolution and scaling, no schema coupling. Costs: **no joins, no ACID across services**, duplicated reference data, harder reporting. The rest of this part exists to pay that bill. See [data architecture](../data/data-architecture.md).

### Shared database (and why it's usually an anti-pattern)
Services share one schema. Easy at first; creates hidden coupling — any schema change risks every service, and you can't scale or deploy independently. Acceptable only temporarily during a [strangler](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation) migration or in service-based architecture.

### Saga
**Problem:** booking needs *reserve seat → take payment → issue ticket* across three services, each with its own DB; there's no distributed transaction.
**Idea:** a **sequence of local transactions**, each publishing an event/command that triggers the next. If a step fails, run **compensating transactions** to undo the earlier steps (semantic rollback — "refund", "release seat").

Two styles:

| | Choreography | Orchestration |
|---|---|---|
| Control | Services react to each other's events | A central orchestrator tells each service what to do |
| Coupling | Low, but flow is implicit and spread out | Orchestrator knows all participants |
| Visibility | Hard to see the whole flow | One place shows the flow and state |
| Fits | 2–4 steps, simple | Long, branching, or business-critical flows |
| Tools | Kafka/SNS events | Temporal, Camunda, AWS Step Functions, Spring State Machine, Axon |

**Pseudocode — orchestrated booking saga**

```
START saga(bookingRequest)
  1. seat   = Inventory.reserveSeat(request)            -- local tx in Inventory
        IF fails  → END saga as FAILED
  2. pay    = Payment.charge(request)                   -- local tx in Payment
        IF fails  → Inventory.releaseSeat(seat); END as FAILED     -- compensate step 1
  3. ticket = Ticketing.issue(request)                  -- local tx in Ticketing
        IF fails  → Payment.refund(pay);
                    Inventory.releaseSeat(seat);        -- compensate in REVERSE order
                    END as FAILED
  4. mark booking CONFIRMED; publish BookingConfirmed
END
```

*Reading it:* each step commits on its own, so other users can briefly see intermediate states
(no isolation). Every step needs a compensation, and **every step and compensation must be
idempotent** because messages get redelivered. Persist saga state so a crash resumes instead of
losing track.

- **Gotchas:** lack of isolation (use semantic locks like `PENDING` status, commutative updates, versioning); compensations can fail too (retry forever, then alert a human); pivot step — after the point of no return, you go forward not back.
- **Detail:** [Choreography vs orchestration](../backend-and-messaging/messaging-and-event-driven-architecture.md) and [semantic coupling, step count and sagas](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Two-Phase Commit (2PC) — and why it's mostly avoided
Coordinator asks all participants to *prepare* (vote), then *commit* if all say yes. Gives atomicity but **blocks** while locks are held, a coordinator failure can leave participants stuck, and it hurts availability and latency. Fine inside one database cluster; rejected between microservices. Compare with Saga in [microservices notes §4](../notes/microservices.md).

### Transactional outbox
**Problem (dual write):** "save the booking **and** publish `BookingConfirmed`" — doing both can leave one done and the other lost.
**Idea:** in the **same local transaction**, write the business row **and** an `outbox` row. A separate relay publishes outbox rows to the broker and marks them sent. At-least-once publishing → consumers must be idempotent.

```
BEGIN TX
   INSERT booking …
   INSERT outbox(event_id, type, payload) …
COMMIT                                         -- atomic: both or neither

RELAY (loop, or CDC on the outbox table):
   rows = SELECT unsent FROM outbox ORDER BY id
   FOR each row: publish(row) ; mark row sent   -- may publish twice if it crashes between
```

- **Relay options:** polling publisher, or **CDC** (Debezium reads the DB log).
- **Detail:** [reliable publishing / dual write](../backend-and-messaging/messaging-and-event-driven-architecture.md), [outbox + CDC deep-dive](../notes/event-driven-architecture-notes.md).
- **Mirror image — Inbox:** store processed message IDs in the consumer's DB to dedupe.

### Change Data Capture (CDC) / Log tailing
Stream database changes from the transaction log to other systems (search index, cache, warehouse, other services). Avoids dual writes and application-level publishing. Tools: Debezium, DMS. Beware: it exposes your table schema as a contract — publish through an outbox or a contract view.

### Tombstones, soft delete and compaction
Deleting in a distributed log means writing a marker (tombstone) that downstream compaction respects. See Kafka retention/compaction in the messaging note.

---

## Part B — Reads across services

Because you cannot join across services, reads need a plan: compose at call time, or pre-build a read model from events.

### API composition and CQRS views (queries across services)
Because you can't join across services:
- **API Composition:** a composer (API gateway/BFF) calls several services and merges results in memory. Simple; latency = slowest call; poor for large joins/pagination across services.
- **CQRS read model (materialized view):** a service subscribes to events from several services and keeps a pre-joined, query-optimised store. Fast reads; eventual consistency; more moving parts.

### CQRS — Command Query Responsibility Segregation
**Idea:** use a different model for **writes** (commands, enforce rules) and **reads** (queries, shaped for screens).

- **Levels:** (1) separate classes/methods, same DB → (2) separate read model/tables, updated synchronously → (3) separate read store, updated asynchronously via events (eventual consistency).
- **Use when:** reads and writes have very different shapes or scale; complex domain with simple reporting; many read-optimised views.
- **Avoid when:** simple CRUD — you'll double your code for no gain. It's a pattern for *parts* of a system, not the whole thing.
- **Detail:** [messaging note — CQRS in practice](../backend-and-messaging/messaging-and-event-driven-architecture.md) and [EDA notes — CQRS](../notes/event-driven-architecture-notes.md).

### Event Sourcing
**Idea:** store the **sequence of events** (`SeatReserved`, `PaymentTaken`, `BookingCancelled`) as the source of truth; derive current state by replaying them.

- **Wins:** perfect audit trail, time travel, rebuild any projection, natural fit for event-driven integration.
- **Costs:** querying needs projections (so usually CQRS), event schema evolution is hard (upcasting), snapshots for long streams, GDPR deletion is awkward (crypto-shredding), a steep learning curve.
- **Use when:** audit is a first-class requirement or domain is naturally a ledger (payments, inventory, bookings history). **Don't** use as a default.
- **Detail:** [when do you actually need event sourcing?](../notes/event-driven-architecture-notes.md)

- **Distributed role:** the event log is a durable integration backbone; every consumer can rebuild its own read model by replaying it.

### Event-carried state transfer
Events carry enough data (booking id, passenger, fare) so consumers keep a local copy and don't call back. Reduces runtime coupling and latency; costs data duplication and versioning. Compare "thin" vs "fat" events: [event vs command, fat vs thin](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Materialised view / precomputed results
Compute expensive aggregates ahead of time (daily availability summaries) and serve them cheaply; trade freshness for speed.

---

## Part C — Spreading data: replication and partitioning

### Replication
Copy data to more nodes for availability and read scale.
- **Leader–follower (primary–replica):** writes to the leader, reads from followers; simple; replication lag → stale reads.
- **Multi-leader:** writes in several regions; needs conflict resolution.
- **Leaderless (Dynamo-style):** any node accepts writes; quorum `R + W > N`; read repair, hinted handoff.
- Detail: [replication](../data/data-architecture.md) and [read replicas](../notes/distributed-systems-interview-notes.md).

### Partitioning / Sharding
Split data across nodes by a key. Strategies: **hash** (even, no range scans), **range** (range scans, hotspots), **directory/lookup** (flexible, a lookup is a bottleneck), **geo**. Pick a key with high cardinality and even access; beware hot keys (celebrity flight, Black Friday). Rebalancing is the hard part → [consistent hashing](../data-structures/systems-data-structures.md).
Detail: [partitioning and sharding](../data/data-architecture.md).

### Consistent hashing
Map nodes and keys onto a ring; each key belongs to the next node clockwise. Adding/removing a node moves only ~1/N of keys. Virtual nodes smooth the load. Used by Dynamo, Cassandra, memcached clients, CDNs.

### Sharded counters, bloom filters, and probabilistic structures
Spread a hot counter across N keys and sum on read; use Bloom filters/HyperLogLog/Count-Min to answer "have I seen this?" / "how many distinct?" cheaply. See [systems data structures](../data-structures/systems-data-structures.md).

---

## Part D — Coordination

Idempotency and exactly-once *effect* are covered by the [Idempotent Receiver](06-integration-patterns.md#idempotent-receiver).

### Leader election
Only one node should do a job (run the scheduler, own a partition). Nodes elect a leader; if it dies, another takes over. Implementations: ZooKeeper/etcd/Consul lease, Raft, database row lock, Kubernetes lease. Danger: **split brain** — two nodes both think they lead.

### Distributed lock, lease and fencing token
A lock with an expiry (**lease**) so a crashed holder can't block forever. But a paused holder (GC pause) may wake up after its lease expired and still write — so each lock grant carries a monotonically increasing **fencing token** that the resource checks and rejects if older.

```
acquire(lock):  token = ++counter ; lease until now+10s ; return token     -- e.g. 33
holder writes:  storage.write(data, token=33)
storage:        IF token < highestSeenToken → REJECT   -- stale holder (e.g. token 32) loses
```

Prefer avoiding distributed locks: use idempotent operations, single-writer partitions, or optimistic versioning. Red-lock-style Redis locks are contentious for correctness-critical use.

### Consensus (Raft / Paxos)
Nodes agree on a value/log despite failures, as long as a **majority (quorum)** is alive. Foundation of etcd, Consul, ZooKeeper (ZAB), Kafka KRaft, Spanner. With `N = 2f + 1` nodes you tolerate `f` failures; use odd cluster sizes.

### Quorum
Require agreement of a majority so any two quorums overlap. For replicated data `R + W > N` guarantees a read overlaps the latest write.

### Heartbeat and failure detection
Nodes send periodic "I'm alive" signals; absence beyond a timeout marks a node suspect. Trade-off: fast detection vs false positives. Phi-accrual detectors adapt to network variance.

### Gossip protocol
Nodes periodically exchange state with random peers; information spreads exponentially, with no central coordinator. Cassandra membership, Consul (Serf), DynamoDB-style systems.

### Write-Ahead Log (WAL)
Append the change to a durable log *before* applying it, so a crash can replay. Used by databases, Kafka, Raft.

### Logical clocks and ordering
Wall clocks drift, so "just use timestamps" fails. **Lamport timestamps** give a consistent happens-before order; **vector clocks** detect concurrency/conflicts; **hybrid logical clocks** combine physical + logical. Use for conflict detection in multi-leader/leaderless systems.

### Conflict resolution
When two replicas change the same data: last-write-wins (loses data), application merge, **CRDTs** (data types that always merge), operational transform.

### Singleton work and scheduler patterns
Run a cron-like job once across a fleet: leader election, a lock with a lease, ShedLock, or a managed scheduler (Quartz clustered, Kubernetes CronJob).

---

## Part E — Consistency models

Strong (linearizable), sequential, causal, read-your-writes, monotonic reads, eventual. Pick the *weakest* model the use case tolerates — e.g. a booking confirmation page needs read-your-writes; a "seats left" counter can be eventual. See [CAP/PACELC and the consistency spectrum](../architecture/cap-theorem.md).

---

## Level 3 — deep dive

### Which data pattern for which problem

| Problem | Pattern |
|---|---|
| Multi-step business operation across services | Saga (+ compensation, idempotency) |
| Save to DB and publish an event atomically | Transactional outbox (or CDC) |
| Need a screen that joins data from three services | API composition, or a CQRS read model |
| Consumers keep calling back for the same data | Event-carried state transfer |
| Need a full audit trail / replay | Event sourcing |
| One database can't take the writes | Sharding (hash/range/directory), consistent hashing |
| One database can't take the reads | Replicas (accept lag) or cache ([file 05](05-distributed-communication-resilience-and-scaling.md#caching-patterns)) |
| Only one node may do a job | Leader election with lease and fencing token |
| Nodes must agree on a value or order | Consensus (Raft/Paxos), quorum |
| Two replicas changed the same record | Last-write-wins, application merge, CRDT |

### Pattern dependencies — what needs what

| If you adopt… | You also need… |
|---|---|
| Database per service | Saga/outbox, API composition or CQRS views, idempotency |
| Asynchronous messaging | Idempotent receivers, DLQ, schema evolution, observability |
| Event sourcing | CQRS projections, upcasting, snapshots |
| Leader election / locks | Fencing tokens or idempotent writes |
| Multi-leader replication | Conflict resolution (CRDT, merge, LWW) |
| Sharding | A hot-key plan and a rebalancing plan |

### Anti-patterns here
Shared database between services · 2PC across services · dual write ("save, then publish") · ordering by wall-clock time across machines · a distributed lock without fencing · exactly-once assumed. Full catalogue: [anti-patterns](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue).

### Checklist for any cross-service data interaction
1. Who owns this data? Who else holds a copy, and how stale may it be?
2. What if the second step fails after the first committed? What compensates?
3. Is each step idempotent? What is the dedupe key?
4. What does the user see in between (pending state)? Which consistency model do they need?
5. What happens if this message arrives out of order, late, or twice?
6. Can I trace this flow end to end? ([correlation id](08-cross-cutting-concerns.md#part-b--observability-patterns))

---

## Check yourself

1. The booking saga's ticketing step fails after payment. List the compensations in order, and say why each must be idempotent.
2. What problem does the outbox solve, and why must consumers still be idempotent?
3. Orchestration or choreography for a 7-step refund with branches? Why?
4. Why is 2PC rejected between microservices?
5. API composition vs a CQRS read model: when does each break down?
6. When would you add event sourcing to the Booking service, and what does it force you to add next?
7. A paused leader wakes after a new leader was elected. How do fencing tokens protect storage?
8. Why is `R + W > N` enough to guarantee reading the latest write?
9. Hash vs range sharding: one advantage and one failure mode each. What is a hot key?
10. Why do wall clocks fail as an ordering mechanism, and what replaces them?
11. Which consistency model does a booking confirmation page need, and which does a "seats left" counter need?

Previous: [← 03 Architecture styles and decomposition](03-architecture-styles-and-decomposition.md) · Next: [05 Distributed communication, resilience and scaling →](05-distributed-communication-resilience-and-scaling.md)
