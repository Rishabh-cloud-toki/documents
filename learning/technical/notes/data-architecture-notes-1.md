# Data Architecture — Learning Notes

2026-09-18 · @Someone

## 1. OLTP vs OLAP vs Streaming

**Q: What is the full form of OLTP and OLAP?**

- **OLTP** = Online Transaction Processing
- **OLAP** = Online Analytical Processing
- "Online" is a 1970s-80s leftover meaning *interactive* — you type a query and wait for the answer — as opposed to batch processing with punch cards. It does not mean internet.
- The words that matter are the middle ones: **Transaction** (one small piece of business) vs **Analytical** (looking back across many of those pieces).

**The plain version**

- OLTP is the cash register. One small thing, right now, very fast, many registers at once.
- OLAP is the notebook you read at month end. Reads everything, takes a minute, tells you something you didn't know.
- Streaming is the friend at the door counting people. Remembers only the last few minutes, but tells you *now*.

One sentence: **OLTP runs the business, OLAP explains the business, streaming reacts to the business.**

**Q: Tell me about the table — purpose, access pattern, latency, volume etc.**

|  | OLTP | OLAP | Streaming |
| --- | --- | --- | --- |
| Purpose | Run the business — reads/writes of individual records | Analyse the business — aggregate over history | React to events as they arrive |
| Access pattern | Point lookups, small transactions, high concurrency | Large scans, GROUP BY, joins over big tables | Continuous queries over windows |
| Latency | ms | seconds–minutes | ms–seconds |
| Volume per query | Rows | Millions–billions of rows | Bounded by window |
| Store | PostgreSQL, MySQL, Spanner, DynamoDB, MongoDB | BigQuery, Snowflake, Redshift, ClickHouse | Kafka Streams, Flink, Materialize, Druid, Pinot |
| Schema | Normalised | Star/snowflake, denormalised, columnar | Event schema |

Only the first row is a real choice. Everything below it falls out of that choice.

**Why each row is what it is**

- **Purpose** — OLTP *changes* state correctly (money moved, seat booked); correctness beats speed, which is why it cares about transactions and consistency. OLAP *summarises* state; nothing is being changed, so it can relax rules OLTP can't. Streaming *notices* something within seconds; its enemy is delay, not volume.
- **Access pattern** — the real mental flip: in OLTP and OLAP the data sits still and queries move. In streaming, the query is written once, **stays running**, and data flows past it.
- **Latency** — ms for OLTP because a human waits on a page load, and holding a lock longer blocks everyone else. Seconds to minutes for OLAP because reading a billion rows physically takes that long and an analyst tolerates it. ms–seconds for streaming because a fraud alert 10 minutes late is worthless.
- **Volume per query** — OLTP a few rows; OLAP millions to billions; streaming whatever fits the window ("last 5 minutes"), which keeps memory bounded however long the stream runs.
- **Schema** — OLTP normalises so each fact lives in exactly one place; an update can't leave two copies disagreeing. Cost is joins on read, fine when fetching one order. OLAP denormalises (region copied onto every order row) because there's no join at query time, and historical data isn't updated anyway, so the risk normalisation protects against barely exists. Streaming's schema is just the event: what happened, when, to what — usually flat Avro, Protobuf or JSON.
- **Store** — the tools aren't interchangeable brands, they're built around these trade-offs. Postgres has row storage, indexes and locking because of the OLTP row. BigQuery and ClickHouse have columnar storage and no real transactions because of the OLAP row. Flink and Kafka Streams manage windows and out-of-order events, which is streaming's hardest problem.

**Practical note:** most real systems run all three. Postgres takes the orders, a pipeline copies them into BigQuery overnight, Kafka carries the same events to a Flink job watching for anomalies. Same data, three shapes, three purposes.

## 2. Row vs Columnar Storage

**Q: Columnar is a separate idea from denormalisation — what is it?**

A disk or a file is one long line of bytes. A table is two-dimensional. So you must pick an order to flatten it, and there are only two sensible choices: go along the rows, or go down the columns. That is the entire difference.

Take this tiny table:

| region | amount | date |
| --- | --- | --- |
| EU | 50 | Jan |
| US | 20 | Jan |
| EU | 80 | Feb |

- **Row store on disk:** `EU 50 Jan | US 20 Jan | EU 80 Feb` — each record's fields kept together.
- **Column store on disk:** `EU US EU | 50 20 80 | Jan Jan Feb` — each column's values kept together.

Same nine values, same table. Only the storage order changes.

**Win 1 — you stop reading what you don't need**

Real table: 80 columns, a billion rows. Query: `SELECT region, SUM(amount) FROM orders GROUP BY region;` — 2 columns out of 80.

- Row store: the values you want are scattered. Disk reads in blocks, not bytes, so you pull all 80 columns off disk and throw away 78. Roughly 40x more data than needed.
- Column store: `amount` is one continuous run of bytes. Read that file and the `region` file, never touch the other 78.

This isn't a clever optimisation — it's just not reading things.

**Win 2 — compression, bigger than people expect**

A billion `region` values next to each other look like `EU EU EU EU US US EU...`. Same type, few distinct values, often already sorted.

- Run-length encoding turns a million repeats into "EU x 1,000,000".
- Dictionary encoding turns each region into a 1-byte number.
- 10x compression on a column is common, sometimes far more.

In a row store the neighbours are `EU`, a float, a timestamp, a UUID — nothing looks like its neighbour, so there's almost nothing to compress. Less data on disk compounds with win 1.

**Win 3 — the CPU gets to be stupid**

A column of integers in memory is a flat array of one type. The CPU can add 8 per instruction (SIMD), no branching, no pointer chasing. ClickHouse and similar engines process in "vectors" of a few thousand values for exactly this reason. A row store forces you to hop through a record and decode each field's type, one at a time.

**Why you'd never do this to your OLTP database**

`SELECT * FROM orders WHERE order_id = 48213;`

- Row store: one seek. The whole record sits together in one block.
- Column store: 80 separate reads, one per column file, reassembled into a row. You made the common OLTP query 80x worse.

Writes are worse still. Inserting one order means appending to 80 files instead of one. Updating a single field means breaking a compressed run apart. That's why column stores usually don't support proper row-level updates — ClickHouse calls them "mutations" and they rewrite whole chunks. Deletes are often just a flag saying "ignore this row".

**The trade, in one line**

Column stores optimise for reading **few columns of many rows**. Row stores optimise for reading **many columns of few rows**. Neither is better; they answer different questions.

## 3. Parquet

**Q: Will a Parquet file always be columnar storage based?**

Yes. It's not a configuration option — it's the format's whole reason for existing. Same for ORC.

But "all of column A, then all of column B" isn't quite how it works.

**Row groups — the hybrid layout**

Parquet splits the rows into **row groups** first (typically 100k–1M rows, roughly 128MB–1GB). *Inside* each row group, the data is stored column by column.

```
orders.parquet
├── Row group 1 (rows 1–100k)
│     ├── column chunk: region
│     ├── column chunk: amount
│     └── column chunk: date
├── Row group 2 (rows 100k–200k)
│     ├── column chunk: region
│     ├── column chunk: amount
│     └── column chunk: date
└── Footer — schema, row group offsets, min/max per column chunk
```

Why row groups exist: a pure column-per-file layout would mean reconstructing a row requires seeking across the entire file — awful on object storage like S3, and impossible to parallelise. Row groups make each chunk self-contained: a worker grabs row group 7 and processes it alone, and reassembling a row never reaches outside its own group.

**The footer and pruning**

The footer is read first and holds min/max values for every column chunk. If your query says `WHERE date > 'Feb'` and row group 1's `date` chunk has max = `Jan`, the engine skips that entire row group without reading a byte. A well-sorted Parquet file can answer a query touching 1% of the data by reading roughly 1% of the file.

**Row-based file formats — the contrast**

- **Avro** — row-based. Often confused with Parquet because they appear in the same pipelines, but Avro is for moving records around (Kafka messages, event streams), where you want whole records.
- **CSV, JSON, JSONL** — row-based, one record per line.
- **Parquet, ORC** — columnar, for analytics.

Rule of thumb: **Avro for data in motion, Parquet for data at rest.**

**Caveat on "always columnar"**

The format is fixed, but the *benefit* depends on how you write it. A Parquet file with one row group and one column is technically columnar and pointlessly so. Badly written Parquet is a real thing — usually thousands of tiny files with 500 rows each. You get the benefit by writing sensibly sized row groups and sorting on the column you filter by.

## 4. Choosing a Datastore — the Framework

Ask these in order. The answers usually eliminate all but one or two options.

1. **What are the access patterns?** Write them all down as concrete queries. Model the data to serve these, not an abstract entity diagram.
2. **What consistency does each operation need?** Linearizable (a balance, an inventory count, a unique username) vs eventual (a feed, a product page, a recommendation). Per-operation, not per-system.
3. **What is the read:write ratio and absolute volume?** 1000:1 reads → cache + replicas. Write-heavy at scale → LSM-based store, partitioning from day one.
4. **How structured and how variable is the data?** Fixed relational, flexible documents, a graph of relationships, append-only time series, or full-text.
5. **What are the query needs?** Ad-hoc joins and aggregation → relational. Known key-based access → KV / wide-column. Relationship traversal → graph. Search/ranking → search engine.
6. **Scale ceiling and growth?** Will a single primary with read replicas carry you 3 years, or do you need horizontal write scaling now?
7. **Operational reality:** team skills, managed-service availability, backup and DR story, cost model (per-request vs provisioned vs node-hours).

**Why the order does work**

- Questions 1–2 are about **correctness** — get these wrong and the system is broken.
- Questions 3–6 are about **scale** — get these wrong and the system is slow or expensive.
- Question 7 is about **you** — get this wrong and the system is unmaintainable. It's last but it vetoes the others.

Most people start at question 5 or 6 ("we need something that scales") and work backwards. That's how teams end up with Cassandra holding 40GB that Postgres would have served happily, plus a team that can't debug it at 2am.

**Notes on the individual questions**

- **Q3** — two separate numbers, often conflated. The *ratio* tells you the shape of the fix: read-heavy is easy (copies), write-heavy is hard (every copy must be written to). The *absolute volume* tells you whether you have a problem at all — 1000:1 at 100 req/sec is a laptop; at 500,000 req/sec it's an architecture.
- **Q3, on LSM** — B-trees (Postgres, MySQL) update in place: a write is a random disk write. LSM trees (Cassandra, RocksDB, ScyllaDB) append to a log and sort it out later in the background. Appends are sequential and much faster; the cost is reads may check several places and compaction burns CPU and disk.
- **Q4 warning** — "our schema changes a lot" is usually not a reason to go document-store. It's often a reason to get better at migrations. Schema flexibility isn't free; you've moved the schema from the database into application code, where nothing enforces it.
- **Q4 vs Q5** — Q4 is the data's shape, Q5 is what you do with it. When they disagree, Q5 wins: you can store relational data in a KV store, but you can't make a KV store do ad-hoc joins.
- **Q5, graph test** — specifically *multi-hop traversal*. One or two hops is a join and Postgres is fine. "Find all accounts within 4 degrees of this flagged account" is a recursive CTE that gets slow fast — that's Neo4j territory.
- **Q5, search test** — *ranking*. "Best matching results by relevance, with typo tolerance and stemming" is Elasticsearch. "Rows containing this word" is Postgres full-text.
- **Q6** — most people wildly underestimate one well-tuned Postgres primary: tens of thousands of transactions per second, terabytes of data, one machine. The real question is whether you need horizontal *write* scaling; read scaling is easy and vertical scaling goes far.
- **Q7** — can your team debug this at 3am? Is there a managed version? Have you actually **tested a restore**? An untested backup is not a backup. Per-request pricing is great for spiky traffic and brutal for steady high volume; provisioned is the opposite.

**The default: start with a well-run PostgreSQL**

It does OLTP, JSONB documents with indexes, full-text, PostGIS geospatial, and moderate analytics. One system, one backup story, one set of skills, one thing to monitor.

Every store you add multiplies operational surface: another failure mode, another backup, another runbook — plus the hardest problem, keeping data consistent *across* stores, which nobody's transaction guarantees cover.

The key word is **measured**: add Elasticsearch when your Postgres full-text search is measurably too slow, not when you imagine it will be.

**Worked example — order management for a mid-size retailer**

1. Patterns: get order by id (high volume), list a customer's orders by date, update status, revenue by region by month (a few times a day), free-text address search.
2. Consistency: order creation and status updates linearizable; revenue report can be a day stale; search can lag a minute.
3. Volume: \~200 orders/minute peak, read:write \~50:1. Small.
4. Shape: firmly relational.
5. Query needs: joins and aggregation, plus one full-text case.
6. Ceiling: a single primary handles this for years.
7. Ops: small team, RDS available.

→ **Postgres.** Add a read replica so the reporting query doesn't lock up the primary. Use Postgres full-text for address search initially. Nothing else until something measured says otherwise.

Now change one input — the retailer becomes a marketplace at 50,000 orders/minute with sub-second fraud checks. Q2, Q3 and Q6 all change answers, and you end up with Postgres for orders, Kafka + Flink for the fraud stream, and a warehouse for reporting. Same framework, different answers.

## 5. Access Patterns First, Not Entities

**Q: Is the intent here to decide columnar vs row based?**

No — that's a much smaller decision, and it's usually already settled by purpose: running the business is row, analysing it is columnar.

Question 1 decides something harder: **what shape the data physically takes, and how it's keyed.**

**What a written-down access pattern actually determines**

| Pattern | What it decides |
| --- | --- |
| "Get order by id" | `order_id` is the primary access key. Trivial in Postgres. In DynamoDB this is your partition key — and that choice is permanent. |
| "List orders for a customer, sorted by date" | `customer_id` and `date` must be co-located in storage order. Postgres: composite index on `(customer_id, date)` — the column order matters, and you only know it because you wrote the query down. DynamoDB: partition key `customer_id`, sort key `date`. |
| "Sum revenue by region by month" | `region` and `month` must be available without joining to a customer table across a billion rows → denormalise region onto the order row. Also tells you this query shouldn't run on the primary. |

None of those answers was "columnar" or "row". They were: which key, which sort order, which index, what gets copied where.

**Where it does touch storage layout — indirectly**

The third pattern is many rows, few columns, aggregate. That's the signature of an OLAP query, and OLAP means columnar. But you didn't learn "columnar" from the access pattern — you learned that this pattern belongs in a **different system** from the first two. That's the real output: some of your patterns don't belong in your main database.

**Why "not entities" is the sharp part**

Entity-first gives you a clean normalised model that answers every query *adequately* and none of them *well*. You draw Customer, Order, OrderLine, Product, normalise, then discover the customer-orders-by-date screen does a join plus a sort over 200k rows on every page load.

Access-pattern-first flips it: **the queries are the requirement, the model is the implementation.**

- In Postgres this mostly changes your indexes and a few denormalisations.
- In a KV or wide-column store it changes *everything*. Your table structure is a transcription of your query list, and people routinely store the same data three times under three different keys to serve three access patterns. Insane in Postgres, completely normal in DynamoDB.

**The practical rule**

Write the query list, with frequencies ("get order by id — 5000/sec" is a different design input from "revenue by region — 4x a day"). Then for each one ask: *what would have to be true on disk for this to be one cheap lookup?*

If the answer conflicts between two queries, you've found either an index you need, a denormalisation you need, or a second store you need. That conflict is the output of question 1, and it feeds every question after it.

## 6. Consistency Per Operation

**Q: How will my choices be affected — linearizable vs non-linearizable?**

**What linearizable means:** every read sees the most recent completed write, and the system behaves as if there is exactly one copy of the data. No reading a value from 200ms ago; no two clients disagreeing about what the value is.

**What it costs:** all operations on that data must funnel through one authoritative point — a single primary, or a quorum agreeing with each other. Funnelling costs latency, costs availability when that point is unreachable, and caps throughput.

So the same application's three operations go down three completely different paths:

| Operation | Requirement | Where it runs |
| --- | --- | --- |
| Decrement inventory | Linearizable | Primary, in a transaction. No cache, no replica. |
| Show product page | Seconds stale is fine | Cache or read replica. TTL = your staleness budget. |
| Recommendation feed | Minutes stale is fine | Precomputed KV store, rebuilt by a batch job. |

**Inventory — what linearizable forces on you**

The classic bug is read-modify-write in application code:

```java
int qty = repo.getQuantity(productId);   // reads 1
if (qty > 0) {
    repo.setQuantity(productId, qty - 1); // writes 0
}
```

Two threads both read 1, both write 0. Two units sold, one in stock. The gap between the read and the write is where the money leaks.

The fix is to make check-and-update a single atomic operation the database evaluates:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = ? AND quantity > 0;
```

Then check the affected row count. Zero rows = already sold out, reject the order. The database holds a row lock, so the second transaction waits and sees `quantity = 0`.

What this rules out:

- **No reading inventory from a read replica during checkout.** Replica lag is usually milliseconds — "usually" is exactly the assumption that breaks under load.
- **No caching the count for the decision.** You can cache it for *display* ("only 3 left!") — that's a display, not a decision.
- **No eventually-consistent store for this row.** DynamoDB can do it (conditional writes are linearizable on a single item); Cassandra needs lightweight transactions, which are slow enough that people regret it.

**Single-key vs multi-key — the thing that makes it affordable**

Inventory is a **single-key** operation: all contention is on one row keyed by `product_id`. Single-key linearizability is cheap everywhere.

*Multi-key* linearizability — "decrement inventory AND charge the card AND create the order, atomically" — is what gets expensive, because you need a distributed transaction across systems that don't share one.

So most real checkout flows use a **reservation**: hold the stock for 10 minutes (a linearizable single-key write), then take payment, then confirm. If payment fails, the hold expires. One hard distributed transaction traded for two easy local ones plus a timeout.

**Product page — what eventual buys you**

Redis in front, read replicas behind, CDN at the edge. Your TTL is literally your staleness budget as a number.

Two traps:

- **Read-your-own-writes.** A seller edits their product, the page reloads from a lagging replica, they see the old description and edit again. Fix: a *sticky read* — for a few seconds after a user writes, route **that user's** reads to the primary. Everyone else keeps hitting the replica.
- **Stale prices are a business decision, not a technical one.** Cached page shows €40, real price is €45 — what happens at checkout? Most retailers honour the displayed price and eat the difference. Someone has to decide that; it shouldn't be a surprise.

**Recommendations — no transaction at all**

A batch or streaming job computes them and writes into a KV store. Serving is a single key lookup, sub-millisecond. If the job fails, users see yesterday's recommendations and nobody notices. No correctness to protect, so you buy nothing and pay nothing.

**The pattern underneath**

Ask one question per operation: **if this value is 200ms out of date, what breaks?**

- Money or a count goes wrong → linearizable, primary, transaction, no cache.
- Someone sees a slightly old page → eventual, cache freely.
- Nobody can even tell → precompute it offline.

The failure mode in real systems is almost never "we chose the wrong database". It's that one operation quietly got routed down the wrong path — someone added a read replica for performance and a checkout query moved onto it. Mark each operation's requirement explicitly, in code or a comment, rather than leaving it as tribal knowledge.

## 7. Stale Reads and Safe Writes

**Q: Two users read a stale value from a replica and then do a linearizable operation on it. How do I make it correct?**

**You don't fix this by making the read fresh. You fix it by making the write refuse to commit against state that changed.**

The stale read is not the bug. Reads from a replica are always potentially stale — even a read from the primary is stale by the time your application acts on it, because time passed. The bug is *trusting* the value at write time. Push the check down to the one place that can't be stale: the primary, inside the write.

**Case A — the write can be expressed relative to current state**

Inventory, balances, counters. You don't need the value you read, you need the *change*.

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = ? AND quantity > 0;
```

The replica said "3 left". Doesn't matter — the write never uses the number 3. The primary evaluates `quantity > 0` against the truth at that instant. Both users can read stale values from different replicas and the outcome is still correct.

Cheapest and most robust answer. Whenever you can phrase the write as a relative change with a guard condition, do that, and the stale-read problem disappears.

**Case B — the write depends on what the user actually saw**

Editing a product, approving an order at a displayed price. Here the decision *was* based on the stale value, so the write must detect that the world moved. Add a version column to the write condition:

```sql
UPDATE product
SET price = 45, version = version + 1
WHERE id = ? AND version = 7;
```

The read returned the row *and* `version = 7`; you carry that token through to the write. If someone else updated it in between, version is 8, zero rows match, and the write fails loudly instead of silently overwriting their change.

Spring Data / JPA has this built in:

```java
@Entity
public class Product {
    @Id private Long id;
    @Version private Long version;
    private BigDecimal price;
}
```

A stale version throws `ObjectOptimisticLockingFailureException` on flush. This is **optimistic concurrency control** — same idea as HTTP `ETag` + `If-Match`. "Optimistic" because you assume conflicts are rare and pay only for detection, not locking.

**What to do when the check fails** (the part people skip)

- Case A (inventory): don't retry. The answer is genuinely "sold out". Reject and tell the user.
- Case B (concurrent edit): reload the current row. A background job incrementing a counter can retry with backoff, 2–3 attempts. A human editing a form should usually be **shown** the conflict — silently retrying means silently discarding somebody's edit.
- Make the operation **idempotent**: a client-generated idempotency key means a retry after a timeout doesn't decrement stock twice.

**When you genuinely need a fresh read**

Sometimes the logic can't be pushed into the database — multi-row invariants, a rules engine, an external call. Then route *that specific read* to the primary (`@Transactional` without `readOnly = true` in a typical Spring setup). The point is that this is a per-operation routing decision, not a global one. Checkout reads primary; the product page reads replicas.

Two intermediate options if you want to keep replica reads:

- **Sticky / read-your-own-writes** — after a user writes, pin that user's reads to the primary for a few seconds.
- **Bounded staleness by LSN** — capture the primary's WAL position at write time and have the replica read wait until it has replayed past it. Postgres exposes `pg_current_wal_lsn()` and `pg_last_wal_replay_lsn()`; Aurora and proxies like pgpool/ProxySQL can do this for you. Real, but adds latency and complexity — only when measured.

**Where pessimistic locking fits**

```sql
BEGIN;
SELECT quantity FROM inventory WHERE product_id = ? FOR UPDATE;
-- decide in application code
UPDATE inventory SET quantity = ? WHERE product_id = ?;
COMMIT;
```

Correct, but it holds a lock across your application logic — serialising every buyer of a hot product, and deadlocking if you lock rows in inconsistent order. Use it when conflicts are frequent and retrying is expensive. For normal checkout, the Case A conditional update is better: microseconds of lock instead of milliseconds.

**The rule**

Never let application code perform **check-then-act across a network boundary**. Either express the condition inside the write (`WHERE quantity > 0`, `WHERE version = ?`), or hold a lock for the whole read-decide-write span. A stale read followed by an unconditional write is the bug — and it's still a bug if you read from the primary, because another transaction can slip in between your read and your write regardless.

## 8. Write-Heavy Patterns

**Q: Read-heavy gets replicas, caches and CQRS. What patterns exist for write-heavy?**

CQRS helps the write side too: once reads are served from separate projections, the write model stops carrying indexes and denormalisations that existed only for queries. A write table with 2 indexes is far faster than one with 9.

The patterns form a ladder. Work down it in order — each step costs more than the last.

**1. Don't write it (coalescing and sampling)**

The cheapest write is the one you skip. "User last seen at" on every request becomes one update per minute — keep it in memory, flush periodically. Metrics get sampled or pre-aggregated at the client. View counts get buffered and flushed as `+247` instead of 247 increments.

Unglamorous, and routinely removes 90% of write volume. Do it first.

**2. Batch them**

Databases are far better at one statement writing 1000 rows than 1000 statements writing one row. Each round trip costs network latency, parsing, transaction overhead, often an fsync.

- Hibernate: `spring.jpa.properties.hibernate.jdbc.batch_size` = 50–100
- Postgres bulk load: `COPY` is roughly an order of magnitude faster than `INSERT`

**3. Cut the cost of each write**

- **Indexes are a write tax.** Every index is another B-tree to update on insert. Audit them; unused indexes are pure cost.
- **Synchronous replication doubles write latency.** Every commit waits for a replica ack. If durability allows async, you get that back.
- **Group commit / relaxed fsync.** Postgres `commit_delay` and `synchronous_commit = off` trade a small window of potential data loss for a large throughput gain. Fine for logs and events, not for money.

**4. Put a buffer in front (write-behind)**

App writes to Kafka or a queue, returns immediately; a consumer writes to the database at a steady rate.

What this actually buys: **absorbing spikes.** Your DB might handle 5k writes/sec sustained, but traffic arrives at 40k/sec for 30 seconds during a flash sale. The queue flattens that.

It does **not** increase long-run capacity — if the average exceeds what the consumer can drain, the queue grows until it falls over.

Cost: the write is now asynchronous. The user gets "order received", not "order confirmed". You need idempotency keys (at-least-once delivery means duplicates) and a plan for consumer failure.

**5. Change the shape: append instead of update**

Update-in-place is expensive — find the page, lock it, modify, write WAL, and in Postgres write a whole new row version that autovacuum must clean up later.

Appending is cheap and never contends. **Event sourcing** takes this to its conclusion: never update the balance, append `Deposited(100)`, `Withdrew(30)`, derive the balance. Writes become pure appends; reads become a projection maintained separately — which is exactly CQRS, and why the two are so often paired.

This is also what LSM stores do internally, which is why Cassandra and RocksDB are write-optimised.

**6. Spread the writes: partitioning and sharding**

Where horizontal write scaling actually happens, and the expensive step — hence last, but designed for early.

The entire game is the **partition key**. Get it wrong and you have a hot partition: you've bought a distributed system and still have a single-node bottleneck.

Classic mistake: partitioning event data by timestamp. All of today's writes land on one partition while the other 19 sit idle. Fix: a key with natural spread (`customer_id`, `device_id`, or a hash) with time as the *sort* key, not the partition key.

**7. The single-row contention case**

Sometimes the problem isn't total volume — it's that 10,000 writes/sec all target *one row* (a like counter on a viral post, a global sequence). No amount of sharding helps, because it's one key.

- **Sharded counters** — split into 20 rows (`counter_id` 0–19); each write picks one at random, reads sum all 20. A cheap read traded for a 20x cheaper write.
- **Aggregate in a stream** — Kafka + Flink counts in memory over a window and flushes a periodic total. Seconds stale, which for a like counter is fine.

**How this maps to the framework**

Steps 1–3 don't change your architecture at all — they're tuning, and that's where your first effort should go. Step 4 changes your consistency guarantees. Steps 5–6 change your data model and are hard to retrofit, which is exactly why "partitioning from day one".

Honest check before any of it: a well-tuned Postgres on decent hardware does tens of thousands of writes per second. Confirm you're near that ceiling before reaching past step 3.

## 9. LSM Stores

**Q: What are LSM stores?**

**LSM = Log-Structured Merge tree.** The design comes from one hardware fact: writing to a random location on disk is slow; writing sequentially to the end of a file is fast. Even on SSDs the gap is large.

- A **B-tree** (Postgres, MySQL/InnoDB) updates in place: find the page, read it, modify, write it back — a random I/O, plus WAL, plus the same for every index.
- An **LSM tree** refuses to do that. **It never modifies anything on disk. It only ever appends.**

**The write path**

1. Append to a write-ahead log (crash recovery).
2. Insert into the **memtable** — a sorted structure in RAM, usually a skip list or balanced tree.
3. Return. Both operations are cheap, so writes are fast with predictable latency.
4. When the memtable fills (say 64MB), flush it to disk in one sequential pass as an **SSTable** (Sorted String Table): keys in sorted order plus a sparse index. Once written, that file is **immutable** — never edited again.

```
In memory:   Writes → Memtable (sorted, not yet on disk)
                        ↓ flush when full
On disk:     Level 0:  [SSTable][SSTable][SSTable]   newest, may overlap
                        ↓ compaction
             Level 1:  [ SSTable ][ SSTable ]        merged, no overlap
                        ↓
             Level 2:  [      SSTable       ]        largest, oldest
```

**Updates and deletes when nothing can be modified**

- An **update** just writes the new value into the memtable. The key now exists in two places; the rule is simply **the newest copy wins**.
- A **delete** writes a **tombstone** — a marker saying "this key is deleted". The old value still sits on disk and disappears later, during compaction.

**The read path — the hard part**

Check the memtable first, then each SSTable newest to oldest, stopping at the first hit. A key that doesn't exist anywhere means checking everything — the worst case, and why LSM reads are harder than B-tree reads.

Two things rescue it:

- **Bloom filters** — each SSTable carries a small probabilistic structure answering "is this key definitely *not* here?" It says "definitely not" (skip the file, no disk I/O) or "maybe" (go look). A few bits per key eliminates almost all pointless reads.
- **Sparse indexes** — because keys are sorted, a small in-memory index of every Nth key tells you which block to read.

**Compaction — the background merge**

Files accumulate, so a background process merges them: read several SSTables, merge-sort them (cheap, already sorted), keep only the newest version of each key, physically drop tombstoned rows, write one new file, delete the old ones. This is where wasted space is reclaimed and where deletes finally take effect.

- **Leveled** (RocksDB, Cassandra LCS) — each level \~10x bigger than the one above, no overlap within a level. Better reads, more compaction work.
- **Size-tiered** (Cassandra default) — merge files of similar size. Better writes, worse reads, and can temporarily need double the disk space.

**The three amplifications — the real vocabulary**

- **Write amplification** — one logical write gets rewritten several times by compaction. 10–30x is normal. You still win, because all of it is sequential.
- **Read amplification** — one logical read may touch several files.
- **Space amplification** — old versions and tombstones occupy disk until compaction runs.

B-trees have low read and space amplification but high random-write cost. LSM inverts that. That's the whole trade.

**Operational gotchas**

- Compaction competes with live traffic for CPU and disk. This causes latency spikes invisible in average response time but obvious at p99. Compaction tuning is an ongoing job, not a one-time setting.
- Classic Cassandra failure: a queue-like table where rows are constantly deleted. Reads scan through piles of tombstones before finding live data, and queries crawl. LSM dislikes heavy deletes on the same key range.

**Who uses what**

- **LSM:** Cassandra, ScyllaDB, RocksDB, LevelDB, HBase, ClickHouse MergeTree, MongoDB's WiredTiger engine
- **B-tree:** PostgreSQL, MySQL/InnoDB, Oracle, SQL Server

**When it's right:** sustained high-volume writes with a known key access pattern — event ingestion, time series, messaging, IoT telemetry, write-heavy KV — especially with a natural partition key that spreads writes evenly.

**When it's wrong:** ad-hoc analytical queries, heavy joins, complex transactions, or any write volume comfortably within a single relational database's ceiling. Which covers most systems.

## 10. Partitioning vs Sharding — the Basics

**Q: I've heard about partitioning and sharding but never implemented them. Explain simply first.**

**The toy-box version**

You have one toy box. Every toy goes in it. Finding anything means digging through everything, and eventually the lid won't shut.

So you get more boxes and make a **rule** for which toy goes where — dinosaurs in box 1, cars in box 2, LEGO in box 3. Finding a dinosaur means opening one box, not four. New toys spread out instead of piling into one place.

**That rule is the only thing that matters.** Pick a bad rule — "toys I got before my birthday" in box 1, "after" in box 2 — and box 2 grows forever while box 1 sits still. Four boxes, still one problem.

**The two words**

- **Partitioning** = splitting the toys across several boxes *in your own room*. One room, one kid, several boxes.
- **Sharding** = the boxes are in different houses. You need to know which house to walk to, and wanting dinosaurs *and* cars means visiting two houses.

Same idea — split by a rule — but sharding crosses a **machine boundary**, and that's what makes it hard. Within one machine the database can still join across partitions, keep transactions atomic, and count things. Across machines it can't do any of that for free.

Hence: **partition early, shard late.** Partitioning is a tuning decision. Sharding changes what your application is allowed to do.

**The one word to hold onto**

The rule for deciding which box is the **partition key** (or shard key). Almost every real problem in this space — hot partitions, expensive queries, migrations you can't undo — traces back to that one choice.

## 11. Partitioning Inside One Database

**Two things share the name "partitioning"**

- **Vertical partitioning** — splitting *columns* into separate tables. Move rarely-read `description TEXT` and `image_blob` out of `products` into `product_details`. The hot table gets narrow rows, so more rows fit per 8KB page and scans read fewer pages. A hand-done modelling decision; not what people usually mean.
- **Horizontal partitioning** — splitting *rows* across several physical tables by a rule. One logical table, many physical ones. This is what everyone means.

**Three strategies**

- **Range** — by a continuous value, usually a date. Most common by far.
- **List** — by discrete values (`region IN ('EU','UK')`).
- **Hash** — hash the key, modulo the partition count. For even spread with no natural range.

**Postgres example** (declarative partitioning since PG 10, genuinely good from 12–13)

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

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

The `DEFAULT` partition catches rows matching no range. Useful as a safety net, dangerous as a habit — rows silently pile up, and you can't add a new partition covering a range that already has rows in default without moving them first. Some teams deliberately omit it so a missing partition fails loudly.

**What makes it fast: partition pruning**

```sql
EXPLAIN SELECT sum(amount) FROM orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01';

-- Aggregate
--   ->  Seq Scan on orders_2026_02 orders
```

One partition in the plan. The others aren't scanned or opened.

Now the query that ruins it:

```sql
SELECT * FROM orders WHERE customer_id = 4711;
```

No partition key in the `WHERE`, so Postgres scans **every** partition — one index lookup became 24. Partitioning made this query worse. Same trade as everything else: you optimise for the access patterns you wrote down and pay for the ones you didn't.

Two refinements:

- Pruning happens at **execution time** too (PG 11+), not just planning. Parameters in prepared statements and values known only from a join can still prune. Older tutorials claim otherwise.
- Pruning needs the **raw column**. `WHERE date_trunc('month', order_date) = '2026-02-01'` won't prune — the planner can't reason through the function. Write the range out.

**The benefit people actually partition for: data lifecycle**

```sql
-- Deleting 50M rows normally: hours, massive WAL, bloat, vacuum pain
DELETE FROM orders WHERE order_date < '2025-01-01';

-- With partitioning:
ALTER TABLE orders DETACH PARTITION orders_2024_12;
DROP TABLE orders_2024_12;
```

Near-instant metadata work. No row-by-row deletion, no dead tuples, no vacuum storm. If you have a retention policy, this alone justifies partitioning.

- Use `DETACH PARTITION ... CONCURRENTLY` in production to avoid a heavy lock.
- `ATTACH PARTITION` scans the table to verify rows fit the range — add a matching `CHECK` constraint *before* attaching and Postgres trusts it and skips the scan.

**Other real benefits**

- **Smaller indexes** — a B-tree over 24 monthly partitions is 24 shallower trees. Better cache locality; the current month's index may fit entirely in RAM.
- **Autovacuum parallelism** — each partition vacuumed independently; old read-only partitions stop being vacuumed at all. On a huge single table vacuum becomes chronic; partitioning largely dissolves it.
- **Maintenance granularity** — `REINDEX`, `ANALYZE`, backups, even `VACUUM FULL` operate on one partition, locking only that slice.
- **Parallel scans** across partitions for analytical queries.

**The costs, honestly**

- **Planning overhead** — each partition costs planning time. A few hundred is fine on modern Postgres; a few thousand hurts, especially for OLTP where planning time rivals execution. Prefer monthly over daily without a reason.
- **No global unique index** (see next section).
- **`UPDATE` changing the partition key** moves the row between partitions — implemented as delete + insert. Works since PG 11, but expensive and invalidates cursors.
- **`CREATE INDEX CONCURRENTLY`** can't run on the parent directly: create on each partition concurrently, then on the parent with `ONLY`, then attach. Awkward but scriptable.
- **Foreign keys** *from* a partitioned table are fine; keys *pointing at* one work from PG 12 but must reference the full key including the partition column.

**Automation**

Nobody creates partitions by hand forever. **pg\_partman** is the standard extension — pre-creates future partitions, drops or archives old ones on a schedule. Without it you eventually forget, rows land in `DEFAULT` or fail outright, and you find out at 2am.

**Sub-partitioning**

```sql
CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01')
    PARTITION BY HASH (customer_id);
```

Helps when a single month is still too large and writes concentrate on it. But it multiplies partition count fast — 24 months x 8 hashes = 192 tables. Only when measurement says the month partition is the bottleneck.

**How other databases differ**

- **Oracle** — most mature, and has what Postgres lacks: **global indexes** spanning all partitions. So `order_id` *can* be globally unique, and a query on a non-partition-key column stays a single index lookup. Cost: dropping a partition invalidates the global index (`UPDATE GLOBAL INDEXES`), reintroducing the work you were avoiding. Also has interval partitioning (auto-creates the next partition) and reference partitioning (child table partitioned like its parent).
- **MySQL** — supports RANGE/LIST/HASH/KEY, but every unique key must contain the partition column and there's **no foreign key support at all** on partitioned tables. Much less used.
- **SQL Server** — partition functions and schemes; partition switching is its equivalent of detach.

**When not to partition**

Below roughly 50–100GB a well-indexed single table usually beats a partitioned one — you get the planning overhead and constraint restrictions without a meaningful payoff.

Honest triggers: a retention policy you want to enforce cheaply, a table whose vacuum or index maintenance has become unmanageable, or a measured query pattern that prunes well. And check first that your access patterns actually contain the partition key — if your hot query is `WHERE customer_id = ?` and you partition by date, you've made the system slower and more complicated.

## 12. Partition Keys and Unique Constraints

**Q: Do we need the partition key in the primary key so Postgres can check all partitions for uniqueness?**

**It's the reverse.** Including the partition key doesn't let Postgres check across all partitions — it lets Postgres **avoid** having to.

Postgres has only **local indexes**: one index per partition, each knowing nothing about the others. The only way it can enforce a unique constraint is if it can guarantee that two rows which would collide are *physically in the same partition*. Requiring the partition key in the unique key guarantees exactly that: two rows with the same `(order_id, order_date)` must have the same `order_date`, so they route to the same partition, so checking that one partition's local index is sufficient.

**It's a routing guarantee, not a global check.**

**Q: But what if order\_id is the same and order\_date is different? For me that's a duplicate.**

Exactly — and Postgres will not stop you. The constraint you declared is `UNIQUE (order_id, order_date)`, not `UNIQUE (order_id)`:

```sql
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-01-15');  -- ok
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-02-20');  -- also ok!
INSERT INTO orders (order_id, order_date) VALUES (5, '2026-01-20');  -- ok too
```

The third one is in the *same* partition as the first, but the pair differs, so the composite index sees no conflict. Duplicate `order_id`s are not prevented anywhere. Nothing is checking it at all.

**Why Postgres made this choice**

A global unique index would be consulted on every insert into every partition — a single shared structure all partitions contend on, undoing much of the point. Worse, it would break cheap `DETACH PARTITION`: you couldn't detach without rewriting or invalidating a global index referencing its rows. Oracle accepts that cost and offers global indexes; Postgres declines.

**Four ways to actually get a unique order\_id**

**1. Generate IDs that can't collide (the usual answer)**

```sql
order_id bigserial   -- or GENERATED ALWAYS AS IDENTITY
```

The sequence lives on the parent and is shared by every partition; it never returns the same value twice. Same for UUIDv7 or Snowflake IDs (UUIDv7 is time-ordered, so unlike UUIDv4 it doesn't destroy index locality).

Covers \~95% of cases. The gap: it's a *convention*, not a constraint. A bulk load supplying explicit IDs, a restore into the wrong partition, or a migration script assigning its own IDs sails straight through.

**2. Partition by something the ID already implies**

```sql
CREATE TABLE orders (...) PARTITION BY HASH (order_id);
```

Now `PRIMARY KEY (order_id)` is legal — a given `order_id` always hashes to the same partition, so the routing guarantee holds with `order_id` alone. Real, enforced uniqueness.

Catch: you lose date-range pruning and cheap `DROP PARTITION` for retention. Right call when your dominant access is by ID with no retention policy; wrong call when you partition specifically to age out old data.

**3. Derive the partition key from the ID**

If IDs are time-ordered (UUIDv7, Snowflake, an ascending sequence), the date is redundant:

```sql
PARTITION BY RANGE (order_id);
-- orders_p1: 0 .. 100_000_000
-- orders_p2: 100_000_000 .. 200_000_000
```

`PRIMARY KEY (order_id)` works, and because IDs ascend with time, old partitions still hold old data and remain droppable. You lose direct pruning on an arbitrary date range, though a lookup table of "which ID range covers which month" recovers most of it. A reasonable middle path when both uniqueness and retention matter.

**4. Enforce it yourself with a registry table**

```sql
CREATE TABLE order_id_registry (
    order_id bigint PRIMARY KEY
);
```

Insert into the registry in the same transaction as the order. A normal unpartitioned table with a real unique index, so duplicates fail there.

Cost: every insert touches a second, ever-growing, unpartitioned table — precisely what you partitioned to avoid. A shared contention point that can't be aged out. Reserve for cases where a duplicate would be a correctness disaster (financial, regulatory), not as a default.

**How to decide**

Ask what actually produces a duplicate in your system. IDs only from the database sequence → option 1 is enough and everything else is over-engineering. External systems supply IDs (ERP feed, partner API, migration) → option 1 gives you nothing; use 2, 3 or 4 depending on whether you can move the partition key.

**The real choice:** you can have an enforced global unique key, **or** date-range partitioning, but Postgres won't give you both on the same column. Oracle's global indexes exist to sell you both, and the price there is that dropping a partition stops being cheap.

**The foreign key consequence**

Any table pointing at `orders` must reference the full declared key:

```sql
CREATE TABLE order_lines (
    order_id    bigint NOT NULL,
    order_date  date   NOT NULL,   -- carried purely to satisfy the FK
    ...
    FOREIGN KEY (order_id, order_date) REFERENCES orders (order_id, order_date)
);
```

`order_date` is now duplicated into every child table and every join carries two columns. A genuine modelling cost, and a common reason teams either partition by a key that already appears in child tables, or decide the table isn't big enough to be worth partitioning yet.

## 13. DynamoDB

**Q: Is there anything special about partitioning in DynamoDB?**

In Postgres, partitioning is an optimisation you add later. In DynamoDB it **is the data model**, it's mandatory, and it's permanent.

**The mechanism**

Every item has a **partition key** (a.k.a. hash key). DynamoDB hashes it, and the hash decides which physical partition holds the item. No range partitioning, no list partitioning, no choice of strategy.

An optional **sort key** orders items *within* a partition. Together they form the primary key and the pair must be unique — the same composite-key situation as partitioned Postgres, except with no alternative and no sequence to fall back on.

**Q: By "item" do you mean the thing being saved? Who decides which element is the partition key?**

**Item = row.** The vocabulary maps: table → table, **item** → row, **attribute** → column.

**You decide the partition key once, at table creation.** Not per-write:

```
aws dynamodb create-table \
  --table-name AppData \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE
```

`KeyType=HASH` is the partition key, `KeyType=RANGE` the sort key. Fixed for the life of the table.

After that, **every item must supply a value for `PK`** — DynamoDB rejects the write without it. The writer doesn't choose *which attribute* is the key; they supply a *value* for the attribute already designated.

**Two different decisions, easy to conflate:**

- Table designer decides **which attribute is the partition key** — once, permanent.
- The writer decides **what value that attribute has** — per item, every write. (Same as choosing what goes in a Postgres column.)

**Worked example**

```json
{ "PK": "CUSTOMER#42", "SK": "PROFILE",               "name": "Anna",  "email": "a@x.com" }
{ "PK": "CUSTOMER#42", "SK": "ORDER#2026-01-15#1001", "amount": 50,   "status": "SHIPPED" }
{ "PK": "CUSTOMER#43", "SK": "PROFILE",               "name": "Ben",   "email": "b@x.com" }
{ "PK": "CUSTOMER#99", "SK": "PROFILE",               "name": "Chloe", "email": "c@x.com" }
```

Items have **different attributes** — the profile has `email`, the order has `amount` and `status`. Allowed: DynamoDB is schemaless apart from the key.

The two `CUSTOMER#42` items land in the **same partition** — same input to the hash, same output, always. So:

- `Query: PK = "CUSTOMER#42"` → profile and all orders, already sorted by SK. One fast read of one partition.
- "Give me Anna and Ben" → two separate lookups, two partitions.
- "Give me all customers" → `Scan`, reads the whole table.
- "Customers in the EU" → `Scan` + filter, or a GSI on region.

**Why `PK` and not `customerId`?** In single-table design the same attribute holds `CUSTOMER#42` on one item and `PRODUCT#77` on another; a generic name is honest about that. If your table only holds one entity type, name it `customerId` — perfectly normal.

**The application builds the key value.** `"CUSTOMER#" + id` is string concatenation in your code. DynamoDB doesn't know `CUSTOMER#` means anything; it just hashes the string. The `#` separator is convention, chosen because it sorts low and rarely appears in real data.

**The thing with no Postgres equivalent**

In Postgres, a query without the partition key is *slower* — it scans all partitions and still works. In DynamoDB, a query without the partition key **is not a query**. It's a `Scan`: reads the entire table, filters afterwards, bills you for everything it read. There's no "add an index later" escape — a GSI creates a whole second copy of the data with its own partition key, and you pay for the storage and the writes forever.

This is why "access patterns first" is *advice* in Postgres and a *hard requirement* here.

**Physical limits that bite**

- **10GB per partition** for a single item collection (all items sharing one partition key, when an LSI exists). Exceed it and writes fail.
- **3,000 RCU / 1,000 WCU per partition.** Your table may be provisioned for 100,000 WCU, but if every write targets one partition key you're capped at 1,000 and throttled while 99% of capacity sits unused.

That's a **hot partition**. `status = 'PENDING'` as a partition key puts every pending order on one machine. `date = '2026-09-16'` puts today's entire write volume on one partition — the timestamp mistake in its most punishing form.

**Adaptive capacity** reallocates throughput toward busy partitions and can split a hot partition. It softens the problem but doesn't remove it — a single hot *key* (one viral product) can't be split, because it's one key.

Standard workaround: **write sharding** — append a suffix, `PENDING#0` … `PENDING#9`, spreading writes across ten partitions. Reads query all ten and merge. Read cost traded for write throughput, deliberately.

**Single-table design**

| PK | SK | attributes |
| --- | --- | --- |
| `CUSTOMER#42` | `PROFILE` | name, email |
| `CUSTOMER#42` | `ORDER#2026-01-15#1001` | amount, status |
| `CUSTOMER#42` | `ORDER#2026-02-20#1002` | amount, status |
| `CUSTOMER#43` | `PROFILE` | name, email |

"Get customer 42 and their recent orders" is **one query on one partition**, because those items are physically adjacent on disk. The sort key prefix `ORDER#` plus `begins_with` gives just the orders; the date in the sort key gives ordering and range queries for free.

That's the actual reason for single-table design: DynamoDB has no joins, so you co-locate related items in the same partition and let the sort key do what a join would have done. Denormalisation taken to its logical end — and it only works because you enumerated the access patterns first.

**Indexes**

- **LSI** (Local Secondary Index) — same partition key, different sort key. Must be created with the table, can never be added. Brings the 10GB item-collection limit into play.
- **GSI** (Global Secondary Index) — different partition key entirely. Can be added anytime. A separate replicated copy, **eventually consistent**, with its own throughput. Five GSIs means each write costs six writes.

**Q: When would I use DynamoDB vs Postgres? I thought it was when I don't need atomicity and easy reads.**

Two corrections:

- **DynamoDB does have atomicity.** Every single-item write is atomic and strongly consistent. Conditional writes (`attribute_not_exists`, `quantity > 0`) are linearizable on that item — exactly the inventory-decrement pattern. `TransactWriteItems` gives real ACID across up to 100 items. Single-key atomic operations are a *strength*.
- **Reads aren't "easier" — they're narrower.** Reads matching your key design are faster and more predictable than Postgres. Reads that don't match are effectively impossible. Not a weaker SQL; a different shape.

**The actual question:** *Do I know all my access patterns in advance, and will they stay stable?*

DynamoDB trades **query flexibility** for **predictable latency and unbounded horizontal scale**. If you can't enumerate your queries up front, you pay that price and get nothing back.

The deal, plainly:

- **DynamoDB:** "Tell me exactly which questions you'll ask. Those are instant forever, at any size. But I'll only answer *those* questions."
- **Postgres:** "Ask me anything, whenever. Some answers will be slower, and past a certain size I'll struggle."

**Concrete version.** Order system on DynamoDB, PK `CUSTOMER#42`. "Get a customer's orders" is 5ms at a billion orders. Six months later: *"How many orders shipped last Tuesday?"*

- Postgres: write a query, maybe add an index. An afternoon.
- DynamoDB: that question doesn't exist in your key design. Scan the whole table, or build a GSI keyed on ship date — a second copy of your data, an extra write on every order, permanently.

**The sting:** if your system does 200 orders a minute, Postgres was never going to break a sweat. You never needed the scale, so you accepted the rigidity and received nothing for it. The cost is paid up front and feels free; the benefit only materialises at a scale most systems never reach.

**Choose DynamoDB when most are true**

- Access is by a known key (session by token, user's cart, device readings for last hour)
- Consistent single-digit-ms latency at any scale, not just average-case
- Genuinely large write volume spreading naturally across many keys
- You want zero operational work — no vacuum, failover, connection pool tuning, version upgrades
- Spiky unpredictable traffic where on-demand billing beats provisioning for peak
- The data has no interesting relationships to traverse

Good fits: session stores, shopping carts, user preferences, IoT/device state, feature flags, idempotency keys, leaderboards — anything shaped like a big distributed hash map.

**Choose Postgres when**

- You'll ask questions you haven't thought of yet (this one reason covers most business systems)
- Data has relationships worth joining across
- You need real multi-row constraints: unique email, referential integrity, check constraints. DynamoDB has **none** of these — every invariant lives in application code
- Aggregation matters (`SUM`, `GROUP BY`, window functions)
- Your volume fits comfortably on one machine — true far more often than people assume

**On "schemaless" and "document-based"**

Half right. Items can have different attributes, so schemaless in that sense. But it's a **wide-column / key-value** store, not a document database like MongoDB. You can store nested JSON but can't index into it or query nested fields efficiently. Don't pick DynamoDB because you like documents — Postgres `JSONB` gives flexible documents *with* indexes, joins and SQL.

**Recommendation:** default to Postgres. An unanticipated query costs you some SQL there; in DynamoDB it's a GSI you pay for forever, or a table migration. Reach for DynamoDB for a *specific* workload with a known key pattern and either real scale or a hard latency requirement. Running both is completely normal — Postgres for orders and customers, DynamoDB for sessions and device state.

**The tell:** if a product manager will eventually ask "can we see this broken down by region?", you want Postgres.

*Pricing note: on-demand mode removed most of the old capacity-planning pain, and AWS cut on-demand prices substantially in late 2024. Check current pricing before modelling costs.*

## 14. Sharding — Deep Dive

**The one difference that creates every other difference**

Partitioning splits data across tables on one machine. Sharding splits it across **separate database servers that don't know about each other**.

The moment you cross that line, the database stops being able to help you: no joins across shards, no transaction across shards, no unique constraint across shards, no `ORDER BY` across shards. Every one becomes your application's problem.

You're not buying a feature — you're giving up features to get write scaling.

**What actually forces sharding**

- **Write throughput** exceeds what one primary can absorb.
- **Dataset size** exceeds what one machine can hold or back up in a reasonable window.
- **Regulatory data residency** — EU customer data must physically live in the EU. Common, and a legitimate reason at modest volume.
- **Blast radius** — you don't want one bad query taking down all customers.

"Our database is slow" isn't on the list. That's usually missing indexes, N+1 queries, or a query that should never have hit the primary.

**Strategies**

- **Range sharding** — customers A–F on shard 1, G–M on shard 2. Range queries work naturally; rebalance by moving boundary points. Problem: skew. Surnames aren't uniform, and any monotonically increasing key (timestamps, sequential IDs) sends every new write to the last shard.
- **Hash sharding** — `hash(customer_id) % N`. Excellent spread. Fatal flaw: the `% N` — change N and almost every key maps somewhere new, meaning a full reshuffle to add one server.
- **Directory / lookup-based** — a lookup table says which shard holds which key. Total flexibility (move one noisy customer onto their own shard). Cost: a lookup on every request, and a component that must never go down. Common in multi-tenant SaaS, where "tenant 4711 lives on shard 3" is a natural row in a control database.

**Consistent hashing and virtual buckets — how rebalancing is actually done**

The fix for `% N`: stop mapping keys to shards directly. Map keys to a large fixed number of **virtual buckets** (1024, 4096 — chosen once, never changed), then map buckets to physical shards. `hash(key) % 1024` never changes; only the bucket→shard assignment changes.

```
Before (3 shards):  A A A A  B B B B  C C C C
After adding D:     A A A D  B B B D  C C C D
```

Only 3 of 12 buckets move. Buckets are the unit of movement — you move whole buckets, never individual rows, and a key never changes bucket. Adding a shard moves roughly 1/N of the data instead of all of it.

**Consistent hashing** (the ring, used by Cassandra and DynamoDB internally) achieves the same property differently: nodes and keys are both placed on a circle, and a key belongs to the next node clockwise, so adding a node only steals from its neighbour. Virtual nodes are added because a plain ring distributes unevenly with few nodes.

**Routing — who decides which shard to talk to**

- **Client-side** — the app computes the shard and connects directly. Fastest, but every service needs the routing logic and config, and changing the map means redeploying everything.
- **Proxy** — a middle layer speaks the database protocol and routes (Vitess, ProxySQL). Your app thinks it's talking to one database. Costs a network hop and a component to run.
- **Database-native** — Citus, MongoDB `mongos`, CockroachDB, Spanner handle it internally. Least work, most lock-in.

**What you actually lose**

- **Cross-shard joins.** `orders JOIN customers` works only if both rows are on the same shard. Mitigations: **co-location** (shard both tables by `customer_id`, so a customer's orders sit with the customer — this is why the shard key is almost always a tenant or customer ID) and **reference tables** (replicate small lookup tables like `countries` to every shard).
- **Cross-shard transactions.** Gone, unless you adopt two-phase commit, which is slow with underestimated failure modes. In practice you restructure so transactions stay within a shard, or use a **saga** — a sequence of local transactions with compensating actions when a step fails.
- **Global unique constraints.** `UNIQUE (email)` can't be enforced across eight machines. You need a registry table, or IDs that can't collide — UUIDv7, Snowflake, or per-shard sequence offsets (shard 1 issues 1, 9, 17…; shard 2 issues 2, 10, 18…).
- **Auto-increment IDs.** Each shard's sequence starts at 1; two shards both produce order 1. Decide this *before* you shard.
- **Aggregations and pagination.** `SUM(amount)` means querying all shards and summing in your app — a scatter-gather, as slow as the slowest shard. `ORDER BY date LIMIT 20` is worse: fetch 20 from every shard, merge, discard. Offset pagination deep into results becomes impractical; cursor pagination is the workaround.
- **Schema migrations.** `ALTER TABLE` must run on every shard, and shards can drift if one fails. Tooling is mandatory, not optional.

**Resharding a live system**

1. Stand up the new shard.
2. **Backfill** the buckets being moved while traffic continues.
3. **Double-write** — writes go to both old and new location during the transition.
4. **Verify** the two sides agree (a reconciliation job comparing checksums).
5. Cut reads over, bucket by bucket, with the ability to flip back.
6. Stop double-writing, drop the old data.

People underestimate step 4. Without verification you find out about drift from a customer, months later. Vitess and Citus automate much of this; rolling your own is weeks of work with subtle failure modes.

**Real systems**

- **Vitess** — MySQL sharding, built at YouTube, runs Slack and Square. Most battle-tested.
- **Citus** — turns Postgres into a sharded cluster (now part of Azure). Pick a distribution column; it handles co-location and reference tables. Best fit if you're already on Postgres and multi-tenant.
- **MongoDB** — native sharding with a chosen shard key, `mongos` routers, automatic chunk balancing.
- **CockroachDB / Spanner / YugabyteDB** — sharded from the ground up, and they *do* give you distributed transactions, at the cost of higher write latency (consensus per write) and a different operational model.
- **Cassandra / DynamoDB** — sharding isn't a feature, it's the architecture. You never choose to shard; you chose it when you picked the partition key.

**Exhaust these before sharding** (cheapest first — one usually suffices)

1. Fix the queries and indexes. Measure first.
2. Vertical scaling. A 128-core machine with 2TB RAM is real and cheaper than an engineering team.
3. Read replicas for read load.
4. Caching for hot reads.
5. Move analytics off the primary to a warehouse.
6. Partition within one database — reclaims vacuum and index costs.
7. **Functional split** — move a whole subsystem (billing, notifications) to its own database. Much easier than sharding, and often enough.
8. Archive cold data out of the hot tables.

**Picking the shard key**

Same criteria as a partition key with higher stakes: high cardinality, even distribution, present in nearly every query, and **stable** (a key that changes means moving the row between machines).

For most business systems the answer is `customer_id` or `tenant_id`, because it co-locates everything one customer touches so almost every query stays single-shard. Failure case: a tenant so large it doesn't fit on one shard — then give them a dedicated shard (where the directory approach earns its keep) or sub-shard within them.

**Honest summary:** sharding converts a database problem into an application problem. You get write scaling and blast-radius isolation; you pay in joins, transactions, constraints, migrations and operational complexity — permanently, not just during the migration.

## 15. Replicas vs Shards, and CQRS on Top

**Q: How does sharding help with write throughput? Why don't replicas?**

**Replicas copy writes. Shards divide writes.**

A read replica is a full copy of the database. For it to stay a full copy, every write to the primary must also be applied on the replica:

- Primary handles 10,000 writes/sec
- Replica 1 also applies all 10,000
- Replica 2 also applies all 10,000

Adding replicas doesn't reduce anyone's write load by a single row. Every machine still does 100% of the writes. You've added *read* capacity — reads can be split across copies because reading doesn't change anything — but the write ceiling is unchanged. It actually gets slightly worse: the primary now ships WAL to three replicas, and with synchronous replication it waits for acknowledgements on every commit.

With 4 shards, each is a *different* database holding a *different* quarter of the data:

- Shard A: customers 1–250k, \~2,500 writes/sec
- Shard B: customers 250k–500k, \~2,500 writes/sec
- Shards C and D: the same

Each machine does a quarter of the work. Adding a fifth shard raises the ceiling again. That's what **horizontal write scaling** means — capacity grows with machine count.

**The asymmetry in one line:** reads can be duplicated, writes must be divided.

**They compose**

```
Shard A: primary + 2 replicas   (2,500 writes/sec, reads spread over 3)
Shard B: primary + 2 replicas
Shard C: primary + 2 replicas
Shard D: primary + 2 replicas
```

Twelve machines. Write capacity 4x a single primary (the shards). Read capacity 12x (every copy serves reads).

This is why the framework asks for the read:write **ratio** separately from absolute volume. A 1000:1 ratio means you almost certainly need replicas and almost certainly don't need shards — even at high total traffic, the write component stays small. Write-heavy at 1:1 is what pushes you toward sharding, because there's no copying trick available.

**Q: So for a write-heavy system, sharding for writes plus replicas / CQRS views for reads?**

Almost — but there are **two different read problems** in there, and conflating them is where designs go wrong.

| Problem | Solved by | Why |
| --- | --- | --- |
| Read volume *within* a shard | Replicas | Shard B's replicas hold shard B's data, so they serve any read that includes the shard key |
| Reads that don't fit the shard key | CQRS projection | Replicas do **nothing** here — a replica of shard B still only knows shard B |

If you sharded by `customer_id` and someone asks "all orders shipped last Tuesday", every shard holds part of the answer. You're scatter-gathering across all of them, and replicas just give you more copies of one quarter of the data. *That* second problem is why CQRS belongs in this architecture.

```
Writes ──routed by key──> Shard A (primary + replicas) ──┐
                          Shard B (primary + replicas) ──┼──CDC──> Read projection
                          Shard C (primary + replicas) ──┘         (cross-shard reads)
```

**How the projection gets built**

Not by your application writing twice. You tap each shard's replication log — Postgres logical decoding via Debezium, MySQL binlog, DynamoDB Streams — and publish changes to Kafka. A consumer merges the streams from all shards and writes into whatever store answers the query well:

- **Elasticsearch** for search and filtering
- **A columnar warehouse** (BigQuery, ClickHouse) for aggregation and reporting
- **A denormalised Postgres table** keyed the way that query needs
- **Redis** for a precomputed leaderboard or counter

The key property: the projection is keyed for the **read**, not for the write. That's the point — a second organisation of the same data, in a separate store.

**Why CDC rather than dual writes from the app:** dual writes have no atomicity. The database commit succeeds, the Kafka publish fails, and the two are permanently out of sync with nothing to detect it. CDC reads the committed log, so it can't disagree with what was committed, and it replays from an offset after a failure.

**The consistency caveat**

The projection is eventually consistent — typically seconds behind, occasionally minutes if a consumer lags. So the routing rule from section 6 applies directly:

- **Linearizable decisions** → shard primary, in a transaction. Never the projection, never a replica.
- **Reads including the shard key, tolerating a little staleness** → shard replicas.
- **Cross-shard, analytical, search, reporting** → projection.

Common bug: someone points a checkout-path read at the projection because it's convenient and fast. It's fast because it's stale.

**One correction to the phrasing**

"Read replicas **or** read views" — it's usually both, and they're not alternatives. Replicas are copies of the write model with the same keying. Projections are a different model with different keying. You need replicas for volume and projections for shape.

**The caveat worth keeping**

CQRS is not a default. It's justified when your read patterns genuinely don't fit your write partitioning — *usually* true once you've sharded, since sharding forces one key and real systems have several access patterns.

But it costs: another pipeline to operate, another store to back up, replication lag to monitor, and the hardest part — **reconciliation**. When the projection drifts from the shards (a consumer bug, a poison message, a schema change), you need a way to detect it and rebuild. **Build the rebuild path early**; you'll need it, and you don't want to be writing it during an incident.
