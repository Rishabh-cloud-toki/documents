# CAP Theorem

Precise reference notes on the CAP theorem — what it actually states, how each
term is formally defined, the common misreadings, and how it is applied in
practice (including the PACELC extension).

## Contents

- [One-line statement](#one-line-statement)
- [The three properties, precisely](#the-three-properties-precisely)
- [The formal theorem](#the-formal-theorem)
- [Why you cannot have all three during a partition](#why-you-cannot-have-all-three-during-a-partition)
- [The real choice: CP vs AP](#the-real-choice-cp-vs-ap)
- [Common misconceptions](#common-misconceptions)
- [PACELC — the more complete model](#pacelc--the-more-complete-model)
- [Consistency is a spectrum](#consistency-is-a-spectrum)
- [Worked example](#worked-example)
- [System classifications](#system-classifications)
- [How to use CAP in a design discussion](#how-to-use-cap-in-a-design-discussion)
- [Sources](#sources)
- [Appendix: Q&A deep-dives](#appendix-qa-deep-dives)

---

## One-line statement

**In the presence of a network partition, a distributed data store cannot be
both consistent and available; it must sacrifice one of them.**

CAP is about the behaviour of a replicated data store *when the network between
replicas fails*. When there is no partition, a system can be both consistent and
available — CAP says nothing against that.

---

## The three properties, precisely

> Deep-dive: [distributed data store and network partition](#deep-dive-distributed-data-store-and-network-partition)
> — the two terms in plain language, and the two-options walk-through of a write
> that arrives during a partition.

The definitions below are the ones used in the formal proof (Gilbert & Lynch,
2002), not the loose colloquial versions.

### C — Consistency (linearizability)

Every read receives the most recent completed write, or an error. Formally, the
system behaves as if there is a single copy of the data and every operation takes
effect atomically at some instant between its invocation and its response
(**linearizability**, a.k.a. atomic or strong consistency).

- This is **not** the "C" of ACID. ACID consistency means "transactions preserve
  database invariants". CAP consistency means "all nodes see the same data at the
  same time / operations appear to occur in a single total order".

### A — Availability

Every request received by a **non-failing** node must result in a **non-error
response** — the node is not allowed to reject the request or hang indefinitely.

- The response is allowed to be *slow*, but it must eventually come and it must
  not be an error.
- Crucially, availability here means **every** non-failing node stays responsive,
  even one that has been cut off from all the others.
- A response that returns *stale* data still counts as available (it just may not
  be consistent).

### P — Partition tolerance

The system continues to operate despite an arbitrary number of messages between
nodes being dropped or delayed by the network. A partition is a split of the
nodes into groups that cannot communicate with each other.

- Partition tolerance is not really a "choice" for any real distributed system:
  networks *do* partition (switch failures, GC pauses, misconfiguration, cross-AZ
  or cross-region link loss). If your nodes talk over a network, you must tolerate
  partitions.

---

## The formal theorem

> It is impossible for a distributed data store to simultaneously provide all
> three of **Consistency**, **Availability**, and **Partition tolerance**.

More useful phrasing, since P is not optional:

> **When a partition occurs, you must choose between Consistency and
> Availability.** When there is no partition, you can have both.

So CAP forces a binary decision **only during a partition**:

| Choice | Behaviour during a partition |
|---|---|
| **CP** (choose Consistency) | Refuse or block requests that can't be served consistently — return errors or time out until the partition heals. |
| **AP** (choose Availability) | Keep serving every node, accepting that different nodes may return different (stale or conflicting) data until the partition heals and state is reconciled. |

"CA" — consistent and available but not partition-tolerant — is only meaningful
for a single node or a system that assumes the network never fails. It is not a
sensible category for a real distributed system; a "CA" system simply loses
consistency *or* availability the moment a partition happens.

---

## Why you cannot have all three during a partition

Sketch of the impossibility argument:

1. Suppose the network partitions nodes into two groups, **G1** and **G2**, that
   cannot exchange any messages.
2. A client writes value `v2` (over a previous `v1`) to a node in **G1**. That
   node cannot propagate the write to **G2** because of the partition.
3. Another client now reads from a node in **G2**.
4. Two options:
   - The **G2** node responds with `v1` (the value it still has). The system is
     **available** but **not consistent** — the read did not return the latest
     write.
   - The **G2** node refuses to answer (error or block) until it can confirm it
     has the latest data. The system is **consistent** but **not available**.
5. There is no third option: with no messages crossing the partition, a node in
   **G2** cannot both know about `v2` and return a non-error response.

---

## The real choice: CP vs AP

> Deep-dives: [read replicas, and scaling reads vs writes](#deep-dive-read-replicas-and-scaling-reads-vs-writes)
> — why a single primary, replication lag and read-your-own-writes, and sharding
> for write scale; [keeping a separate read model up to date](#deep-dive-keeping-a-separate-read-model-up-to-date)
> — RDS replica vs a different engine, dual writes / outbox / CDC, and the
> idempotency and ordering rules.

### CP — consistency over availability

- During a partition, the minority side (and sometimes both sides) stops
  accepting reads and/or writes rather than risk serving stale or divergent data.
- Typically implemented with a **quorum / consensus** protocol (Paxos, Raft,
  Zab): an operation only succeeds if a majority of replicas acknowledge it, so a
  minority partition cannot make progress.
- **Use when** correctness matters more than uptime for that operation: financial
  ledgers, inventory decrements, unique-constraint enforcement, leader election,
  configuration/coordination stores.

### AP — availability over consistency

- During a partition, every reachable replica keeps answering reads and accepting
  writes. Divergent versions are reconciled after the partition heals via
  **conflict resolution** (last-write-wins, version vectors, CRDTs,
  application-level merge).
- Result is **eventual consistency**: if writes stop, all replicas converge to
  the same value after some time.
- **Use when** uptime and low latency matter more than always-fresh reads:
  shopping carts, user sessions, social feeds, product catalogues, telemetry,
  DNS, caches.

The choice is **per-operation, not per-system**. A single product can be CP for
"place order / take payment" and AP for "show product page / recommendations".

---

## Common misconceptions

| Misconception | Reality |
|---|---|
| "Pick 2 of 3." | You don't freely pick 2. P is forced on any networked system, so the only real choice is **C vs A, and only during a partition**. |
| "CAP's C is the same as ACID's C." | No. CAP-C = linearizability (single-copy illusion). ACID-C = invariant preservation by transactions. |
| "A CA system exists." | Only as a single node or under an assumption that the network never fails. Not a meaningful distributed-systems category. |
| "NoSQL = AP, SQL = CP." | Wrong. Many NoSQL stores can be tuned CP or AP (e.g. Cassandra per-query consistency levels; MongoDB is CP by default). Distributed SQL (Spanner, CockroachDB) is CP. |
| "CAP is a permanent, system-wide setting." | The trade-off only bites *during a partition*; outside partitions you get both C and A. And it can be chosen per operation. |
| "Choosing AP means giving up consistency forever." | AP systems are usually *eventually* consistent — they converge once the partition heals. |
| "Latency has nothing to do with CAP." | CAP itself ignores latency, but the practical cost of consistency is latency even with no partition — see PACELC. |

---

## PACELC — the more complete model

> Deep-dive: [PACELC — the "else" branch and the physics](#deep-dive-pacelc-the-else-branch-and-the-physics)
> — why the else branch exists at all, the network round-trip bounds that force
> it, and RDS async (EL) vs Multi-AZ synchronous (EC).

Proposed by Daniel Abadi (2012) to capture what CAP leaves out: the
**consistency/latency trade-off that exists even when the network is healthy**.

> **PAC** — if there is a **P**artition, choose between **A**vailability and **C**onsistency.
> **ELC** — **E**lse (normal operation), choose between **L**atency and **C**onsistency.

Reasoning for the "else" clause: to guarantee strong consistency you must
coordinate replicas (quorum reads/writes, synchronous replication), and that
coordination adds latency on **every** request, partition or not. Relax
consistency and you can serve from the nearest replica with no coordination —
lower latency.

| System | PACELC classification | Meaning |
|---|---|---|
| DynamoDB / Cassandra (default) | **PA/EL** | Favours availability under partition, latency otherwise |
| Google Spanner | **PC/EC** | Favours consistency always (uses TrueTime + Paxos, pays the latency) |
| CockroachDB | **PC/EC** | Strong consistency always |
| MongoDB (default) | **PC/EC** | Consistency-leaning |
| Cassandra with `QUORUM` reads+writes | **PC/EC** | Can be tuned toward consistency |
| PNUTS (Yahoo) | **PC/EL** | Consistent under partition, latency-favouring otherwise |

PACELC is a better interview answer than CAP alone because it acknowledges that
**most of the time your system is not partitioned**, and the consistency cost you
pay day-to-day is latency.

---

## Consistency is a spectrum

CAP treats consistency as binary (linearizable or not). In practice there is a
hierarchy of models, from strongest to weakest:

1. **Linearizable / strong** — single-copy illusion; every read sees the latest write.
2. **Sequential consistency** — all nodes see operations in the same order, not necessarily real-time order.
3. **Causal consistency** — operations that are causally related are seen in order by everyone; concurrent operations may be seen in different orders.
4. **Read-your-writes / monotonic reads / monotonic writes** — useful "session" guarantees.
5. **Eventual consistency** — if writes stop, replicas eventually converge; no ordering guarantee in the meantime.

AP systems typically land at causal or eventual; CP systems provide linearizable
or sequential.

---

## Worked example

A key-value store replicated across nodes **N1**, **N2**, **N3**. Current value:
`x = 10`.

**Normal operation (no partition):** client writes `x = 20` to N1; N1
synchronously replicates to N2 and N3; all subsequent reads on any node return
`20`. Both consistent and available.

**Partition:** the link drops, isolating **N3** from **N1/N2**.

- **CP design:** writes require a majority ack. `x = 30` written via N1 succeeds
  (N1 + N2 = majority). A client reading from the isolated **N3** gets an
  **error / timeout** — N3 knows it might be stale and refuses to answer.
  Consistency preserved, N3 unavailable.
- **AP design:** every node keeps serving. `x = 30` goes to N1; meanwhile another
  client writes `x = 40` to the isolated N3. Reads return whatever the local node
  has (`30` on N1/N2, `40` on N3). When the partition heals, the system detects
  the conflict and resolves it (e.g. last-write-wins by timestamp, or surfaces
  both versions via version vectors for the application to merge). Availability
  preserved, consistency temporarily lost.

---

## System classifications

Rough groupings (many are tunable — treat as defaults, not absolutes):

| CP-leaning | AP-leaning |
|---|---|
| ZooKeeper, etcd, Consul | Amazon Dynamo, Cassandra (default) |
| Google Spanner, CockroachDB, YugabyteDB | Riak, Voldemort |
| HBase, MongoDB (default) | CouchDB |
| Redis Sentinel/Cluster (mostly CP) | DynamoDB (eventually-consistent reads) |
| Kafka (with `acks=all`, `min.insync.replicas`) | DNS, web caches, CDNs |
| Relational DBs with synchronous replication | Relational DBs with async read replicas |

---

## How to use CAP in a design discussion

1. **Don't say "I'll pick AP" or "I'll pick CP" for the whole system.** Say which
   *operations* need linearizability and which can tolerate staleness.
2. **Name the partition behaviour explicitly:** "During a partition, the payment
   path returns 503 and we queue the request; the catalogue path keeps serving
   from local replicas and reconciles afterward."
3. **State the reconciliation strategy** for any AP path: LWW, version vectors,
   CRDTs, or human/application merge.
4. **Bring in PACELC:** "Even with no partition, strong consistency here costs us
   a cross-region round trip per write, so for the feed we accept eventual
   consistency to keep p99 latency under X."
5. **Quantify the staleness window** an AP choice implies and give the business
   the reconciliation rule ("seat map may be up to ~200 ms stale; on conflict the
   earlier confirmed booking wins").

---

## Sources

- E. Brewer, "Towards Robust Distributed Systems" (PODC 2000 keynote) — the
  original CAP conjecture.
- S. Gilbert & N. Lynch, "Brewer's Conjecture and the Feasibility of Consistent,
  Available, Partition-Tolerant Web Services" (ACM SIGACT News, 2002) — the formal
  proof.
- E. Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed" (IEEE
  Computer, 2012) — clarifies that the "2 of 3" framing is misleading.
- D. Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design"
  (IEEE Computer, 2012) — introduces PACELC.
- M. Kleppmann, *Designing Data-Intensive Applications* (2017), ch. 5 & 9 —
  replication, linearizability, and a critique of CAP's precision.

---

## Appendix: Q&A deep-dives

Plain-language walk-throughs from working through this note — the questions that
needed more than the reference above. Full transcript:
[distributed-systems-interview-notes.md](../notes/distributed-systems-interview-notes.md).
Each block links back to the section it belongs to.

- [Distributed data store and network partition](#deep-dive-distributed-data-store-and-network-partition) — the three properties
- [Read replicas, and scaling reads vs writes](#deep-dive-read-replicas-and-scaling-reads-vs-writes) — CP vs AP
- [Keeping a separate read model up to date](#deep-dive-keeping-a-separate-read-model-up-to-date) — CP vs AP
- [PACELC: the "else" branch and the physics](#deep-dive-pacelc-the-else-branch-and-the-physics) — PACELC

### Deep-dive: distributed data store and network partition

Relates to [The three properties, precisely](#the-three-properties-precisely) and
[Why you cannot have all three during a partition](#why-you-cannot-have-all-three-during-a-partition).

**Distributed data store** — a database whose data doesn't live on one machine;
the same data is copied onto several nodes / replicas, often in different regions.
Why: survive a machine dying, serve users nearby, spread load. The catch — those
copies must be kept in agreement, and that only works while the machines can talk.

**Network partition** — the machines are alive and healthy, but the network
*between* them breaks (cut cable, failed switch, dead cross-region link). Each
side keeps running and serving users; neither can reach the other, and neither
can tell whether the other is dead or merely unreachable.

```
   Writes X = 2                      Reads X
        |                               |
        v                               v
  +-------------+                 +-------------+
  |  Replica A  |----- X X X -----|  Replica B  |
  |   Europe    |   link broken   |    India    |
  +-------------+                 +-------------+
```

A user writes `X = 2` to Replica A. A can't pass it to B. Exactly two options:

1. **Accept the write anyway** — the system stays available, but B keeps serving
   the old X. The copies disagree: consistency lost.
2. **Refuse the write until it can reach B** — the copies stay in agreement, but
   the user gets an error or a hang: availability lost.

There is no third option, because A cannot magically reach B. Two things
interviewers probe: **partition tolerance isn't really a choice** — networks
fail, so any real distributed system must survive partitions, and the actual
decision is only C vs A (hence "CP or AP"); and **with no partition you keep
both** — the trade-off appears only when the link breaks.

### Deep-dive: read replicas, and scaling reads vs writes

Relates to [The real choice: CP vs AP](#the-real-choice-cp-vs-ap) — the reason a
single primary exists is that *one place deciding write order* is what makes
consistency cheap.

**The confusion to untangle:** "write service" and "database primary" are
different layers. The application tier usually runs many identical copies behind a
load balancer — already replicated. But all those copies still write to *one*
database primary, so scaling the app tier doesn't help when the database itself is
the bottleneck.

**Why reads copy easily but writes don't.** A read changes nothing, so ten
machines can answer the same read independently and all be correct. A write
changes things; if two machines accept conflicting writes to the same row at the
same time, something has to decide which wins. Funnelling all writes through one
primary makes that ordering trivial — one place decides. Multi-writer setups
(Aurora Multi-Master, Galera, CockroachDB) exist, but pay with coordination
latency on every write or conflict-resolution rules you design around — a far
bigger commitment than adding a read replica.

**Do read replicas actually help?** Usually yes — most apps are 80–95% reads, and
relieving the primary of that load frees capacity for writes. Two caveats:

- **Replication lag** — a replica runs ms-to-seconds behind; a user who writes
  then immediately reads may not see their own change. Usual fix: route
  read-your-own-writes to the primary, everything else to replicas.
- **Replicas don't reduce write load at all** — every write still hits the
  primary, and each replica must replay it too. If writes are the bottleneck,
  adding read replicas makes it slightly worse.

**When writes are the bottleneck the answer is sharding, not more replicas:**
split data by a key (customer ID, region) so each shard has its own primary
handling only its slice of the writes. That is the real horizontal-scaling story
for writes — and the likely follow-up question.

### Deep-dive: keeping a separate read model up to date

Relates to [The real choice: CP vs AP](#the-real-choice-cp-vs-ap) (a separate read
model is eventually consistent and rebuildable — an AP-flavoured choice). Overlaps
with the outbox / CDC / stream–table material in
[messaging-and-event-driven-architecture.md](../backend-and-messaging/messaging-and-event-driven-architecture.md#appendix-qa-deep-dives).

It depends on whether the read store is the **same engine** or a **different** one.

**Same engine — an RDS read replica.** Nothing event-based happens at application
level: the engine ships its own write-ahead log (Postgres WAL, MySQL binlog) to
the replica, which replays it. Your code writes once and knows nothing about it;
the schema is identical. This is replication, *not* CQRS.

**A genuinely different read model** — writes to Postgres, reads from
Elasticsearch / Redis / a denormalized table. The engine can't help, so this is
normally event-driven:

- **Dual writes** (write Postgres, then write Elasticsearch) — no shared
  transaction, so if the second write fails the stores silently diverge and
  nothing tells you. Interviewers treat this as the wrong answer.
- **Transactional outbox** — write the business row and an event row into an
  outbox table in the *same* transaction (both land or neither); a separate relay
  reads new outbox rows and publishes them.
- **CDC (Debezium)** — a tool tails the database's own replication log and turns
  each committed row change into an event. No app changes, and you can't forget
  to emit; costs another piece of infrastructure and a schema coupled to your
  table structure.
- **Event sourcing** — the events *are* the source of truth; current state isn't
  stored, only the sequence of what happened, and read models are projections
  built by replay. Powerful, large commitment, rarely the right default.
- **Scheduled batch / materialized views** — a job rebuilds the read table every
  few minutes. No streaming infrastructure; fine when minutes-old data is
  acceptable.

**Three things to say about any event-based option:** delivery is
**at-least-once**, so the projection must be idempotent (key on an event ID or
version, ignore anything already applied); **ordering matters per entity, not
globally** (partition the topic by entity ID); and the read model is **eventually
consistent and rebuildable** by replaying the stream — a genuine advantage over
dual writes.

### Deep-dive: PACELC, the "else" branch and the physics

Relates to [PACELC — the more complete model](#pacelc--the-more-complete-model).

**Why the "else" branch exists.** If every read must return the very latest
write, the system has to check with other nodes before answering — either the
write waits for replicas to acknowledge, or the read contacts a quorum. Either
way somebody waits for a network round trip, and that round trip is bounded by
physics:

- Within one data centre: under a millisecond.
- Across availability zones: a few milliseconds.
- Mumbai to Frankfurt: roughly 100 ms — and no engineering removes it, because
  light doesn't go faster.

So: coordinate and be correct but slow, or skip coordination, answer from the
nearest replica, and accept possible staleness. This choice exists **on a
perfectly healthy network** — exactly the case CAP says nothing about.

**Concrete version:** an RDS read replica is asynchronous — writes return without
waiting, so writes stay fast and reads may be stale. That is choosing **EL**. RDS
Multi-AZ with synchronous replication makes the write wait for the standby — an
extra round trip, nothing lost. That is choosing **EC**.

| Class | Systems | Behaviour |
|---|---|---|
| PA/EL | Cassandra, DynamoDB, Riak | Up under partition, fast normally, tolerate staleness in both |
| PC/EC | HBase, BigTable, VoltDB, etcd | Correctness above all, in both situations |
| PA/EC | MongoDB (classic default config) | Available under partition, coordinates when healthy |
| PC/EL | Yahoo PNUTS | Refuses to serve inconsistently under partition, serves stale data when healthy |

**PC/EL shows the two halves are genuinely independent decisions**, not one
decision stated twice. One-line answer: CAP describes the failure case; PACELC
adds that even without failures you are constantly trading consistency against
latency — and most real systems make that second trade far more often, because
partitions are rare and requests are constant.
