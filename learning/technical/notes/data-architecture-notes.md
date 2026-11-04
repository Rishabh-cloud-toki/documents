# Data Architecture — Consolidated Notes

A single reference covering everything discussed: workload types, storage layouts, choosing a datastore, consistency, scaling, partitioning, sharding, and the datastore families (with deep dives on DynamoDB, MongoDB and Cassandra).

Questions asked during the discussion are marked **Q:** so the reasoning behind each answer stays visible.

---

## Table of contents

1. [OLTP vs OLAP vs Streaming](#1-oltp-vs-olap-vs-streaming)
2. [Row vs Columnar Storage](#2-row-vs-columnar-storage)
3. [Parquet](#3-parquet)
4. [Choosing a Datastore — the Framework](#4-choosing-a-datastore--the-framework)
5. [Access Patterns First, Not Entities](#5-access-patterns-first-not-entities)
6. [Consistency Per Operation](#6-consistency-per-operation)
7. [Stale Reads and Safe Writes](#7-stale-reads-and-safe-writes)
8. [Write-Heavy Patterns](#8-write-heavy-patterns)
9. [LSM Stores](#9-lsm-stores)
10. [Partitioning vs Sharding — the Basics](#10-partitioning-vs-sharding--the-basics)
11. [Partitioning Inside One Database](#11-partitioning-inside-one-database)
12. [Partition Keys and Unique Constraints](#12-partition-keys-and-unique-constraints)
13. [DynamoDB](#13-dynamodb)
14. [Sharding — Deep Dive](#14-sharding--deep-dive)
15. [Replicas vs Shards, and CQRS](#15-replicas-vs-shards-and-cqrs)
16. [The Datastore Families](#16-the-datastore-families)
17. [Why Relational Gives Up Write Scaling](#17-why-relational-gives-up-write-scaling)
18. [⭐ Four Families Compared](#18--four-families-compared)
19. [Document Databases](#19-document-databases)
20. [⭐ Wide-Column and Cassandra](#20--wide-column-and-cassandra)
21. [The Through-Line](#21-the-through-line)
22. [Open Topics](#22-open-topics)

---

## 1. OLTP vs OLAP vs Streaming

**Q: What is the full form of OLTP and OLAP?**

- **OLTP** = Online Transaction Processing
- **OLAP** = Online Analytical Processing
- "Online" is a 1970s–80s leftover meaning *interactive* (answer while you wait), not internet. The opposite of batch processing with punch cards.
- The words that matter are the middle ones: **Transaction** (one small piece of business) vs **Analytical** (looking back across many of those pieces).

**The plain version**

- OLTP is the cash register — one small thing, right now, many registers at once.
- OLAP is the notebook you read at month end — reads everything, takes a minute, tells you something you didn't know.
- Streaming is the friend at the door counting people — remembers only the last few minutes, but tells you *now*.

> **OLTP runs the business, OLAP explains the business, streaming reacts to the business.**

**Q: Explain the table — purpose, access pattern, latency, volume.**

| | OLTP | OLAP | Streaming |
|---|---|---|---|
| Purpose | Run the business — individual records | Analyse the business — aggregate over history | React to events as they arrive |
| Access pattern | Point lookups, small transactions, high concurrency | Large scans, GROUP BY, joins | Continuous queries over windows |
| Latency | ms | seconds–minutes | ms–seconds |
| Volume per query | Rows | Millions–billions of rows | Bounded by window |
| Store | PostgreSQL, MySQL, Spanner, DynamoDB, MongoDB | BigQuery, Snowflake, Redshift, ClickHouse | Kafka Streams, Flink, Materialize, Druid, Pinot |
| Schema | Normalised | Star/snowflake, denormalised, columnar | Event schema |

Only the first row is a real choice. Everything below falls out of it.

- **Purpose** — OLTP *changes* state correctly; correctness beats speed. OLAP *summarises*; nothing changes, so it can relax rules OLTP can't. Streaming *notices* within seconds; its enemy is delay, not volume.
- **Access pattern** — the mental flip: in OLTP/OLAP the data sits still and queries move. In streaming, the query sits still and data moves past it.
- **Latency** — ms for OLTP because a human waits and locks block others. Seconds–minutes for OLAP because reading a billion rows takes that long. Streaming must be fast because a fraud alert 10 minutes late is worthless.
- **Schema** — OLTP normalises so each fact lives in one place (an update can't leave copies disagreeing); cost is joins on read. OLAP denormalises (region copied onto every order row) because there's no join at query time and historical data isn't updated anyway.

**Most real systems run all three.** Postgres takes the orders, a pipeline copies them into BigQuery overnight, Kafka carries the same events to a Flink job watching for anomalies.

---

## 2. Row vs Columnar Storage

**Q: Columnar is separate from denormalisation — what is it?**

A disk is one long line of bytes; a table is two-dimensional. You must pick an order to flatten it, and there are only two choices: along the rows, or down the columns. That's the entire difference.

| region | amount | date |
|---|---|---|
| EU | 50 | Jan |
| US | 20 | Jan |
| EU | 80 | Feb |

- **Row store on disk:** `EU 50 Jan | US 20 Jan | EU 80 Feb`
- **Column store on disk:** `EU US EU | 50 20 80 | Jan Jan Feb`

Same nine values. Only the storage order changes.

**Win 1 — you stop reading what you don't need.** Real table: 80 columns, a billion rows. `SELECT region, SUM(amount) ... GROUP BY region` needs 2 columns of 80. A row store pulls all 80 off disk (disk reads in blocks) and throws away 78 — ~40x more data than needed. A column store reads two files and never touches the rest. Not a clever optimisation; just not reading things.

**Win 2 — compression, bigger than expected.** A billion `region` values in a row look like `EU EU EU US US EU...` — same type, few distinct values, often sorted. Run-length encoding turns a million repeats into "EU × 1,000,000"; dictionary encoding makes each region 1 byte. 10x is common. In a row store the neighbours are a string, a float, a timestamp, a UUID — nothing compresses.

**Win 3 — the CPU gets to be stupid.** A column of integers is a flat array of one type: SIMD adds 8 per instruction, no branching, no pointer chasing. ClickHouse and similar process in vectors of a few thousand values for this reason.

**Why you'd never do this to OLTP.** `SELECT * WHERE order_id = 48213` is one seek in a row store (whole record in one block) and 80 separate reads in a column store. Writes are worse: inserting one order appends to 80 files; updating one field breaks a compressed run. Hence column stores don't really support row-level updates (ClickHouse calls them "mutations" and they rewrite chunks); deletes are often just a flag.

> **Column stores optimise for reading few columns of many rows. Row stores optimise for reading many columns of few rows.**

---

## 3. Parquet

**Q: Is a Parquet file always columnar?**

Yes — not a configuration option, it's the format's reason for existing. Same for ORC. But "all of column A then all of column B" isn't quite how it works.

**Row groups.** Parquet splits rows into row groups first (100k–1M rows, ~128MB–1GB). *Inside* each row group, data is stored column by column.

```
orders.parquet
├── Row group 1 (rows 1–100k)
│     ├── column chunk: region
│     ├── column chunk: amount
│     └── column chunk: date
├── Row group 2 (rows 100k–200k)
│     └── ...
└── Footer — schema, row group offsets, min/max per column chunk
```

Why: a pure column-per-file layout would make reconstructing a row require seeking across the whole file — awful on S3, impossible to parallelise. Row groups are self-contained, so a worker takes row group 7 alone.

**The footer and pruning.** Read first, holds min/max per column chunk. `WHERE date > 'Feb'` skips a row group whose `date` max is `Jan` without reading a byte. A well-sorted file can answer a 1%-of-data query by reading ~1% of the file.

**Row-based formats, for contrast**

- **Avro** — row-based; for moving records around (Kafka, event streams) where you want whole records
- **CSV, JSON, JSONL** — row-based, one record per line
- **Parquet, ORC** — columnar, for analytics

> **Avro for data in motion, Parquet for data at rest.**

**Caveat:** the format is fixed, the *benefit* depends on writing sensibly sized row groups and sorting on the column you filter by. Badly written Parquet (thousands of tiny files with 500 rows each) is a real thing.

---

## 4. Choosing a Datastore — the Framework

Ask in order. The answers usually eliminate all but one or two options.

1. **What are the access patterns?** Write them as concrete queries, with frequencies. Model the data to serve these, not an abstract entity diagram.
2. **What consistency does each operation need?** Linearizable (a balance, inventory, a unique username) vs eventual (a feed, a product page). Per-operation, not per-system.
3. **What is the read:write ratio and absolute volume?** 1000:1 reads → cache + replicas. Write-heavy at scale → LSM store, partitioning from day one.
4. **How structured and how variable is the data?** Fixed relational, documents, graph, time series, full-text.
5. **What are the query needs?** Ad-hoc joins/aggregation → relational. Known key access → KV/wide-column. Traversal → graph. Search/ranking → search engine.
6. **Scale ceiling and growth?** Will one primary + replicas carry you 3 years, or do you need horizontal write scaling now?
7. **Operational reality:** team skills, managed service, backup/DR, cost model.

**Why the order works**

- Q1–2 = **correctness**. Get them wrong and the system is broken.
- Q3–6 = **scale**. Get them wrong and it's slow or expensive.
- Q7 = **you**. Last, but it vetoes the others.

Most people start at Q5 or Q6 ("we need something that scales") and work backwards. That's how teams end up with Cassandra holding 40GB plus a team that can't debug it at 2am.

**Notes**

- **Q3** — two numbers, often conflated. *Ratio* tells you the shape of the fix (read-heavy is easy — copies; write-heavy is hard). *Absolute volume* tells you whether you have a problem: 1000:1 at 100 req/sec is a laptop; at 500,000 req/sec it's an architecture.
- **Q3, LSM** — B-trees (Postgres, MySQL) update in place: random disk writes. LSM (Cassandra, RocksDB) appends to a log and sorts later. Appends are sequential and faster; cost is reads may check several places and compaction burns CPU/disk.
- **Q4 warning** — "our schema changes a lot" is usually not a reason for a document store. It's often a reason to get better at migrations. The schema just moves into application code where nothing enforces it.
- **Q4 vs Q5** — Q4 is the data's shape, Q5 is what you do with it. When they disagree, **Q5 wins**: you can store relational data in a KV store; you can't make a KV store do ad-hoc joins.
- **Q5, graph test** — *multi-hop*. One or two hops is a join. "Accounts within 4 degrees" is Neo4j territory.
- **Q5, search test** — *ranking*. Relevance + typos + stemming = Elasticsearch. "Rows containing this word" = Postgres full-text.
- **Q6** — people wildly underestimate one tuned Postgres primary: tens of thousands of TPS, terabytes, one machine.
- **Q7** — can your team debug this at 3am? Is there a managed version? **Have you tested a restore?** An untested backup is not a backup. Per-request pricing suits spiky traffic and is brutal for steady volume; provisioned is the opposite.

**The default: start with a well-run PostgreSQL.** It does OLTP, JSONB documents, full-text, PostGIS, and moderate analytics. One system, one backup story, one set of skills. Every store you add multiplies operational surface — plus the hardest problem, consistency *across* stores, which no transaction covers.

The key word is **measured**: add Elasticsearch when Postgres full-text is measurably too slow, not when you imagine it will be.

**Worked example — order management, mid-size retailer**

Patterns: get order by id, list customer's orders by date, update status, revenue by region by month, free-text address search. Consistency: order creation linearizable; reporting a day stale; search a minute stale. Volume: ~200 orders/min, 50:1 read:write. Shape: relational. Ceiling: one primary for years. Ops: small team, RDS.

→ **Postgres**, plus a read replica for reporting, Postgres full-text for search. Nothing else until something measured says otherwise.

Change one input — a marketplace at 50,000 orders/min with sub-second fraud checks — and Q2, Q3, Q6 all change: Postgres for orders, Kafka + Flink for fraud, a warehouse for reporting.

---

## 5. Access Patterns First, Not Entities

**Q: Is the intent here to decide columnar vs row-based?**

No — that's usually settled by purpose (running the business = row, analysing it = columnar). Question 1 decides something harder: **what shape the data physically takes, and how it's keyed.**

| Pattern | What it decides |
|---|---|
| "Get order by id" | `order_id` is the access key. Trivial in Postgres; in DynamoDB this is the partition key and it's permanent. |
| "List orders for a customer, sorted by date" | `customer_id` and `date` must be co-located in storage order. Postgres: composite index `(customer_id, date)` — column order matters, and you only know it because you wrote the query down. DynamoDB: partition key `customer_id`, sort key `date`. |
| "Sum revenue by region by month" | `region` and `month` must be available without joining across a billion rows → denormalise region onto the order row. Also: this shouldn't run on the primary. |

None of those answers was "columnar" or "row". They were: which key, which sort order, which index, what gets copied where.

**Where it touches storage layout — indirectly.** The third pattern is many rows, few columns, aggregate — the OLAP signature. But you didn't learn "columnar" from the pattern; you learned this pattern belongs in a **different system**. That's the real output: some patterns don't belong in your main database.

**Why "not entities" is sharp.** Entity-first gives a clean normalised model that answers every query *adequately* and none *well*. In Postgres, access-pattern-first mostly changes indexes and a few denormalisations. In a KV or wide-column store it changes *everything* — the table structure is a transcription of your query list, and storing the same data three times under three keys is normal.

**The rule:** for each query ask *what would have to be true on disk for this to be one cheap lookup?* Conflicts between two queries reveal either an index, a denormalisation, or a second store you need.

---

## 6. Consistency Per Operation

**Q: How do choices change — linearizable vs not?**

**Linearizable** = every read sees the most recent completed write; the system behaves as if there is exactly one copy. **Cost:** all operations on that data funnel through one authoritative point (a primary, or a quorum). Funnelling costs latency, costs availability, caps throughput.

| Operation | Requirement | Where it runs |
|---|---|---|
| Decrement inventory | Linearizable | Primary, in a transaction. No cache, no replica. |
| Show product page | Seconds stale is fine | Cache or read replica. TTL = your staleness budget. |
| Recommendation feed | Minutes stale is fine | Precomputed KV store, rebuilt by a batch job. |

**Inventory — the classic bug**

```java
int qty = repo.getQuantity(productId);    // reads 1
if (qty > 0) {
    repo.setQuantity(productId, qty - 1); // writes 0
}
```

Two threads both read 1, both write 0. Two units sold, one in stock. The gap between read and write is where the money leaks.

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = ? AND quantity > 0;
```

Check the affected row count; zero means sold out. The database holds a row lock, so the second transaction sees `quantity = 0`.

What this rules out: no reading inventory from a replica during checkout (lag is *usually* ms — that's the assumption that breaks under load); no caching the count *for the decision* (caching it for display is fine); no eventually-consistent store for this row (DynamoDB conditional writes can do it; Cassandra needs lightweight transactions and people regret it).

**Single-key vs multi-key.** Inventory is single-key, so linearizability is cheap everywhere. *Multi-key* ("decrement inventory AND charge the card AND create the order, atomically") is what's expensive. Most real checkouts use a **reservation**: hold stock for 10 minutes (single-key write), take payment, confirm; if payment fails the hold expires. One hard distributed transaction traded for two easy local ones plus a timeout.

**Product page — two traps**

- **Read-your-own-writes.** A seller edits, the page reloads from a lagging replica, they see the old text and edit again → support ticket. Fix: **sticky read** — for a few seconds after a user writes, route *that user's* reads to the primary.
- **Stale prices are a business decision.** Cached page shows €40, real price €45 — most retailers honour the displayed price. Someone has to decide; it shouldn't be a surprise.

**The pattern:** ask per operation — **if this value is 200ms out of date, what breaks?** Money or a count → linearizable. A slightly old page → eventual. Nobody can tell → precompute.

The real-world failure is almost never "wrong database". It's one operation quietly routed down the wrong path because someone added a replica for performance.

---

## 7. Stale Reads and Safe Writes

**Q: Two users read a stale value from a replica, then do a linearizable operation. How do I make it correct?**

> **You don't fix this by making the read fresh. You fix it by making the write refuse to commit against state that changed.**

The stale read isn't the bug — even a read from the primary is stale by the time your code acts on it. The bug is *trusting* the value at write time.

**Case A — the write can be expressed relative to current state** (inventory, balances, counters)

```sql
UPDATE inventory SET quantity = quantity - 1
WHERE product_id = ? AND quantity > 0;
```

The replica said "3 left" — doesn't matter, the write never uses the number 3. Both users can read stale values from different replicas and the outcome is still correct. Cheapest and most robust; whenever you can phrase a write as a relative change with a guard, do that.

**Case B — the write depends on what the user saw** (editing a product, approving at a displayed price)

```sql
UPDATE product SET price = 45, version = version + 1
WHERE id = ? AND version = 7;
```

```java
@Entity
public class Product {
    @Id private Long id;
    @Version private Long version;
    private BigDecimal price;
}
```

A stale version throws `ObjectOptimisticLockingFailureException` on flush. This is **optimistic concurrency control** — same idea as HTTP `ETag` + `If-Match`.

**When the check fails** (the part people skip)

- Case A: don't retry — it's genuinely sold out. Reject and tell the user.
- Case B: reload. A background counter job can retry with backoff; a human editing a form should be **shown** the conflict — silently retrying discards somebody's edit.
- Make it **idempotent**: a client-generated idempotency key means a retry after timeout doesn't decrement twice.

**When you genuinely need a fresh read** — route *that operation* to the primary (`@Transactional` without `readOnly = true`). It's a per-operation routing decision, not global. Two intermediate options: **sticky reads**, or **bounded staleness by LSN** (`pg_current_wal_lsn()` / `pg_last_wal_replay_lsn()`; Aurora and proxies can do it) — real, but adds latency and complexity, so only when measured.

**Pessimistic locking** (`SELECT ... FOR UPDATE`) is correct but holds a lock across your application logic — serialising every buyer of a hot product, and deadlocking on inconsistent lock order. Use when conflicts are frequent and retrying is expensive.

> **Never let application code do check-then-act across a network boundary.** Either express the condition inside the write, or hold a lock for the whole read-decide-write span. A stale read followed by an unconditional write is still a bug even if you read from the primary.

---

## 8. Write-Heavy Patterns

**Q: Read-heavy gets replicas, caches and CQRS. What about write-heavy?**

CQRS helps the write side too: once reads come from projections, the write model stops carrying indexes that existed only for queries. A table with 2 indexes is far faster than one with 9.

A ladder — work down it in order, each step costs more than the last.

**1. Don't write it.** "User last seen at" on every request → one update per minute, kept in memory. Metrics sampled or pre-aggregated at the client. View counts buffered and flushed as `+247`. Unglamorous; routinely removes 90% of write volume.

**2. Batch them.** One statement writing 1000 rows beats 1000 statements. Hibernate `jdbc.batch_size` = 50–100; Postgres `COPY` is ~an order of magnitude faster than `INSERT`.

**3. Cut the cost of each write.**
- Indexes are a write tax — every index is another B-tree to update. Audit them.
- Synchronous replication doubles write latency.
- Group commit / relaxed fsync (`commit_delay`, `synchronous_commit = off`) trades a small loss window for throughput. Fine for logs, not for money.

**4. Buffer in front (write-behind).** App writes to Kafka, returns; a consumer drains at a steady rate. This buys **spike absorption** (5k/sec sustained capacity vs 40k/sec for 30 seconds during a flash sale). It does **not** increase long-run capacity — if the average exceeds the drain rate, the queue grows until it falls over. Cost: the write is async ("order received", not "confirmed"), needs idempotency keys and a consumer-failure plan.

**5. Append instead of update.** Update-in-place means find the page, lock, modify, write WAL, and in Postgres a new row version autovacuum must clean. Appending never contends. **Event sourcing** is this taken to its conclusion — append `Deposited(100)`, `Withdrew(30)`, derive the balance. Reads become a projection, which is exactly CQRS. Also what LSM stores do internally.

**6. Spread the writes: partitioning and sharding.** The entire game is the **partition key**. Classic mistake: partitioning event data by timestamp — all of today's writes land on one partition. Fix: a key with natural spread (`customer_id`, `device_id`, hash) with time as the *sort* key.

**7. Single-row contention.** 10,000 writes/sec all targeting *one row* (a viral post's like counter). Sharding doesn't help — it's one key.
- **Sharded counters** — 20 rows, each write picks one at random, reads sum all 20.
- **Aggregate in a stream** — Flink counts in memory over a window, flushes periodically.

Steps 1–3 don't change your architecture (tuning — spend your first effort here). Step 4 changes consistency guarantees. Steps 5–6 change the data model and are hard to retrofit. **Honest check:** a tuned Postgres does tens of thousands of writes/sec. Confirm you're near that before reaching past step 3.

---

## 9. LSM Stores

**Q: What are LSM stores?**

**LSM = Log-Structured Merge tree.** From one hardware fact: random disk writes are slow, sequential appends are fast.

- **B-tree** (Postgres, InnoDB) updates in place — random I/O, plus WAL, plus every index.
- **LSM** never modifies anything on disk. **It only appends.**

**Write path:** append to a commit log (crash recovery) → insert into the **memtable** (sorted structure in RAM, skip list or balanced tree) → return. When the memtable fills (~64MB), flush it sequentially as an **SSTable** (Sorted String Table: sorted keys + sparse index). Once written, that file is **immutable**.

```
In memory:   Writes → Memtable (sorted, not yet on disk)
                        ↓ flush when full
On disk:     Level 0:  [SSTable][SSTable][SSTable]   newest, may overlap
                        ↓ compaction
             Level 1:  [ SSTable ][ SSTable ]        merged, no overlap
                        ↓
             Level 2:  [      SSTable       ]        largest, oldest
```

**Updates and deletes.** An update writes the new value to the memtable; the key now exists twice and **the newest copy wins**. A delete writes a **tombstone** — "this key is deleted" — and the old value sits on disk until compaction.

**Read path** — check memtable, then SSTables newest to oldest, stopping at the first hit. A missing key means checking everything. Two rescues:
- **Bloom filters** — "is this key definitely *not* here?" A few bits per key eliminates almost all pointless reads.
- **Sparse indexes** — keys are sorted, so an in-memory index of every Nth key tells you which block to read.

**Compaction** merges SSTables: merge-sort (cheap, already sorted), keep the newest version of each key, drop tombstoned rows, write one file, delete the old ones. **Leveled** (RocksDB, Cassandra LCS): each level ~10x bigger, no overlap within a level — better reads, more compaction. **Size-tiered** (Cassandra default): merge files of similar size — better writes, worse reads, can temporarily need double disk.

**The three amplifications**

- **Write amplification** — one logical write rewritten several times by compaction (10–30x normal). You still win, because it's all sequential.
- **Read amplification** — one logical read may touch several files.
- **Space amplification** — old versions and tombstones occupy disk until compaction.

B-trees have low read and space amplification, high random-write cost. LSM inverts it.

**Operational gotchas:** compaction competes with live traffic — latency spikes invisible in averages, obvious at p99; tuning is ongoing. And the classic Cassandra failure: a queue-like table with constant deletes, where reads scan piles of tombstones.

**Who uses what:** LSM — Cassandra, ScyllaDB, RocksDB, LevelDB, HBase, ClickHouse MergeTree, MongoDB WiredTiger. B-tree — PostgreSQL, MySQL/InnoDB, Oracle, SQL Server.

---

## 10. Partitioning vs Sharding — the Basics

**The toy-box version.** One toy box, everything in it, eventually the lid won't shut. So you get more boxes and make a **rule** — dinosaurs in box 1, cars in box 2. Finding a dinosaur means opening one box. **That rule is the only thing that matters:** pick a bad one ("toys from before my birthday" vs "after") and box 2 grows forever while box 1 sits still.

- **Partitioning** = several boxes *in your own room*. One machine.
- **Sharding** = the boxes are in different houses. Wanting dinosaurs *and* cars means visiting two.

Sharding crosses a **machine boundary**. Within one machine the database can still join, keep transactions atomic, and count. Across machines it can't do any of that for free.

> **Partition early, shard late.** Partitioning is a tuning decision. Sharding changes what your application is allowed to do.

The rule for deciding which box is the **partition key** (or shard key). Almost every real problem here — hot partitions, expensive queries, migrations you can't undo — traces back to that one choice.

---

## 11. Partitioning Inside One Database

**Two things share the name**

- **Vertical** — splitting *columns* into separate tables (move `description TEXT` and `image_blob` out of `products`). Narrower rows → more rows per 8KB page → scans read fewer pages. Done by hand; not what people usually mean.
- **Horizontal** — splitting *rows* across physical tables. One logical table, many physical ones. This is what everyone means.

**Three strategies:** **range** (usually a date — most common), **list** (discrete values), **hash** (even spread, no natural range).

```sql
CREATE TABLE orders (
    order_id    bigserial,
    customer_id bigint NOT NULL,
    order_date  date   NOT NULL,
    region      text,
    amount      numeric(12,2),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

The `DEFAULT` partition is a useful safety net and a dangerous habit — rows silently pile up, and you can't add a partition covering a range that already has rows in default without moving them. Some teams omit it so a missing partition fails loudly.

**Partition pruning**

```sql
EXPLAIN SELECT sum(amount) FROM orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01';
-- Aggregate
--   ->  Seq Scan on orders_2026_02 orders
```

One partition in the plan. But `SELECT * FROM orders WHERE customer_id = 4711` scans **every** partition — one index lookup became 24. Partitioning made that query worse.

Two refinements: pruning also happens at **execution time** (PG 11+), so prepared-statement parameters and join-derived values still prune. And pruning needs the **raw column** — `WHERE date_trunc('month', order_date) = ...` won't prune.

**The benefit people actually partition for: data lifecycle**

```sql
-- Normally: hours, massive WAL, bloat, vacuum pain
DELETE FROM orders WHERE order_date < '2025-01-01';

-- With partitioning: near-instant metadata work
ALTER TABLE orders DETACH PARTITION orders_2024_12;
DROP TABLE orders_2024_12;
```

Use `DETACH PARTITION ... CONCURRENTLY` in production. `ATTACH PARTITION` scans to verify — add a matching `CHECK` constraint *before* attaching and Postgres skips the scan.

**Other benefits:** smaller indexes (24 shallower trees, current month may fit in RAM); autovacuum parallelism (old partitions stop being vacuumed at all — on a huge single table vacuum becomes chronic); maintenance granularity (`REINDEX`, `ANALYZE`, backups per partition); parallel scans.

**Costs:** planning overhead (a few hundred partitions fine, a few thousand hurts — prefer monthly over daily); no global unique index; `UPDATE` changing the partition key is delete+insert; `CREATE INDEX CONCURRENTLY` can't run on the parent directly; FKs pointing *at* a partitioned table need the full key.

**Automation:** `pg_partman` pre-creates and drops partitions. Without it you eventually forget and find out at 2am.

**Sub-partitioning** (range on date, then hash on customer within each month) helps when one month is still too large — but 24 months × 8 hashes = 192 tables. Only when measured.

**Other databases:** **Oracle** has **global indexes** spanning all partitions (so `order_id` *can* be globally unique) — cost: dropping a partition invalidates them. Also interval and reference partitioning. **MySQL** requires the partition column in every unique key and has **no FK support at all** on partitioned tables. **SQL Server** uses partition functions/schemes with switching.

**When not to partition:** below ~50–100GB a well-indexed single table usually wins. Honest triggers: a retention policy, unmanageable vacuum/index maintenance, or a measured pruning pattern. And check your access patterns contain the partition key first.

---

## 12. Partition Keys and Unique Constraints

**Q: Do we need the partition key in the primary key so Postgres can check all partitions for uniqueness?**

**It's the reverse.** Including the partition key doesn't let Postgres check across partitions — it lets Postgres **avoid** having to.

Postgres has only **local indexes**. The only way to enforce uniqueness is to guarantee that two colliding rows are *physically in the same partition*. Requiring the partition key in the unique key guarantees that: same `order_date` → same partition → checking one local index suffices. **A routing guarantee, not a global check.**

**Q: But what if order_id is the same and order_date differs? To me that's a duplicate.**

Exactly — and Postgres will not stop you. You declared `UNIQUE (order_id, order_date)`, not `UNIQUE (order_id)`:

```sql
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-01-15');  -- ok
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-02-20');  -- also ok!
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-01-20');  -- ok too
```

The third is in the *same* partition as the first, but the pair differs so the composite index sees no conflict. **Nothing is checking `order_id` at all.**

**Why:** a global unique index would be consulted on every insert into every partition — a shared contention point — and would break cheap `DETACH PARTITION`. Oracle accepts that cost; Postgres declines.

**Four ways to get a unique `order_id`**

1. **Generate IDs that can't collide** — `bigserial` / `GENERATED ALWAYS AS IDENTITY` (sequence on the parent, shared by all partitions), or UUIDv7 / Snowflake (UUIDv7 is time-ordered, so unlike v4 it doesn't destroy index locality). Covers ~95%. Gap: it's a *convention*, not a constraint — bulk loads with explicit IDs, restores, or migration scripts sail through.
2. **Partition by hash of `order_id`** — now `PRIMARY KEY (order_id)` is legal and truly enforced. Cost: you lose date-range pruning and cheap `DROP PARTITION` for retention.
3. **Partition by range of `order_id`** — works if IDs are time-ordered, so old partitions still hold old data and remain droppable. You lose direct date pruning (a lookup table of ID-range-per-month recovers most of it). Good middle path.
4. **Registry table** — `CREATE TABLE order_id_registry (order_id bigint PRIMARY KEY)`, written in the same transaction. Real enforcement, but every insert touches a second ever-growing unpartitioned table that can't be aged out. Reserve for correctness disasters (financial, regulatory).

**How to decide:** what actually produces a duplicate? IDs only from the sequence → option 1. External systems supply IDs → option 1 gives nothing; use 2, 3 or 4.

> **You can have an enforced global unique key, or date-range partitioning, but Postgres won't give you both on the same column.**

**FK consequence:** children must reference the full key, so `order_date` gets duplicated into every child table and every join carries two columns.

```sql
CREATE TABLE order_lines (
    order_id   bigint NOT NULL,
    order_date date   NOT NULL,   -- carried purely to satisfy the FK
    FOREIGN KEY (order_id, order_date) REFERENCES orders (order_id, order_date)
);
```

---

## 13. DynamoDB

**Q: Is there anything special about partitioning in DynamoDB?**

In Postgres, partitioning is an optimisation you add later. In DynamoDB it **is the data model**, it's mandatory, and it's permanent.

Every item has a **partition key** (hash key), hashed to decide which physical partition holds it. No range or list partitioning, no choice of strategy. An optional **sort key** orders items *within* a partition. Together they form the primary key and must be unique.

**Q: By "item" do you mean the thing being saved? Who decides which element is the partition key?**

**Item = row.** (table → table, item → row, attribute → column.)

**You decide the partition key once, at table creation** — not per-write:

```
--key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE
```

After that, **every item must supply a value for `PK`**. Two different decisions: the *designer* picks which attribute is the key (once, permanent); the *writer* supplies its value (per item, every write — same as any column).

```json
{ "PK": "CUSTOMER#42", "SK": "PROFILE",               "name": "Anna",  "email": "a@x.com" }
{ "PK": "CUSTOMER#42", "SK": "ORDER#2026-01-15#1001", "amount": 50,   "status": "SHIPPED" }
{ "PK": "CUSTOMER#43", "SK": "PROFILE",               "name": "Ben",   "email": "b@x.com" }
```

Items have **different attributes** — schemaless apart from the key. The two `CUSTOMER#42` items land in the **same partition**, always. So `Query: PK = "CUSTOMER#42"` returns profile and orders sorted by SK in one read. But "all customers" is a `Scan`, and "customers in the EU" is a Scan + filter or a GSI.

*Why `PK` and not `customerId`?* In single-table design the same attribute holds `CUSTOMER#42` and `PRODUCT#77`. If your table holds one entity type, name it `customerId`. **The application builds the key value** — `"CUSTOMER#" + id` is string concatenation; `#` is convention (sorts low, rare in real data).

**The thing with no Postgres equivalent.** In Postgres a query without the partition key is *slower*. In DynamoDB it **is not a query** — it's a `Scan` that reads the whole table and bills you for it. No "add an index later": a GSI is a whole second copy with its own partition key, paid for forever.

**Physical limits:** 10GB per item collection (with an LSI); **3,000 RCU / 1,000 WCU per partition**. Your table may be provisioned for 100,000 WCU, but if every write targets one key you're capped at 1,000. That's a **hot partition** — `status = 'PENDING'` or `date = '2026-09-16'` as partition keys are the classic mistakes. **Adaptive capacity** reallocates throughput and can split hot partitions, but a single hot *key* can't be split. Workaround: **write sharding** — `PENDING#0` … `PENDING#9`, reads query all ten and merge.

**Single-table design**

| PK | SK | attributes |
|---|---|---|
| `CUSTOMER#42` | `PROFILE` | name, email |
| `CUSTOMER#42` | `ORDER#2026-01-15#1001` | amount, status |
| `CUSTOMER#42` | `ORDER#2026-02-20#1002` | amount, status |

"Customer 42 and their recent orders" is one query on one partition, because those items are physically adjacent. The sort key prefix + `begins_with` filters to orders; the date in the sort key gives ordering and ranges free. **The reason: DynamoDB has no joins, so you co-locate and let the sort key do a join's work.**

**Indexes:** **LSI** — same partition key, different sort key; must be created with the table, never added. **GSI** — different partition key, addable anytime, a separate eventually-consistent replicated copy with its own throughput. Five GSIs = six writes per write.

**Q: When would I use DynamoDB vs Postgres? I thought it was when I don't need atomicity and easy reads.**

Two corrections:

- **DynamoDB does have atomicity.** Single-item writes are atomic and strongly consistent. Conditional writes (`attribute_not_exists`, `quantity > 0`) are linearizable on that item — exactly the inventory pattern. `TransactWriteItems` gives ACID across up to 100 items. Single-key atomic ops are a *strength*.
- **Reads aren't "easier" — they're narrower.** Matching reads are faster and more predictable than Postgres; non-matching reads are effectively impossible.

**The actual question:** *Do I know all my access patterns in advance, and will they stay stable?*

> DynamoDB: "Tell me exactly which questions you'll ask. Those are instant forever, at any size. But I'll only answer *those* questions."
> Postgres: "Ask me anything, whenever. Some answers will be slower, and past a certain size I'll struggle."

**Concrete:** order system on DynamoDB, PK `CUSTOMER#42`. Six months later — *"How many orders shipped last Tuesday?"* Postgres: write a query, maybe an index, an afternoon. DynamoDB: scan the whole table, or build a GSI keyed on ship date (a second copy, an extra write on every order, permanently).

**The sting:** at 200 orders a minute you never needed the scale — so you accepted the rigidity and received nothing for it. The cost is paid up front and feels free; the benefit only materialises at a scale most systems never reach.

**Choose DynamoDB when:** access is by a known key; you need consistent single-digit-ms latency at any scale; write volume is large and spreads across many keys; you want zero operational work; traffic is spiky; the data has no relationships worth traversing. *(Sessions, carts, preferences, IoT state, feature flags, idempotency keys, leaderboards.)*

**Choose Postgres when:** you'll ask questions you haven't thought of yet (covers most business systems); data has relationships; you need multi-row constraints (DynamoDB has **none** — every invariant lives in app code); aggregation matters; your volume fits on one machine.

**On "schemaless/document":** half right. It's **wide-column / key-value**, not a document DB — you can store nested JSON but can't index into it. Postgres `JSONB` gives flexible documents *with* indexes and SQL.

**The tell:** if a product manager will eventually ask "can we see this by region?", you want Postgres.

*Pricing note: on-demand removed most capacity-planning pain, and AWS cut on-demand prices substantially in late 2024. Check current pricing.*

---

## 14. Sharding — Deep Dive

Partitioning splits data across tables on one machine. Sharding splits it across **separate servers that don't know about each other**. Then: no joins, no transactions, no unique constraints, no `ORDER BY` across shards. **You're not buying a feature — you're giving up features to get write scaling.**

**What actually forces it:** write throughput beyond one primary; dataset size beyond one machine's storage or backup window; **regulatory data residency** (legitimate at modest volume); **blast radius**. "Our database is slow" isn't on the list — that's usually indexes or N+1 queries.

**Strategies**

- **Range** — natural range queries, rebalance by moving boundaries. Problem: skew, and any monotonically increasing key sends every new write to the last shard.
- **Hash** — excellent spread; fatal flaw is `% N` (change N and almost every key moves).
- **Directory / lookup** — a table says which shard holds which key. Total flexibility (move one noisy tenant to its own shard); cost is a lookup per request and a component that must never go down. Common in multi-tenant SaaS.

**Virtual buckets** fix the `% N` problem: map keys to a large fixed number of buckets (1024, 4096 — chosen once), then buckets to shards. `hash(key) % 1024` never changes; only the bucket→shard assignment does.

```
Before (3 shards):  A A A A  B B B B  C C C C
After adding D:     A A A D  B B B D  C C C D
```

Only 3 of 12 buckets move. Buckets are the unit of movement — you move whole buckets, never rows, and a key never changes bucket.

**Consistent hashing** (the ring — Cassandra, DynamoDB internally) gets the same property differently: nodes and keys on a circle, a key belongs to the next node clockwise, so adding a node steals only from its neighbour. Virtual nodes exist because a plain ring distributes unevenly with few nodes.

**Routing:** **client-side** (fastest; every service needs the logic and a redeploy to change the map), **proxy** (Vitess, ProxySQL — your app sees one database; costs a hop), **database-native** (Citus, `mongos`, CockroachDB — least work, most lock-in).

**What you lose**

- **Cross-shard joins.** Mitigations: **co-location** (shard both tables by `customer_id` — why the shard key is almost always a tenant/customer ID) and **reference tables** (replicate small lookups to every shard).
- **Cross-shard transactions.** Gone, unless 2PC (slow, underestimated failure modes). In practice: keep transactions within a shard, or use a **saga** (local transactions + compensating actions).
- **Global unique constraints.** Need a registry table, or non-colliding IDs — UUIDv7, Snowflake, or per-shard sequence offsets (shard 1: 1, 9, 17…; shard 2: 2, 10, 18…).
- **Auto-increment IDs.** Each shard starts at 1. Decide *before* sharding.
- **Aggregations and pagination.** `SUM` is a scatter-gather, as slow as the slowest shard. `ORDER BY date LIMIT 20` means fetching 20 from every shard, merging, discarding. Deep offset pagination becomes impractical — use cursors.
- **Schema migrations.** Must run on every shard; shards drift if one fails. Tooling is mandatory.

**Resharding a live system:** stand up the new shard → **backfill** the moving buckets → **double-write** → **verify** (reconciliation job comparing checksums) → cut reads over bucket by bucket with a flip-back → stop double-writing, drop old data. People underestimate the verify step; without it you find out about drift from a customer, months later.

**Real systems:** **Vitess** (MySQL; YouTube, Slack, Square — most battle-tested), **Citus** (Postgres; pick a distribution column, handles co-location and reference tables), **MongoDB** (native, `mongos`, chunk balancing), **CockroachDB / Spanner / YugabyteDB** (sharded from the ground up, *with* distributed transactions, at higher write latency), **Cassandra / DynamoDB** (sharding isn't a feature, it's the architecture).

**Exhaust these first** (cheapest first — one usually suffices): fix queries and indexes → vertical scaling (a 128-core, 2TB machine is cheaper than an engineering team) → read replicas → caching → move analytics to a warehouse → partition within one database → **functional split** (move billing or notifications to its own database — much easier, often enough) → archive cold data.

**Picking the shard key:** high cardinality, even distribution, present in nearly every query, and **stable** (a changing key means moving the row between machines). For most business systems: `customer_id` or `tenant_id`, because it co-locates everything one customer touches. Failure case: a tenant too large for one shard → dedicated shard (directory approach) or sub-shard.

> **Sharding converts a database problem into an application problem.** You get write scaling and blast-radius isolation; you pay in joins, transactions, constraints, migrations and operational complexity — permanently.

---

## 15. Replicas vs Shards, and CQRS

**Q: How does sharding help write throughput? Why don't replicas?**

> **Replicas copy writes. Shards divide writes.**

A replica is a full copy, so every write to the primary is also applied on every replica. Primary does 10,000/sec; replica 1 does 10,000; replica 2 does 10,000. **Adding replicas doesn't reduce anyone's write load by a single row.** It adds *read* capacity — reads can be split across copies because reading changes nothing — but the write ceiling is unchanged. It actually gets slightly worse (WAL shipping, and synchronous replication waits for acks).

With 4 shards, each holds a *different* quarter: ~2,500 writes/sec each. Adding a fifth raises the ceiling again. That's **horizontal write scaling** — capacity grows with machine count.

> **Reads can be duplicated. Writes must be divided.**

**They compose:**

```
Shard A: primary + 2 replicas   (2,500 writes/sec, reads spread over 3)
Shard B: primary + 2 replicas
Shard C: primary + 2 replicas
Shard D: primary + 2 replicas
```

12 machines: 4x write capacity, 12x read capacity. This is why the framework asks for the **ratio** separately from volume — 1000:1 means replicas, not shards, even at high total traffic. Write-heavy at 1:1 is what pushes you to shard.

**Q: So for write-heavy — sharding for writes, plus replicas / CQRS views for reads?**

Almost. There are **two different read problems**, and conflating them is where designs go wrong.

| Problem | Solved by | Why |
|---|---|---|
| Read volume *within* a shard | Replicas | Shard B's replicas hold shard B's data — any read including the shard key |
| Reads that don't fit the shard key | CQRS projection | Replicas do **nothing** — a replica of shard B still only knows shard B |

Sharded by `customer_id`, "all orders shipped last Tuesday" has every shard holding part of the answer. Replicas just give more copies of one quarter of the data. *That* is why CQRS belongs here.

```
Writes ──routed by key──> Shard A (primary + replicas) ──┐
                          Shard B (primary + replicas) ──┼──CDC──> Read projection
                          Shard C (primary + replicas) ──┘         (cross-shard reads)
```

**How the projection gets built** — not by writing twice from the app. Tap each shard's replication log (Postgres logical decoding via Debezium, MySQL binlog, DynamoDB Streams), publish to Kafka, and a consumer merges all shards into whatever store answers the query well: **Elasticsearch** (search/filtering), **a columnar warehouse** (aggregation), **a denormalised Postgres table** (keyed for that query), **Redis** (leaderboards, counters).

**The projection is keyed for the read, not the write.** That's the point.

**Why CDC, not dual writes:** dual writes have no atomicity — the DB commit succeeds, the Kafka publish fails, and the two are permanently out of sync with nothing to detect it. CDC reads the *committed* log, so it can't disagree with what was committed, and replays from an offset after failure.

**Consistency caveat** — the projection is eventually consistent (seconds, sometimes minutes). So:

- **Linearizable decisions** → shard primary, in a transaction. Never the projection, never a replica.
- **Keyed reads tolerating staleness** → shard replicas.
- **Cross-shard, analytical, search, reporting** → projection.

Common bug: pointing a checkout-path read at the projection because it's convenient and fast. *It's fast because it's stale.*

**One correction:** "read replicas **or** read views" — it's both, not alternatives. Replicas are copies of the write model with the same keying; projections are a different model with different keying. Replicas for volume, projections for shape.

**The caveat:** CQRS is not a default. It costs another pipeline, another store to back up, lag to monitor, and the hardest part — **reconciliation**. When the projection drifts (consumer bug, poison message, schema change) you need to detect it and rebuild. **Build the rebuild path early.**

---

## 16. The Datastore Families

**The organising idea:** every family makes one access pattern cheap and everything else expensive. Read each as *what did it decide to be good at, and what did it give up?* And these are **families, not products** — Postgres does documents, full-text, time series and geospatial.

| Family | Good at | Gave up | Examples |
|---|---|---|---|
| Relational | Flexible queries, enforced correctness, mature tooling | Horizontal write scaling, schema churn at scale | PostgreSQL, MySQL, SQL Server |
| Distributed SQL | Scale **and** ACID, global tables | Latency (consensus per write), cost | Spanner, CockroachDB, YugabyteDB, TiDB |
| Key-value | Fastest lookup, trivial scaling, caching | Any question that isn't "by key" | Redis, DynamoDB, Riak, Aerospike |
| Document | Flexible schema, one-aggregate reads | Joins, cross-document transactions | MongoDB, Couchbase, Firestore |
| Wide-column | Massive write throughput, predictable latency, time series | Ad-hoc queries, joins; table per query | Cassandra, ScyllaDB, HBase, Bigtable |
| Graph | Multi-hop traversal, recommendations, fraud rings | Bulk analytics, high write volume | Neo4j, Neptune, JanusGraph |
| Time series | Compression, downsampling, retention, range queries | Non-time-based access | InfluxDB, TimescaleDB, Prometheus |
| Search | Full-text, faceting, fuzzy, relevance | Being a source of truth | Elasticsearch/OpenSearch, Solr |
| Analytical (columnar) | Scans and aggregation over billions of rows | Point writes, single-row updates | BigQuery, Snowflake, Redshift, ClickHouse |

**Notes**

- **Distributed SQL** — consensus (Raft/Paxos) per shard; every write needs a quorum. Spanner's extra trick is **TrueTime** (atomic clocks + GPS giving a bounded uncertainty window, enabling global ordering); others approximate with hybrid logical clocks.
- **Key-value** — the store doesn't know what's in the value, which is *why* it's fast. Redis footnote: it has real data structures (sorted sets, lists, streams), which is why it does leaderboards, rate limiting and queues.
- **Document** — trap: teams pick it for "flexible schema", then find the schema still exists — it moved into application code where nothing enforces it.
- **Graph** — the test is **multi-hop**. If "hops" or "degrees" isn't in your use case, you don't need one.
- **Time series** — real specialisations: compression (delta-of-delta timestamps, XOR consecutive floats — 10x+), downsampling (1s for a day, 1min for a month, 1h for a year), retention by dropping partitions. **TimescaleDB is a Postgres extension** — usually the right first step.
- **Search** — an inverted index (for each term, the documents containing it — opposite direction from a normal index) plus stemming, fuzzy matching, BM25. **Not a source of truth**: no transactions, weaker durability, reindexing is routine. Feed via CDC.
- **Analytical** — **MPP** = massively parallel processing: the query splits across nodes, each scans its slice, partials merge. "Weak at point writes" is not a small caveat — you load in batches.

**"NoSQL = AP / eventually consistent" is outdated.** Cassandra tunes consistency per query; MongoDB has been CP by default for years with multi-document ACID since 4.0; DynamoDB offers strongly consistent reads as a request flag (2x read units) and linearizable conditional writes. **Picking a database no longer picks your consistency level** — it's still a per-operation decision.

---

## 17. Why Relational Gives Up Write Scaling

**Q: Is it because queries stop working across partitions and sharding, and replicas only help by making full copies?**

The cost instinct is right; one piece needs correcting.

**Partitioning doesn't break queries.** Within one Postgres, joins across partitions work, transactions spanning partitions are atomic, FKs work. `WHERE customer_id = ?` on a date-partitioned table returns the right answer — it just scans all 24 partitions. **Slower, not broken.** So partitioning is a **tuning step, not a scaling step** — it doesn't raise the write ceiling at all, because every partition shares one machine's CPU, disk and WAL.

**Sharding is where it breaks — and why relational specifically hurts.** Cassandra and DynamoDB shard happily. Relational doesn't, because the things that break are exactly the things that make it relational: joins, multi-row transactions, FKs, unique constraints. Every one assumes a **single coordinator** that sees all the data and orders operations against it.

A key-value store never promised joins, so distributing it costs nothing. Relational promised all of it, so distributing it means handing it all back.

> **Relational can be scaled horizontally for writes, but only by giving up the properties that made it relational.**

**On replicas:** they don't help writes *at all* — not "a bit less", **zero**. Ten replicas = ten machines each carrying the full write load.

No escape hatch within the model: one primary (ACID needs one authority ordering writes) → replicas copy, they don't divide → partitioning rearranges data on that same primary → sharding divides, and costs the guarantees.

**It's a trade, not a law.** Spanner and CockroachDB *do* give SQL, joins and distributed transactions across machines — they just show the price. Consensus per shard means every write waits for a quorum: milliseconds where Postgres takes microseconds, far worse across regions. **They paid in latency instead of in lost guarantees.**

> Relational's guarantees are *cross-row*, and cross-row guarantees need a single coordinator. Split the rows and either the guarantees go or the latency goes up. No third option.

---

## 18. ⭐ Four Families Compared

| | **Key-value** | **Relational** | **Document** | **Wide-column** |
|---|---|---|---|---|
| **Access** | Only by known key | Any field, joins across tables | Any field in a collection; joins are the weak spot | Partition key required; range-slice on clustering key |
| **Core strength** | Fastest lookup, trivial scaling | **Enforced correctness** — FKs, constraints, multi-row transactions | **Locality** — one aggregate, one read | **Leaderless writes** — any node accepts |
| **Schema** | Opaque value | Fixed, enforced by the DB | Flexible; handles heterogeneous records | Fixed per table, columns sparse/dynamic per row |
| **Ordering** | None | Any `ORDER BY`, at query cost | Any indexed field | Built into storage — free |
| **Write scaling** | Easy — nothing to give up | Hard — needs one coordinator | Easier (joins already surrendered); still one primary per shard | Easiest — no primary at all |
| **Gave up** | Every non-key question | Write scaling without sharding pain | Joins, cross-doc constraints | Ad-hoc queries; data stored once per query |
| **Wrong fight when** | You later query by something else | Writes exceed one primary | Questions keep spanning entities | You can't enumerate queries up front |

### ⭐ Locality — the point worth remembering

| | **Relational** | **Document** |
|---|---|---|
| Fetching one order | 4 tables — header, lines, address, history | 1 document |
| Physical work | 4 index lookups + join | 1 seek, 1 contiguous read |
| Why | Normalised into separate places | Whole aggregate as one blob on disk |

**It's not about schema flexibility — it's about how many places the disk has to visit.**

**The mirror image:** the same locality that makes reads cheap makes shared data expensive. A school name embedded in 500 student documents = 500 updates when it changes. Relational writes it once.

**Wide-column has its own version:** a partition is contiguous and pre-sorted, so "latest 20 for this entity" is one sequential read with no sorting at query time. Same principle, different unit — the *aggregate* in document stores, the *partition* in wide-column.

### Key-value vs wide-column

You *could* jam everything about Anna into one key-value blob. The difference is what happens next.

| | Key-value | Wide-column |
|---|---|---|
| Value | One opaque blob | Many cells, sorted by clustering key |
| Read part of it | No — fetch and parse the whole thing | Yes — range-scan a slice |
| Append one item | Rewrite the blob | One small write |
| Row size ceiling | A few MB, practically | Millions of cells |
| Ordering | None | Built in, free |

Hence **wide** — not many *rows*, but a single row that is enormously wide, with different columns per row.

### Wide-column is not a warehouse

| | Wide-column (Cassandra) | Analytical (BigQuery) |
|---|---|---|
| Workload | OLTP at scale | OLAP |
| Query | Names the key: "Anna's last 20 events" | No key: "avg session by country, last quarter" |
| Latency | Single-digit ms, millions/sec | Seconds to minutes, a few times a day |
| Storage | Row-oriented within a partition | Column-oriented across all rows |

"Columnar" means different things: **wide-column** = rows are wide with dynamic columns; **analytical columnar** = physically stored column-by-column. Similar names, different ideas.

### Choosing — in this order

| Question | If yes |
|---|---|
| 1. Do I only ever fetch by a known key? | **Key-value** |
| 2. Do queries span entities — joins, aggregation, questions I haven't thought of? | **Relational** |
| 3. Is one self-contained document the answer, and are records genuinely heterogeneous? | **Document** |
| 4. Is it "everything for one entity, over time, by recency" at very high write volume? | **Wide-column** |

If 2 and 3 both feel true → **relational**. An unexpected join in Postgres costs some SQL; in MongoDB it costs a data migration.

### Two clarifications

- **Key-only access ≠ permanent key choice.** Both true of DynamoDB, but separate ideas. Redis also needs the key, yet guessing wrong costs "flush and rebuild", not a table migration.
- **Why document joins are hard.** Not "MongoDB's joins are slower" — the model *assumes you won't need them*; you embed precisely so joins aren't required. Wanting joins repeatedly signals the wrong family, not a hard part of using it.

### Postgres JSONB — the collapse case

```sql
CREATE TABLE products (
    id    bigserial PRIMARY KEY,
    sku   text UNIQUE NOT NULL,   -- structured where it matters
    attrs jsonb                    -- flexible where it doesn't
);
CREATE INDEX ON products USING gin (attrs);
```

Flexible documents with real indexing, *plus* joins, transactions and constraints. Which is why the genuine reasons to reach for MongoDB tend to be **operational** (team knowledge, Atlas, measured sharding needs) rather than modelling.

---

## 19. Document Databases

**The picture:** relational is a filing cabinet — one drawer for names, one for hobbies, one for addresses; three drawers to learn everything about Anna. Document is one envelope per person — one trip.

> **The entire design rule: put things together that you always look at together.**

**The warning:** an envelope is only a good idea while it's about one thing. Stuff every receipt into it and it grows forever and becomes slow to open and update. Knowing when to stop stuffing is the actual skill.

**Q: Do I need the key, or can I search by any value inside the document?**

Any field — including nested fields and array contents. That's the difference from key-value.

```javascript
db.users.find({ email: "anna@x.com" })
db.users.find({ age: { $gt: 30 }, city: "Amsterdam" })
db.users.find({ "address.postcode": "5211" })   // nested
db.users.find({ tags: "premium" })               // inside an array
```

But **"can" ≠ "should"**: a query on a non-indexed field scans every document — O(n). Fine on 50,000 docs in dev, a disaster on 50 million.

```javascript
db.users.createIndex({ email: 1 })
db.users.createIndex({ city: 1, age: 1 })        // compound, order matters
```

| | Query by key | Query by any field |
|---|---|---|
| Key-value | The only option | Impossible — the value is opaque bytes |
| Document | Fast | Possible; fast if indexed, full scan if not |
| Relational | Fast | Possible; fast if indexed, full scan if not |

Document and relational land in the same row. On **single-collection** querying, document is genuinely close to relational. What it gave up is **joins** and **cross-document constraints** — not field querying.

*DynamoDB muddies this:* it stores JSON-ish documents but behaves like key-value, because it won't index arbitrary fields — only the keys and explicit GSIs (each a full copy). MongoDB indexes any field, cheaply, anytime. That's the real practical gap.

**Q: Are there tables in a document DB? I want personal details in one envelope type and shopping in another.**

Yes — **collections**:

```
database → collection → document
(schema)   (table)      (row)

shop_db
├── customers   ← personal details, one doc per person
└── orders      ← shopping, one doc per order
```

That instinct is correct, and the reason has a name: **unbounded growth.**

- Personal details are **bounded** — one name, a handful of addresses. Reaches a size and stays there.
- Orders are **unbounded** — ten years is thousands. Embedded, the envelope grows forever; MongoDB has a hard **16MB document limit**, and long before that every customer update rewrites the whole document including all orders.

> **Embed what is bounded and read together. Reference what is unbounded or read separately.**

```javascript
// customers — bounded, embed the small stuff
{
  _id: ObjectId("...42"),
  name: "Anna",
  email: "anna@x.com",
  addresses: [                        // a few, always read with the customer
    { type: "home", street: "...", postcode: "5211" }
  ]
}

// orders — unbounded, separate documents
{
  _id: ObjectId("...1001"),
  customerId: ObjectId("...42"),      // the link
  date: ISODate("2026-01-15"),
  status: "SHIPPED",
  lines: [                            // bounded within one order: embed
    { sku: "ABC", qty: 2, price: 12.50 }
  ]
}
```

**The cost:** `customerId` is a plain field — no FK, no cascade. Delete the customer and orders point at nothing. "Anna with her recent orders" is two round trips:

```javascript
const customer = await db.customers.findOne({ _id: id });
const orders = await db.orders.find({ customerId: id })
                              .sort({ date: -1 }).limit(10).toArray();

db.orders.createIndex({ customerId: 1, date: -1 });  // or that scans everything
```

`$lookup` exists (server-side left join) but is slower than a relational join and behaves poorly on sharded clusters. Most teams do two queries.

**Keep collections homogeneous.** MongoDB doesn't enforce it, but mixing shapes means mixing indexes and access patterns. Turn on `$jsonSchema` validation — *"flexible schema" shouldn't mean "no schema."*

**The counter-case:** DynamoDB's single-table design is the *opposite* advice, because DynamoDB has no joins and can't index arbitrary fields, so co-locating in one partition is the only way. MongoDB has neither constraint — **separate collections by concern is the right default there.**

**Checklist per piece of data:** grows without limit → separate collection. Always read with the parent → embed. Read on its own or shared → separate. Would push past a few hundred KB → separate. Two of three agreeing is your answer.

**Q: Can I achieve the wide-column strategy in MongoDB?**

Partly — the **modelling** translates, the **scaling** doesn't.

*Translates:* a compound index `{ userId: 1, ts: -1 }` gives "Anna's last 20 events" — that index *is* the partition-key-plus-clustering-key idea. (Note you did **not** embed events in the customer document — same unbounded rule.) The **bucket pattern** (one doc per user per month, events pushed into an array) is the closer analogue, and MongoDB **time series collections** (5.0+) do it automatically:

```javascript
db.createCollection("readings", {
  timeseries: { timeField: "ts", metaField: "deviceId", granularity: "minutes" }
})
```

*Doesn't translate:*

| | Cassandra | MongoDB |
|---|---|---|
| Writes go to | Any node | The shard's primary |
| Adding capacity | Add a node, done | Add a shard, rebalance chunks |
| Primary failover | No concept | ~10s election window |
| Row/doc size | Billions of cells | 16MB hard cap |
| Multi-datacentre writes | Native, active-active | Possible, considerably more work |

**Volume check:** Cassandra earns its keep at hundreds of thousands of writes/sec sustained, or when active-active across regions is required. Below that, MongoDB or Postgres (with TimescaleDB for this shape) serve the same pattern with far less operational pain.

---

## 20. ⭐ Wide-Column and Cassandra

**The picture — a ring of houses.** Hash Anna's name and it tells you which house is hers. **Any house will take a note** — no headmaster's office. The note is copied to the next two clockwise, so three hold it; if one burns down nothing is lost and nobody holds a meeting about who's in charge.

In Postgres or MongoDB one machine is the boss for any piece of data; if it dies, everyone pauses for an election. **In Cassandra there is no boss, ever** — no election, no pause, no failover. That's why it keeps taking writes while machines die, and why adding nodes adds write capacity in a straight line.

You must say *whose* note you want. "Whose notes mention a red bicycle?" — nobody knows, and asking means knocking on all twenty doors. So when Anna gets a red bicycle you *also* write a second note filed under "red bicycle". **Writing things twice on purpose is the design, not a workaround.**

Deleting doesn't remove a note — it adds one saying *"ignore the earlier note"*. Nobody's in charge, so you can't erase; the other houses need telling, and an absent note is indistinguishable from one that never arrived. Those pile up: **tombstones**, the most common way people get hurt by Cassandra.

> **No boss, so writing is cheap and endless. But every question must name whose data you want — and if you need a second way to ask, you store the data a second time.**

### Vocabulary

| Cassandra | Relational equivalent |
|---|---|
| Keyspace | Database/schema — also where replication is configured |
| Table | Table |
| Partition | Rows living together on the same nodes |
| Row | Row |
| CQL | SQL-lookalike query language |

### CQL looks like SQL — that's the trap

```sql
CREATE KEYSPACE chat
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

CREATE TABLE messages (
    room_id text, ts timeuuid, sender text, body text,
    PRIMARY KEY (room_id, ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

No joins, no subqueries, no cross-partition `GROUP BY`, and most SQL `WHERE` clauses are rejected. **CQL is a data-access language, not a query language** — it refuses what would be slow rather than letting you do it badly.

### The primary key does two jobs

- **`room_id` = partition key** (first element) — hashed to pick the nodes
- **`ts` = clustering key** (everything after) — sorts rows *within* the partition, and that's how they sit on disk

```
room_id = 'general'              room_id = 'random'
  10:07 · anna  · see you          09:50 · chloe · link below
  10:05 · ben   · on my way        09:42 · ben   · nice one
  10:01 · anna  · hello            09:30 · anna  · morning
```

### Which queries are legal

```sql
-- Legal: partition key given
SELECT * FROM messages WHERE room_id = 'general' LIMIT 20;

-- Legal: partition key + range on clustering key
SELECT * FROM messages WHERE room_id = 'general' AND ts > ?;

-- REJECTED: no partition key
SELECT * FROM messages WHERE sender = 'anna';
-- InvalidRequest: use ALLOW FILTERING
```

**`ALLOW FILTERING` makes it run by scanning every partition on every node.** Works in dev with 100 rows; takes the cluster down in production. Treat it as a syntax error, not an option.

Also no `LIMIT 20 OFFSET 100` — only cursor-style continuation using the clustering key.

### Clustering keys don't have to be time

**Q: Can the clustering key be something else — a value from a specific list?**

Yes. Any column works.

```sql
-- A value from a small fixed list
CREATE TABLE user_settings (
    user_id  uuid,
    category text,        -- 'privacy', 'notifications', 'display'
    setting  text,
    value    text,
    PRIMARY KEY (user_id, category, setting)
);

-- Sorting by something that isn't time
CREATE TABLE leaderboard (
    game_id text, score int, player text,
    PRIMARY KEY (game_id, score, player)
) WITH CLUSTERING ORDER BY (score DESC, player ASC);
```

**The rule you must not break — uniqueness.** Partition key + *all* clustering columns must uniquely identify a row. `PRIMARY KEY (user_id, category)` is BROKEN: Anna's three privacy settings all have `category = 'privacy'`, so each write overwrites the last and one row survives. **No error, no warning.** When picking a low-cardinality clustering column, ask *does anything else distinguish these rows?* — if not, add a discriminator (often a `timeuuid` last).

**Multiple clustering columns are a hierarchy.** Restrict left to right, no gaps, and only the **last** restricted column may use a range:

```sql
PRIMARY KEY (site_id, sensor_type, ts)

-- Legal
WHERE site_id = 'amsterdam'
WHERE site_id = 'amsterdam' AND sensor_type = 'temp'
WHERE site_id = 'amsterdam' AND sensor_type = 'temp' AND ts > ?

-- REJECTED — skipped sensor_type
WHERE site_id = 'amsterdam' AND ts > ?
```

Not arbitrary — it's what a sorted list allows. `sensor_type = temp AND ts > x` is contiguous; `ts > x` across all sensor types is not. **The column order in your PRIMARY KEY declaration decides which queries are possible.** Wrong order means rebuilding the table.

**Composite partition key** — if a low-cardinality column is always specified, put it in the partition key instead:

```sql
PRIMARY KEY ((site_id, sensor_type), ts)
```

The double parentheses split one big partition into smaller ones, spreading load better. Trade: you can no longer read all sensor types for a site in one query.

### Writes

**Every write is an upsert.** `INSERT` and `UPDATE` are the same operation — insert on an existing key overwrites, update on a missing key creates. No "already exists" error, no read-before-write, because reading first would need coordination. Conflicts resolve by **last-write-wins on timestamp, per column.**

- You can't detect that you overwrote someone. `IF NOT EXISTS` (a lightweight transaction using Paxos) gives that at ~4x cost — a code smell if routine.
- **Clock skew between nodes can decide which write wins.** NTP matters.

**TTL is built in:** `INSERT ... USING TTL 86400` expires after 24h. Useful for sessions — and expiry creates tombstones.

### Replication and consistency

RF is per keyspace (`RF = 3` = three copies). Consistency level is chosen **per query**:

| Level | Meaning |
|---|---|
| `ONE` | One replica responds. Fast, may be stale. |
| `QUORUM` | Majority — 2 of 3. |
| `LOCAL_QUORUM` | Majority *within this datacentre*. Usual production choice. |
| `ALL` | Every replica. Strongest, breaks if one node is down. |

**The rule that makes it work: `R + W > N`.** RF=3 with QUORUM write (2) and QUORUM read (2) gives 2 + 2 > 3, so the sets must overlap in at least one node — **strong consistency without a leader**.

### What simply isn't there

No joins. No FKs or unique constraints (except lightweight transactions). No cross-partition aggregation. No `ORDER BY` on anything but the clustering key. No cross-partition transactions.

All absent for one reason: each would need a node to coordinate with other nodes, and the whole design is that nodes don't coordinate.

### ⭐ Query-first modelling and the multi-table write

> **A Cassandra table isn't a place to store entities. It's a stored answer to one specific query**, with the primary key matching that query's `WHERE` and `ORDER BY`.

Note the sharper point: non-contiguous reads aren't merely *slower* — they're mostly **not allowed**. So it's not "this query is faster", it's "this is the set of questions the table can answer at all."

**Q: If I create a second table for another access pattern, do I have to update both on write?**

Yes. No trigger, no cascade — your application performs every write.

```sql
BEGIN BATCH
  INSERT INTO messages_by_room   (room_id, ts, sender, body) VALUES (?, ?, ?, ?);
  INSERT INTO messages_by_sender (sender, ts, room_id, body) VALUES (?, ?, ?, ?);
APPLY BATCH;
```

**What BATCH gives you:** a `LOGGED BATCH` is atomic and eventually applies everywhere, but **not isolated** — a concurrent reader can see table A updated and B not yet. It costs ~30% more (the coordinator writes a batch log first). Use `UNLOGGED BATCH` only for many rows in the **same partition**; across partitions it's an anti-pattern that makes things slower.

**Why this is acceptable:** writes are cheap — no read-before-write, no coordination, LSM appends. Three writes in Cassandra is genuinely cheaper than one write plus an index update in a B-tree database. You spend storage (3x, disk is cheap) and write volume (3x, writes are cheap) to make every read one contiguous fetch.

**Two alternatives, both avoided in practice:**

- **Materialized views** — Cassandra maintains the second table for you. Flagged experimental for years, known to drift from the base table and not self-repair. Know they exist; don't build on them.
- **Secondary indexes** — `CREATE INDEX ON messages (sender)` looks obvious and is usually a trap: the index is **local to each node**, so a lookup fans out to every node. Fine for high-cardinality columns queried *within a known partition*; bad as a general escape from designing tables. (SASI and newer SAI improve this; the principle holds.)

**Nothing enforces agreement between your tables.** A write to B fails after A succeeded, a deploy misses one table, a bug writes only one — Cassandra will not tell you. So:

- Put all writes for one logical event in **one place in the code** — a repository method, not scattered across services. Single most effective control.
- Writes are naturally **idempotent** (upserts), so retries are safe.
- Run a periodic **reconciliation job**. Drift is a when, not an if.

### Event-driven fan-out — and the dual-write trap

**Q: Could I publish an event on write and have consumers update the other tables? Eventually consistent?**

Yes, and it's a real pattern — but **not like this**:

```java
cassandra.insert(messagesByRoom, msg);      // succeeds
eventBus.publish(new MessageCreated(msg));  // fails — broker down
```

Permanent inconsistency, and nothing knows. No retry can fix it — the information that it should have happened is gone. (Same failure as CQRS projections: two systems, no shared transaction.)

Safe versions:

- **Publish first, then consume.** The event log is the source of truth for "this happened". Write to Kafka, commit, return; a consumer writes all Cassandra tables and doesn't advance its offset until they succeed. Retrying is harmless (upserts). Cost: async write, no read-your-own-write.
- **Transactional outbox.** Write the event into the same store as the data, in the same operation, then a relay publishes it. Clean in a relational DB; awkward in Cassandra (no cross-partition atomicity).

| | Direct writes (BATCH) | Event-driven |
|---|---|---|
| Consistency | Atomic-ish, milliseconds | Eventually, seconds |
| Read-your-own-write | Works | Breaks — may read table B before it's populated |
| Failure handling | Batch log retries | Consumer retries from offset |
| Moving parts | Cassandra only | Cassandra + broker + consumers |
| Adding a 4th table | Change the write method | Add a consumer, don't touch the write path |

**When each wins:** event-driven earns its cost with genuinely many consumers, or when derived tables live in *different* systems (Cassandra + Elasticsearch + warehouse). For two or three Cassandra tables holding the same fact, a LOGGED BATCH in one repository method is simpler, faster and has fewer ways to fail. Either way you still need reconciliation — the event bus makes drift **recoverable**, not impossible.

### The mental shift

In Postgres you model the data once and add indexes for new queries. **In Cassandra, a table *is* an index** — one materialisation shaped for one query. Adding a query means adding a table, which means adding a write.

Which is why "write down all your access patterns first" isn't advice here. **It's the design process.**

---

## 21. The Through-Line

Five rules held across every family:

**1. Every family optimises one access pattern and taxes the rest.** Nothing is generally "faster" — things are faster at the thing they were built for.

**2. Contiguity is the mechanism underneath most of it.** Columnar reads columns contiguously. Parquet row groups keep column chunks contiguous. Document stores keep an aggregate contiguous. Cassandra keeps a partition contiguous and pre-sorted. Every performance story here is a version of *how many places must the disk visit.*

**3. Guarantees need a coordinator, and coordinators don't distribute.** Relational's cross-row guarantees need one authority — which is why it can't scale writes horizontally without giving them up. Cassandra scales writes precisely because it refuses to coordinate. Distributed SQL keeps the guarantees and pays in latency. Same trade, three answers.

**4. The further right you go, the earlier you must know your queries.** Postgres: model once, add indexes later. MongoDB: decide embed vs reference up front. Cassandra and DynamoDB: the key choice *is* the design, and it's effectively permanent. **Query flexibility and write scalability trade against each other.**

**5. Duplicated data always needs reconciliation.** Cassandra's multiple tables, CQRS projections, search indexes, warehouses — the moment one fact lives in two places with no shared transaction, keeping them in step is something you own. **Build the rebuild path early.**

### The practical default

Start with a well-run PostgreSQL: relational, documents (JSONB + GIN), full-text, geospatial, time series (TimescaleDB), moderate analytics. One system, one backup story, one set of skills.

Add a specialised store when a **measured** access pattern demands it. Not a predicted one.

---

## 22. Open Topics

Threads left open, in case of a future session:

- **Shard key design in depth** — hot shards, skewed tenants, composite keys
- **The saga pattern** for cross-shard transactions
- **Cassandra internals** — read/write path, tombstone problems, repairs, compaction strategies in practice
- **Cassandra vs ScyllaDB vs Bigtable/HBase vs DynamoDB** — same family, different bargains
- **Remaining families:** graph, time series, search — none covered beyond the summary table
- **Materialized views and secondary indexes in Cassandra** — named as traps, not explored
- **Partition/shard key selection** — the general treatment promised but not given
