# Distributed Systems — Interview Notes

CAP, read replicas, read-model updates, and PACELC.

---

## 1. The two terms behind CAP

**Distributed data store**
A database whose data does not live on one machine. The same data is copied onto
several servers (nodes / replicas), often in different data centres or countries.

Why people do it:
- If one machine dies, the data survives.
- Users are served by a machine near them.
- Load is spread out.

The catch: those copies now have to be kept in agreement, and that only works if
the machines can talk to each other.

**Network partition**
The machines are alive and healthy, but the network *between* them breaks — a cut
cable, a failed switch, a dead link between regions. Each side keeps running and
keeps serving users, but neither side can reach the other, and neither can tell
whether the other is dead or merely unreachable.

```
   Writes X = 2                      Reads X
        |                               |
        v                               v
  +-------------+                 +-------------+
  |  Replica A  |----- X X X -----|  Replica B  |
  |   Europe    |   link broken   |    India    |
  +-------------+                 +-------------+

  Neither replica knows if the other is alive.
```

---

## 2. CAP theorem

> In the presence of a network partition, a distributed data store cannot be both
> consistent and available; it must sacrifice one of them.

Using the picture above: a user writes `X = 2` to Replica A. A cannot pass the
update to B. A has exactly two options.

1. **Accept the write anyway.** The system stays available, but B keeps handing
   out the old value of X. The copies disagree — consistency is lost.
2. **Refuse the write until it can reach B.** Both copies stay in agreement, but
   the user gets an error or a hang — availability is lost.

There is no third option, because A cannot magically reach B.

**Two points interviewers probe:**

- **Partition tolerance is not really a choice.** Networks fail, so any real
  distributed system must survive partitions. The actual decision is only between
  C and A — hence "CP or AP".
- **With no partition you keep both.** A healthy system is consistent *and*
  available. The trade-off only appears when the link breaks.

**Rough examples**
- Lean CP (refuse rather than serve stale): traditional RDBMS, HBase, etcd.
- Lean AP (answer now, reconcile later): Cassandra, DynamoDB.

---

## 3. Read replicas — and why writes don't scale the same way

A common setup (e.g. Amazon RDS): one **primary** that takes all writes, plus
several **read replicas** that serve reads.

```
  [App server 1]   [App server 2]   [App server 3]
         \               |               /
          \              |              /
           +--------> +---------+ <----+
                      | Primary |
                      | all     |
                      | writes  |
                      +---------+
                     /     |     \
                    /      |      \
        [Read replica] [Read replica] [Read replica]

  The app sends read queries straight to the replicas,
  not through the primary.
```

### The confusion to untangle
"Write service" and "database primary" are two different layers.

The **application tier** can and usually does run many identical copies behind a
load balancer — that part is already replicated. But all those copies still write
to *one* database primary. So scaling the app tier does not help when the database
itself is the bottleneck.

### Why reads copy easily but writes don't
- A read changes nothing, so ten machines can answer the same read independently
  and all be correct.
- A write changes things. If two machines accept conflicting writes to the same
  row at the same time, something has to decide which wins. Funnelling all writes
  through one primary makes that ordering trivial — one place decides.

Multi-writer setups exist (Aurora Multi-Master, MySQL Galera, CockroachDB), but
they pay with either coordination latency on every write or conflict-resolution
rules you must design around. A far bigger commitment than adding a read replica.

### Do read replicas actually help?
Usually yes — most business applications are read-heavy (often 80–95% reads).
Relieving the primary of that load lets it spend capacity on writes. Two caveats:

- **Replication lag.** A replica runs milliseconds to seconds behind. A user who
  writes then immediately reads may not see their own change. Usual fix: route
  "read your own writes" to the primary, everything else to replicas.
- **Replicas don't reduce write load at all.** Every write still hits the primary,
  and each replica must replay it too. If writes are the bottleneck, adding read
  replicas makes things slightly worse.

### When writes are the bottleneck
The answer is **sharding**, not more replicas: split data by a key (customer ID,
region) so each shard has its own primary handling only its slice of the writes.
That is the real horizontal-scaling story for writes — and the likely follow-up
question.

---

## 4. Keeping a read store up to date

It depends on whether the read store is the **same engine** or a **different one**.

### Case 1 — an RDS read replica
Nothing event-based happens at application level. The engine ships its own
write-ahead log (Postgres WAL, MySQL binlog) to the replica, which replays it.
Your code writes once and knows nothing about it. The schema is identical. This
is replication, not CQRS.

### Case 2 — a genuinely different read model
Writes go to Postgres, reads come from Elasticsearch / Redis / a denormalized
table. The engine can't help, so this is normally event-driven.

**Dual writes** — write to Postgres, then write to Elasticsearch. Simple and
tempting, but there's no shared transaction: if the second write fails the stores
silently diverge and nothing tells you. Interviewers treat this as the wrong
answer.

**Transactional outbox** — write the business row and an "event" row into an
outbox table in the *same* transaction, so either both land or neither does. A
separate relay reads new outbox rows and publishes them.

```
  +-------------------------------------------+
  |        One database transaction           |
  |   [Orders table]      [Outbox table]      |
  +-------------------------------------------+
                      |
                      v
              +-----------------------+
              | Relay                 |
              | reads new outbox rows |
              +-----------------------+
                      |
                      v
              +-----------------------+
              | Event stream          |
              | Kafka, SQS, etc.      |
              +-----------------------+
                      |
                      v
              +-----------------------+
              | Read store            |
              | projection updated    |
              +-----------------------+
```

**Change data capture (CDC)** — a tool like Debezium tails the database's own
replication log and turns each committed row change into an event. Application
code doesn't change and you can't forget to emit an event. Costs: another piece of
infrastructure, and a schema coupled to your table structure.

**Event sourcing** — the events *are* the source of truth; current state isn't
stored, only the sequence of things that happened. Read models are projections
built by replaying events. Full CQRS + ES: powerful, large commitment, rarely the
right default.

**Scheduled batch or materialized views** — a job rebuilds the read table every
few minutes, or Postgres refreshes a materialized view. No streaming
infrastructure. Fine when minutes-old data is acceptable.

### Three things to say about any event-based option
- **Delivery is at-least-once**, so the same event can arrive twice. The
  projection must be idempotent — key on an event ID or version number and ignore
  anything already applied.
- **Ordering matters per entity**, not globally. Partition the topic by entity ID
  so events for one order are processed in sequence.
- **The read model is eventually consistent**, and it can be rebuilt from scratch
  by replaying the stream. That replay ability is a genuine advantage over dual
  writes.

---

## 5. PACELC

Proposed by Daniel Abadi (2012) to capture what CAP leaves out: the
consistency/latency trade-off that exists even when the network is healthy.

- **PAC** — if there is a **P**artition, choose between **A**vailability and
  **C**onsistency.
- **ELC** — **E**lse (normal operation), choose between **L**atency and
  **C**onsistency.

```
                     [Every request]
                      /           \
                     v             v
        +-------------------+   +-------------------+
        | Partition (P)     |   | Else (E)          |
        | network is broken |   | everything healthy|
        +-------------------+   +-------------------+
             /        \              /         \
            v          v            v           v
    [Availability] [Consistency] [Latency] [Consistency]

    In each branch you pick one and give up the other.
```

### Why the "else" branch exists
If every read must return the very latest write, the system has to check with
other nodes before answering — either the write waits for replicas to acknowledge,
or the read contacts a quorum. Either way somebody waits for a network round trip.

That round trip is bounded by physics:
- Within one data centre: under a millisecond.
- Across availability zones: a few milliseconds.
- Mumbai to Frankfurt: roughly 100ms — and no engineering removes it, because
  light doesn't go faster.

So: coordinate and be correct but slow, or skip coordination, answer from the
nearest replica, and accept possible staleness. This choice exists on a perfectly
healthy network — exactly the case CAP says nothing about.

**Concrete version you already know:** an RDS read replica is asynchronous —
writes return without waiting, so writes stay fast and reads may be stale. That is
choosing **EL**. RDS Multi-AZ with synchronous replication makes the write wait
for the standby: an extra round trip, nothing lost. That is choosing **EC**.

### The four combinations
| Class | Systems | Behaviour |
|---|---|---|
| PA/EL | Cassandra, DynamoDB, Riak | Stay up under partition, stay fast normally, tolerate staleness in both |
| PC/EC | HBase, BigTable, VoltDB, etcd | Correctness above all, in both situations |
| PA/EC | MongoDB (classic default config) | Available under partition, coordinates when healthy |
| PC/EL | Yahoo PNUTS | Refuses to serve inconsistently under partition, serves stale data when healthy |

PC/EL shows the two halves are genuinely independent decisions, not one decision
stated twice.

### The one-line interview answer
CAP describes the failure case. PACELC adds that even without failures you are
constantly trading consistency against latency — and most real systems make that
second trade far more often than the first, because partitions are rare and
requests are constant.
