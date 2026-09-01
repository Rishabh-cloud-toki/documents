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

---

## One-line statement

**In the presence of a network partition, a distributed data store cannot be
both consistent and available; it must sacrifice one of them.**

CAP is about the behaviour of a replicated data store *when the network between
replicas fails*. When there is no partition, a system can be both consistent and
available — CAP says nothing against that.

---

## The three properties, precisely

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
