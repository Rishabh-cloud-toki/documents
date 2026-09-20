# Data Architecture

Architect-level reference on how to store, model, replicate, partition, evolve,
and cache data in a distributed system. Companion to the unanswered question list
in [database-and-data-architecture-questions.md](database-and-data-architecture-questions.md),
to [../architecture/cap-theorem.md](../architecture/cap-theorem.md), and to the
deep-dive Q&A in [data-architecture-notes.md](../notes/data-architecture-notes.md).

## Contents

- [OLTP vs OLAP vs streaming](#oltp-vs-olap-vs-streaming)
- [Choosing a datastore — a decision framework](#choosing-a-datastore--a-decision-framework)
- [The datastore families](#the-datastore-families)
- [Data modeling](#data-modeling)
- [Indexing and query performance](#indexing-and-query-performance)
- [Transactions and isolation](#transactions-and-isolation)
- [Replication](#replication)
- [Partitioning and sharding](#partitioning-and-sharding)
- [Distributed transactions across stores](#distributed-transactions-across-stores)
- [Caching](#caching)
- [CQRS and Event Sourcing](#cqrs-and-event-sourcing)
- [Change Data Capture (CDC)](#change-data-capture-cdc)
- [Schema evolution and zero-downtime migrations](#schema-evolution-and-zero-downtime-migrations)
- [Multi-tenancy data patterns](#multi-tenancy-data-patterns)
- [Analytical data: warehouse, lake, lakehouse](#analytical-data-warehouse-lake-lakehouse)
- [Polyglot persistence and data mesh](#polyglot-persistence-and-data-mesh)
- [Backup, retention, and data governance](#backup-retention-and-data-governance)
- [Architect checklist](#architect-checklist)
- [Appendix: Q&A deep-dives](#appendix-qa-deep-dives)

---

## OLTP vs OLAP vs streaming

> Deep-dive: [OLTP, OLAP and streaming, and why storage layout follows from purpose](#deep-dive-oltp-olap-and-streaming-and-why-storage-layout-follows-from-purpose)
> — the cash-register / monthly-notebook framing, why row vs columnar is a
> one-dimensional-disk problem, and Parquet's row-group internals.

| | OLTP | OLAP | Streaming / real-time analytics |
|---|---|---|---|
| Purpose | Run the business — reads/writes of individual records | Analyse the business — aggregate over history | React to events as they arrive |
| Access pattern | Point lookups, small transactions, high concurrency | Large scans, GROUP BY, joins over big tables | Continuous queries over windows |
| Latency | ms | seconds–minutes | ms–seconds |
| Data volume per query | Rows | Millions–billions of rows | Bounded by window |
| Store | PostgreSQL, MySQL, Spanner, DynamoDB, MongoDB | BigQuery, Snowflake, Redshift, ClickHouse | Kafka Streams, Flink, Materialize, Druid, Pinot |
| Schema | Normalised | Star / snowflake, denormalised, columnar | Event schema |

**Rule:** never run analytics on your transactional store. Move data out via
[CDC](#change-data-capture-cdc) or an event stream into a warehouse. The
operational store stays lean and predictable.

---

## Choosing a datastore — a decision framework

> Deep-dive: [choosing a datastore, worked](#deep-dive-choosing-a-datastore-worked)
> — why the question order works (correctness, then scale, then you), a retailer
> example carried through all seven questions, and the DynamoDB-vs-Postgres tell.

Ask these in order. The answers usually eliminate all but one or two options.

1. **What are the access patterns?** Write them all down as concrete queries
   ("get order by id", "list orders for a customer sorted by date", "sum revenue
   by region by month"). Model the data to serve *these*, not an abstract entity
   diagram.
2. **What consistency does each operation need?** Linearizable (a balance, an
   inventory count, a unique username) vs eventual (a feed, a product page, a
   recommendation). Per-operation, not per-system — see
   [cap-theorem.md](../architecture/cap-theorem.md).
3. **What is the read:write ratio and absolute volume?** 1000:1 reads → cache +
   replicas. Write-heavy at scale → LSM-based store, partitioning from day one.
4. **How structured and how variable is the data?** Fixed relational schema,
   flexible documents, a graph of relationships, append-only time series, or
   full-text?
5. **What are the query needs?** Ad-hoc joins and aggregation → relational.
   Known key-based access → KV / wide-column. Relationship traversal → graph.
   Search/ranking → search engine.
6. **Scale ceiling and growth?** Will a single primary with read replicas carry
   you for 3 years, or do you need horizontal write scaling now?
7. **Operational reality:** team skills, managed-service availability, backup and
   DR story, cost model (per-request vs provisioned vs node-hours).

**Default:** start with a well-run relational database (PostgreSQL). It does OLTP,
JSON documents, full-text, geospatial, and moderate analytics. Add specialised
stores only when a measured access pattern demands it.

---

## The datastore families

> Deep-dives: [DynamoDB, partitioning as the data model](#deep-dive-dynamodb-partitioning-as-the-data-model)
> — item/attribute vocabulary, single-table design, hot-partition limits, and
> why DynamoDB trades query flexibility for guaranteed latency at any scale;
> [why relational gives up write scaling](#deep-dive-why-relational-gives-up-write-scaling)
> — partitioning doesn't break queries, sharding does, and why that's specific
> to relational's cross-row guarantees;
> [four families compared, and the locality principle](#deep-dive-four-families-compared-and-the-locality-principle)
> — key-value vs relational vs document vs wide-column side by side, and why
> performance here is really "how many places must the disk visit";
> [document databases, embedding vs referencing](#deep-dive-document-databases-embedding-vs-referencing)
> — the bounded/unbounded rule, and where MongoDB's wide-column impersonation
> does and doesn't work;
> [wide-column and Cassandra, query-first modelling](#deep-dive-wide-column-and-cassandra-query-first-modelling)
> — the no-boss ring, why a Cassandra table is a stored answer to one query,
> and the multi-table write problem.

| Family | Model | Strengths | Weak at | Examples |
|---|---|---|---|---|
| **Relational** | Tables, rows, SQL, joins, ACID | Flexible queries, strong consistency, mature tooling | Horizontal write scaling, schema churn at scale | PostgreSQL, MySQL, SQL Server |
| **Distributed SQL ("NewSQL")** | Relational + horizontal scale + strong consistency | Scale *and* ACID, global tables | Cost, latency of cross-region consensus | Google Spanner, CockroachDB, YugabyteDB, TiDB |
| **Key-value** | `key → opaque value` | Fastest possible lookups, simple scaling, caching | Any query that isn't "by key" | Redis, DynamoDB, Riak, Aerospike |
| **Document** | JSON-like documents, secondary indexes | Flexible schema, aggregates that match one document | Cross-document transactions, joins | MongoDB, Couchbase, Firestore |
| **Wide-column** | Rows with dynamic columns, partition + clustering keys | Massive write throughput, predictable latency at scale, time series | Ad-hoc queries, joins; you design the table per query | Cassandra, ScyllaDB, HBase, Bigtable |
| **Graph** | Nodes and edges | Multi-hop relationship traversal, recommendations, fraud rings | Bulk analytics, high write volume | Neo4j, Neptune, JanusGraph |
| **Time series** | Timestamp-indexed metrics/events | Compression, downsampling, retention, range queries | Non-time-based access | InfluxDB, TimescaleDB, Prometheus |
| **Search** | Inverted index, relevance scoring | Full-text, faceting, fuzzy, aggregations | Source of truth (keep the real data elsewhere) | Elasticsearch / OpenSearch, Solr |
| **Analytical (columnar)** | Column-oriented, MPP | Scans and aggregations over billions of rows | Point writes, single-row updates | BigQuery, Snowflake, Redshift, ClickHouse |

"NoSQL = AP / eventually consistent" is outdated — most are tunable (Cassandra
per-query consistency levels; MongoDB is CP by default; DynamoDB offers both
eventually- and strongly-consistent reads).

---

## Data modeling

> Deep-dives: [access patterns first, not entities](#deep-dive-access-patterns-first-not-entities)
> — what a written-down access pattern actually decides (key, sort order,
> denormalisation), and why entity-first modeling answers every query adequately
> and none of them well;
> [four families compared, and the locality principle](#deep-dive-four-families-compared-and-the-locality-principle)
> — why locality, not schema flexibility, is what makes a document read cheap.

### Relational: normalise, then denormalise deliberately

- **Normalisation (up to 3NF)** removes redundancy → one fact in one place → no
  update anomalies. Default for OLTP.
- **Denormalise** only when a measured read path needs it: duplicate a column to
  avoid a hot join, keep a materialised count, precompute a rollup. Every
  denormalised copy is now a consistency obligation — document how it's kept in
  sync (trigger, application code, CDC, scheduled job).

### NoSQL: model from the access patterns

There is no "correct" schema independent of queries. Process:

1. List every query and its frequency and latency target.
2. Design one table/collection per query group so each query is a single
   partition read.
3. Choose the **partition key** for even distribution and query locality; choose
   **clustering/sort keys** for the ordering the query needs.
4. Duplicate data across tables as needed; accept that writes fan out.
5. Handle the write fan-out with batch writes, transactions (if supported), or an
   async updater fed by [CDC](#change-data-capture-cdc).

### Building blocks (also see the DDD note)

- **Entity** — identity + lifecycle (`Order`).
- **Value object** — immutable, no identity (`Money`, `Address`).
- **Aggregate** — consistency boundary; load and save as a unit; reference other
  aggregates by id, not by object.
- Keep aggregates small — large aggregates cause contention and long transactions.

---

## Indexing and query performance

> Deep-dive: [write-heavy patterns and why LSM trees exist](#deep-dive-write-heavy-patterns-and-why-lsm-trees-exist)
> — the seven-step ladder for write-heavy systems, and the LSM internals
> (memtable, SSTable, compaction, bloom filters, the three amplifications)
> behind the B-tree vs LSM row below.

- **B-tree index** (default in relational DBs): `O(log n)` lookups, supports
  range scans and ordering, kept sorted on write. Cost: every write updates every
  index on the table.
- **LSM-tree** (Cassandra, RocksDB, modern KV): writes go to an in-memory
  memtable + append-only log, flushed to immutable SSTables, merged by
  compaction. Great write throughput; reads may touch several SSTables (mitigated
  by bloom filters); compaction competes for I/O.
- **Composite index** `(a, b, c)`: usable for predicates on a leftmost prefix
  (`a`, `a+b`, `a+b+c`) — not for `b` alone. Order the columns by
  equality-before-range.
- **Covering index**: includes every column the query needs, so the engine never
  touches the table ("index-only scan").
- **Partial / filtered index**: index only rows matching a predicate (e.g.
  `WHERE status = 'ACTIVE'`) — smaller, cheaper.
- **Hash index**: `O(1)` equality only, no ranges.
- **Inverted index**: term → list of documents, for full-text search.

### Diagnosing slow queries

1. `EXPLAIN ANALYZE` — read the plan. Look for sequential scans on big tables,
   nested-loop joins over large inputs, sort/hash spills to disk, and a large gap
   between estimated and actual rows (stale statistics).
2. Add or fix an index; rewrite the query; update statistics.
3. Watch for **N+1 queries** from ORMs — one query per row instead of a join or
   batch fetch.
4. Connection pool exhaustion looks like slowness — size the pool to the DB's
   real concurrency limit, not "as high as possible".
5. Beware unbounded result sets — always paginate (keyset/seek pagination beats
   `OFFSET` at depth).

---

## Transactions and isolation

> Deep-dive: [consistency per operation, and making stale reads safe](#deep-dive-consistency-per-operation-and-making-stale-reads-safe)
> — why the same application routes inventory, a product page, and a
> recommendation feed down three different consistency paths, and how to fix a
> stale read at the write, not the read.

**ACID**: Atomicity (all-or-nothing), Consistency (invariants preserved),
Isolation (concurrent transactions don't corrupt each other), Durability
(committed data survives a crash).

### Isolation levels and the anomalies they permit

| Level | Dirty read | Non-repeatable read | Phantom read | Write skew / lost update |
|---|---|---|---|---|
| Read Uncommitted | possible | possible | possible | possible |
| Read Committed *(common default)* | prevented | possible | possible | possible |
| Repeatable Read / Snapshot | prevented | prevented | possible* | possible (write skew) |
| Serializable | prevented | prevented | prevented | prevented |

\* PostgreSQL's Repeatable Read (snapshot isolation) also prevents phantoms.

- **Dirty read** — see another transaction's uncommitted write.
- **Non-repeatable read** — re-reading a row returns a different value.
- **Phantom** — a range query returns different rows on re-execution.
- **Lost update** — two read-modify-write cycles, one overwrites the other. Fix
  with `SELECT ... FOR UPDATE`, atomic operations, or optimistic locking.
- **Write skew** — two transactions read an overlapping set, each makes a
  decision valid alone but not together (both on-call doctors go off shift).
  Needs Serializable, or an explicit lock/materialised conflict row.

### Concurrency control mechanisms

- **Pessimistic locking** — acquire locks up front (`FOR UPDATE`). Simple,
  contention-prone, deadlock risk.
- **Optimistic locking** — a `version` column; on update, `WHERE version = :v`;
  zero rows updated → someone else won → retry. Good for low-contention.
- **MVCC** — readers see a consistent snapshot without blocking writers; each
  write creates a new row version. Old versions are cleaned up (Postgres
  `VACUUM`) — long-running transactions block cleanup and bloat the table.

---

## Replication

> Deep-dive: [replicas vs shards, and CQRS on top](#deep-dive-replicas-vs-shards-and-cqrs-on-top)
> — "replicas copy writes, shards divide writes," why adding a replica never
> raises the write ceiling, and how the two combine with a CDC-fed projection.

Copying data to multiple nodes for availability, read scaling, and locality.

### Topologies

| Topology | How | Use / caution |
|---|---|---|
| **Single leader** (primary/replica) | All writes to the leader; replicas stream the log | Default. Read scaling via replicas. Failover promotes a replica. |
| **Multi-leader** | Writes accepted at several leaders, replicated both ways | Multi-region write locality, offline clients. **Write conflicts** must be resolved (LWW, CRDTs, app merge). |
| **Leaderless** (Dynamo-style) | Client writes to N replicas, reads from several, quorum decides | High availability; tunable consistency with `R + W > N`; needs read-repair and anti-entropy. |

### Sync vs async replication

- **Synchronous** — leader waits for replica ack before confirming. No data loss
  on leader failure; higher write latency; a stalled replica blocks writes.
- **Asynchronous** — leader confirms immediately. Fast; **data loss window** if
  the leader dies before replicas catch up.
- **Semi-synchronous** — wait for at least one replica. Common compromise.

### Replication lag consequences (and fixes)

- **Read-your-own-writes** — user updates profile, then reads a stale replica.
  Fix: read from the leader for a short window after a write, or route the user's
  session to one replica, or read from leader for that user's own data.
- **Monotonic reads** — successive reads must not go backwards in time. Fix:
  pin a user to one replica.
- **Consistent prefix reads** — causally ordered writes must be seen in order.

### Failover hazards

Split brain (two leaders), lost writes on the old leader, cascading failure if a
replica can't handle the full write load, and flapping. Use a consensus-based
controller and fencing.

---

## Partitioning and sharding

> Deep-dives: [partitioning vs sharding, and the partition-key/unique-constraint
> trap](#deep-dive-partitioning-vs-sharding-and-the-partition-keyunique-constraint-trap)
> — the toy-box version, Postgres range partitioning and pruning, and why a
> partition key in the unique key is a routing guarantee, not a global check;
> [sharding, what it actually costs, and picking a shard key](#deep-dive-sharding-what-it-actually-costs-and-picking-a-shard-key)
> — what genuinely forces sharding, consistent hashing, everything you give up,
> and the ladder to exhaust first;
> [wide-column and Cassandra, query-first modelling](#deep-dive-wide-column-and-cassandra-query-first-modelling)
> — what it looks like to shard from day one, with no leader and no cross-partition queries.

Splitting one dataset across nodes so writes and storage scale horizontally.
(Partitioning = within a store; sharding = across stores/nodes; often used
interchangeably.)

### Partitioning strategies

| Strategy | Pros | Cons |
|---|---|---|
| **Key range** | Efficient range scans; keys stay ordered | Hotspots on sequential keys (timestamps, auto-increment ids) |
| **Hash of key** | Even distribution | Range queries hit every partition ("scatter/gather") |
| **Consistent hashing** | Adding/removing a node moves ~`1/N` of keys, not everything; virtual nodes smooth distribution | Still needs care for range queries |
| **Directory / lookup** | Full control, easy rebalancing | The lookup service is a dependency and a bottleneck |
| **Entity-group / by tenant** | Transactions and joins stay within a partition | Uneven tenant sizes → "noisy neighbour" |

### Hotspot avoidance

- Don't partition on monotonically increasing keys — prefix with a hash bucket,
  or use a random/UUID component.
- Split known-hot keys (a celebrity user) across sub-partitions with a salt.

### Secondary indexes on a partitioned store

- **Local index** — each partition indexes its own data; writes are cheap; reads
  must scatter/gather across all partitions.
- **Global index** — the index itself is partitioned by the indexed value; reads
  hit one partition; writes are cross-partition and often async (eventually
  consistent index).

### Rebalancing

- Fixed number of partitions >> nodes; move whole partitions between nodes.
- Avoid "hash mod N" — changing N reshuffles everything.
- Rebalance slowly, throttled, and never automatically in response to a transient
  spike.

---

## Distributed transactions across stores

> The [sharding deep-dive](#deep-dive-sharding-what-it-actually-costs-and-picking-a-shard-key)
> covers why cross-shard transactions disappear once you shard, and why sagas
> are usually the replacement, not 2PC.

Preferred order of approaches (see also the Microservices note's saga/outbox
sections):

1. **Don't.** Redesign so one service/aggregate owns the write. A distributed
   transaction is a design smell more often than a requirement.
2. **Saga** — a sequence of local transactions, each with a compensating action.
   *Choreography* (services react to events) or *orchestration* (a coordinator
   drives steps and compensations). Eventual consistency; needs idempotency and
   traceability.
3. **Transactional outbox** — write the business row and an `outbox_event` row in
   the same local transaction; a relay (poller or CDC) publishes the event.
   Solves the dual-write problem.
4. **TCC (Try-Confirm/Cancel)** — reserve resources in `Try`, then `Confirm` or
   `Cancel`. More coupling than saga; useful when you need a hold (seat, funds).
5. **2PC / XA** — a coordinator drives prepare then commit across resource
   managers. Strong consistency but blocking, poor availability (coordinator is a
   SPOF), and it doesn't scale. Acceptable only within one datacenter, few
   participants, or during a monolith→microservices transition.

**Idempotency is mandatory** for 2–4: every operation needs an idempotency key
and a dedup check so retries are safe.

---

## Caching

### Placement (cheapest/nearest first)

Client / browser → CDN / edge → API gateway → in-process (Caffeine) → distributed
(Redis / Memcached) → database buffer pool / materialised view.

### Patterns

| Pattern | Read | Write | Notes |
|---|---|---|---|
| **Cache-aside (lazy)** | App checks cache; on miss, loads from DB and populates cache | App writes DB, then invalidates (or updates) cache | Most common. Risk: stale entry if invalidation is missed; brief inconsistency on concurrent write+read. |
| **Read-through** | Cache library loads from DB on miss | — | Cache owns the read path; simpler app code. |
| **Write-through** | — | App writes to cache; cache writes to DB synchronously | Cache always fresh; write latency = cache + DB. |
| **Write-behind (write-back)** | — | App writes to cache; cache flushes to DB asynchronously | Fast writes, absorbs bursts; **data loss risk** if cache dies before flush; ordering complexity. |
| **Refresh-ahead** | Cache proactively reloads hot keys before TTL expiry | — | Hides latency for predictable hot keys; wasted work for cold keys. |

### Invalidation and expiry

- **TTL** — simplest; tolerate staleness up to the TTL. Add jitter so entries
  don't all expire together.
- **Explicit invalidation** on write — precise but easy to miss a path; hard
  across services.
- **Versioned keys** — include a version/etag in the key; old versions age out.
- **Event-driven invalidation** — publish a change event; caches subscribe.

### Eviction policies

LRU (default), LFU, FIFO, TTL-only, or size/weight-based. Redis: `allkeys-lru`,
`volatile-ttl`, etc. Monitor **hit ratio** — a cache below ~80–90% hit rate for
hot data is often mis-sized or mis-keyed.

### Failure modes

- **Cache stampede / thundering herd** — a hot key expires, thousands of requests
  miss simultaneously and hammer the DB. Fixes: per-key lock / single-flight
  (only one loader, others wait), probabilistic early expiration, serve-stale
  while refreshing in the background.
- **Cache penetration** — requests for keys that don't exist bypass the cache
  every time. Fix: cache the negative result (short TTL), or a bloom filter.
- **Cache avalanche** — mass simultaneous expiry or a cache-cluster outage. Fix:
  TTL jitter, request coalescing, a small in-process fallback cache, rate-limit
  DB access.
- **Consistency** — cache and DB will diverge briefly; never cache data that must
  be transactionally correct on read (a bank balance) without care.

---

## CQRS and Event Sourcing

> The [replicas-vs-shards deep-dive](#deep-dive-replicas-vs-shards-and-cqrs-on-top)
> covers when CQRS is justified on top of a sharded write side, and how the
> read projection is actually built (CDC, not dual writes).

### CQRS (Command Query Responsibility Segregation)

Separate the **write model** (commands, validation, domain rules, normalised)
from one or more **read models** (denormalised, shaped per query, possibly in a
different store). A projection process keeps read models updated from writes
(often via events).

- **Use when:** reads and writes have very different shapes or scale; you need
  many specialised read views; complex domain on the write side.
- **Cost:** two models to maintain, eventual consistency between them, projection
  lag and rebuild tooling. Don't apply to CRUD.

### Event Sourcing

Persist state as an **append-only log of events** (`OrderPlaced`, `ItemAdded`,
`OrderShipped`). Current state is a left-fold over the events. Snapshots avoid
replaying from the beginning.

- **Gains:** complete audit trail, temporal queries ("state as of last Tuesday"),
  rebuild any read model by replaying, natural fit for event-driven systems.
- **Costs / hazards:** event schema versioning (upcasters), you can't change
  history, GDPR "right to be forgotten" vs an immutable log (crypto-shredding),
  eventual consistency everywhere, higher conceptual load, tooling immaturity.
- Event Sourcing and CQRS often go together but are independent choices.

---

## Change Data Capture (CDC)

Stream row-level changes out of a database's transaction log (WAL / binlog /
redo) as events, with no change to application code.

- **Tools:** Debezium (Kafka Connect), Maxwell, AWS DMS, Google Datastream.
- **Uses:** replicate to a warehouse/lake, feed search indexes and caches,
  drive the outbox pattern, migrate between databases, build read models.
- **vs dual-write:** CDC captures exactly what committed, in commit order — no
  lost or phantom events.
- **Watch for:** initial snapshot load, schema changes flowing through, ordering
  guarantees (per-table/per-key), PII leaving the DB boundary, connector lag and
  restart/offset management, tombstones for deletes.

---

## Schema evolution and zero-downtime migrations

Old and new application versions run **simultaneously** during a rolling deploy,
so every schema change must be compatible with both.

### Expand–contract (parallel change)

1. **Expand** — additive, backward-compatible change: add the nullable column /
   new table / new index (`CREATE INDEX CONCURRENTLY`). Deploy.
2. **Migrate** — backfill data in batches; dual-write old and new from the app.
3. **Contract** — once all instances use the new shape and backfill is done,
   deploy code that reads only the new column, then drop the old one.

Each step is independently deployable and reversible.

### Rules

- Never rename or drop a column in the same release that stops using it.
- Adding a `NOT NULL` column: add nullable → backfill → add the constraint.
- Big backfills: batch with sleeps, off-peak, monitor replication lag and locks.
- Avoid long-held locks — many DBs rewrite the table for certain `ALTER`s; use
  online-schema-change tools (gh-ost, pt-online-schema-change) for MySQL at
  scale.
- Migration tooling: Flyway or Liquibase, versioned, in the CI/CD pipeline, run
  as a separate step before the app rollout, forward-only.

### Event / API schema

Use a schema registry with a compatibility mode (backward, forward, full).
Consumers must tolerate unknown fields; producers must not remove or repurpose
fields. Version the event type when a breaking change is unavoidable.

---

## Multi-tenancy data patterns

| Pattern | Isolation | Cost / density | Ops | Best for |
|---|---|---|---|---|
| **Silo** — database per tenant | Strongest (blast radius, noisy-neighbour, per-tenant restore, data residency) | Lowest density, highest cost | Many DBs to patch, migrate, monitor | Few large / regulated / enterprise tenants |
| **Bridge** — shared database, schema per tenant | Medium | Medium | Migrations fan out across schemas | Mid-market |
| **Pool** — shared schema, `tenant_id` column | Weakest — one query bug leaks data | Highest density, lowest cost | Single migration; must enforce `tenant_id` on **every** query (row-level security, a mandatory filter in the data layer) | Many small tenants, SaaS |

Most mature SaaS platforms mix: pool by default, silo for tenants who pay for it
or require it. Also plan for: per-tenant encryption keys, per-tenant rate limits,
noisy-neighbour throttling, per-tenant backup/restore, and tenant offboarding
(export + hard delete).

---

## Analytical data: warehouse, lake, lakehouse

| | Data warehouse | Data lake | Lakehouse |
|---|---|---|---|
| Storage | Proprietary columnar | Object storage (S3/GCS), open files (Parquet) | Object storage + open table format (Delta, Iceberg, Hudi) |
| Schema | Schema-on-write | Schema-on-read | Schema-on-read with ACID table layer |
| Users | BI / analysts (SQL) | Data scientists / ML | Both |
| Risk | Cost, less flexible | "Data swamp" without governance | Newer tooling |
| Examples | BigQuery, Snowflake, Redshift | S3 + Athena/Spark | Databricks, BigLake, Iceberg + Trino |

### Loading: ETL vs ELT

- **ETL** — transform before load. Fits fixed schemas, limited target compute,
  compliance filtering before landing.
- **ELT** — load raw, transform inside the warehouse (dbt). Fits elastic cloud
  warehouses; keeps raw data for reprocessing; faster to iterate. Default now.
- **Medallion / multi-hop:** Bronze (raw) → Silver (cleaned, conformed) → Gold
  (aggregated, business-ready).

Ingest via [CDC](#change-data-capture-cdc) or an event stream, not by querying
the OLTP database on a schedule.

---

## Polyglot persistence and data mesh

- **Polyglot persistence** — use the right store per bounded context (orders in
  PostgreSQL, session in Redis, catalogue search in Elasticsearch, activity feed
  in Cassandra, recommendations in a graph DB). Cost: operational surface area,
  cross-store consistency, more skills to maintain. Justify each store with a
  measured access pattern.
- **Data mesh** — organisational model: domain teams own their analytical data as
  **data products** (documented, discoverable, SLA'd, quality-tested), with a
  self-serve platform and federated governance. Solves the central-data-team
  bottleneck; needs real platform investment and maturity. Not a technology.

---

## Backup, retention, and data governance

- **RPO** (recovery point objective) — max acceptable data loss, in time. Drives
  backup frequency and replication mode.
- **RTO** (recovery time objective) — max acceptable downtime. Drives standby
  strategy (cold restore vs warm standby vs hot multi-region).
- **Point-in-time recovery (PITR)** — continuous WAL archiving lets you restore
  to any second, essential for recovering from a bad deploy or `DELETE` without a
  `WHERE`.
- **Test restores.** An untested backup is not a backup. Rehearse the full
  recovery runbook on a schedule.
- Protect against logical corruption too (a bug that writes bad data), not just
  hardware loss — replicas faithfully copy the corruption.
- **Governance:** data catalog and lineage (where did this column come from),
  classification (PII / PCI / PHI), retention and deletion policies, access
  control and audit, encryption at rest (envelope encryption + KMS) and in
  transit, data residency / sovereignty, and a defensible answer to GDPR/CCPA
  access and erasure requests.

---

## Architect checklist

- [ ] Every operation classified by consistency need (linearizable vs eventual)
- [ ] Datastore choice justified by written-down access patterns, not entities
- [ ] Analytics separated from OLTP; movement via CDC/events, not scheduled DB scans
- [ ] Partition/shard key chosen for even distribution *and* query locality; hotspots considered
- [ ] Replication mode (sync/async) chosen against the RPO; read-your-writes handled
- [ ] Distributed writes use saga/outbox with idempotency keys; 2PC avoided
- [ ] Caching pattern chosen per data type; stampede/penetration/avalanche mitigated; hit ratio monitored
- [ ] Schema changes follow expand–contract; migrations versioned in CI/CD; both app versions compatible
- [ ] Multi-tenancy isolation model chosen; `tenant_id` enforced in the data layer
- [ ] Backups have a defined RPO/RTO and a *tested* restore runbook; PITR enabled
- [ ] PII located, classified, encrypted, and covered by a retention/erasure policy
- [ ] Event/API schemas governed by a registry with a compatibility mode

---

## Appendix: Q&A deep-dives

Plain-language walk-throughs from working through this note — the questions that
needed more than the reference above. Full transcript:
[data-architecture-notes.md](../notes/data-architecture-notes.md).
Each block links back to the section it belongs to.

- [OLTP, OLAP and streaming, and why storage layout follows from purpose](#deep-dive-oltp-olap-and-streaming-and-why-storage-layout-follows-from-purpose) — OLTP vs OLAP vs streaming
- [Choosing a datastore, worked](#deep-dive-choosing-a-datastore-worked) — the decision framework
- [Access patterns first, not entities](#deep-dive-access-patterns-first-not-entities) — data modeling
- [Consistency per operation, and making stale reads safe](#deep-dive-consistency-per-operation-and-making-stale-reads-safe) — transactions and isolation
- [Write-heavy patterns and why LSM trees exist](#deep-dive-write-heavy-patterns-and-why-lsm-trees-exist) — indexing and query performance
- [Partitioning vs sharding, and the partition-key/unique-constraint trap](#deep-dive-partitioning-vs-sharding-and-the-partition-keyunique-constraint-trap) — partitioning and sharding
- [DynamoDB, partitioning as the data model](#deep-dive-dynamodb-partitioning-as-the-data-model) — the datastore families
- [Sharding, what it actually costs, and picking a shard key](#deep-dive-sharding-what-it-actually-costs-and-picking-a-shard-key) — partitioning and sharding; distributed transactions
- [Replicas vs shards, and CQRS on top](#deep-dive-replicas-vs-shards-and-cqrs-on-top) — replication; CQRS and event sourcing
- [Why relational gives up write scaling](#deep-dive-why-relational-gives-up-write-scaling) — the datastore families
- [Four families compared, and the locality principle](#deep-dive-four-families-compared-and-the-locality-principle) — the datastore families; data modeling
- [Document databases, embedding vs referencing](#deep-dive-document-databases-embedding-vs-referencing) — the datastore families
- [Wide-column and Cassandra, query-first modelling](#deep-dive-wide-column-and-cassandra-query-first-modelling) — the datastore families; partitioning and sharding

### Deep-dive: OLTP, OLAP and streaming, and why storage layout follows from purpose

Relates to [OLTP vs OLAP vs streaming](#oltp-vs-olap-vs-streaming) and
[the datastore families](#the-datastore-families).

**The plain version:** OLTP is the cash register — one small thing, right now,
very fast, many registers at once. OLAP is the notebook you read at month end —
reads everything, takes a minute, tells you something you didn't know.
Streaming is the friend at the door counting people — remembers only the last
few minutes, but tells you *now*. One sentence: **OLTP runs the business, OLAP
explains the business, streaming reacts to the business.** Only the *purpose*
row of the comparison table is a real choice — access pattern, latency, volume,
schema and store all fall out of it.

**Row vs columnar is a one-dimensional-disk problem.** A disk is one long line
of bytes; a table is two-dimensional, so something has to pick an order to
flatten it — along the rows, or down the columns. That's the entire difference.
Three wins follow from going columnar: you stop reading columns the query
didn't ask for (2 of 80 columns read instead of all 80), same-typed neighbouring
values compress far better (run-length and dictionary encoding, 10x is common),
and the CPU can process a flat array of one type with SIMD instead of decoding a
mixed-type record field by field. The cost is symmetric: `SELECT * WHERE
order_id = ?` on a column store means 80 separate file reads reassembled into
one row, and a single-row update means breaking a compressed run apart — which
is why columnar engines support "mutations" that rewrite whole chunks, not
row-level updates. **Column stores optimise for few columns of many rows; row
stores optimise for many columns of few rows.**

**Parquet is columnar, but via row groups, not one file per column.** Rows are
split into row groups (100k–1M rows) and *within* a row group the data is
column-by-column — so a worker can grab one row group and reassemble a row
without seeking across the whole file. The footer holds min/max per column
chunk, so a query like `WHERE date > 'Feb'` can skip entire row groups without
reading a byte — a well-sorted file can answer a query touching 1% of the data
by reading roughly 1% of it. Rule of thumb: **Avro for data in motion (Kafka
messages, whole records), Parquet for data at rest (analytics).** The format
being columnar doesn't guarantee the benefit — a file with one giant row group,
or thousands of tiny 500-row files, gets none of it.

### Deep-dive: choosing a datastore, worked

Relates to [Choosing a datastore — a decision framework](#choosing-a-datastore--a-decision-framework).

**Why the question order works:** questions 1–2 (access patterns, per-operation
consistency) are about *correctness* — get these wrong and the system is
broken. Questions 3–6 (read:write ratio, shape, query needs, scale ceiling) are
about *scale* — get these wrong and the system is slow or expensive. Question 7
(team, ops, cost model) is about *you*, and it's last only in order — it vetoes
the others. Most teams start at question 5 or 6 ("we need something that
scales") and work backwards, which is how a team ends up running Cassandra for
40GB that Postgres would have served happily, with nobody who can debug it at
2am.

**Worked example, carried through all seven questions:** a mid-size retailer's
order management — get order by id (high volume), list a customer's orders by
date, update status, revenue by region by month (a few times a day), free-text
address search. Consistency: creation/status updates linearizable, revenue
report can be a day stale, search can lag a minute. Volume: ~200 orders/minute,
read:write ~50:1 — small. Shape: firmly relational. Query needs: joins and
aggregation plus one full-text case. Ceiling: a single primary for years. Ops:
small team, RDS available. → **Postgres**, with a read replica for the
reporting query and Postgres full-text for address search — nothing else until
something measured says otherwise. Change one input — 50,000 orders/minute with
sub-second fraud checks — and Q2/Q3/Q6 all flip, landing on Postgres for
orders + Kafka/Flink for fraud + a warehouse for reporting. Same framework,
different answers because the inputs changed, not because the framework did.

**The DynamoDB-vs-Postgres tell:** DynamoDB has real atomicity (every
single-item write is linearizable; conditional writes are the inventory-decrement
pattern) and its reads aren't "easier," they're *narrower* — instant at any
scale for the exact queries you designed for, and effectively impossible for
anything else. The actual question is *"do I know all my access patterns in
advance, and will they stay stable?"* If a product manager will eventually ask
"can we see this broken down by region?", that's an unplanned query — cheap in
Postgres (an afternoon of SQL), expensive in DynamoDB (a full table scan, or a
GSI that's a second copy of the data paid for forever). The cost of rigidity is
paid up front and feels free; the benefit only shows up at a scale most systems
never reach.

### Deep-dive: access patterns first, not entities

Relates to [Data modeling](#data-modeling).

Choosing columnar vs row is a smaller, mostly-settled decision (running the
business is row, analysing it is columnar). Question 1 of the datastore
framework decides something harder: **what shape the data physically takes, and
how it's keyed.**

| Pattern | What it decides |
| --- | --- |
| "Get order by id" | `order_id` is the primary access key — trivial in Postgres, and in DynamoDB it's the partition key, a permanent choice. |
| "List orders for a customer, sorted by date" | `customer_id` and `date` must be co-located in storage order — a composite index `(customer_id, date)` in Postgres (column order matters), partition key `customer_id` + sort key `date` in DynamoDB. |
| "Sum revenue by region by month" | `region` and `month` must be available without joining across a billion rows → denormalise region onto the order row, and this query shouldn't run on the primary at all. |

None of those answers was "columnar" or "row" — they were which key, which sort
order, which index, what gets copied where. The third pattern (many rows, few
columns, aggregate) is the signature of an OLAP query, but the access pattern
taught you that it belongs in a *different system*, not that it "wants
columnar" directly.

**Why entity-first is the trap:** modeling Customer/Order/OrderLine/Product and
normalising gives a model that answers every query *adequately* and none of
them *well* — the customer-orders-by-date screen ends up doing a join plus a
sort over 200k rows on every page load. Access-pattern-first flips it: **the
queries are the requirement, the model is the implementation.** In Postgres
this mostly changes indexes and a few denormalisations; in a KV or wide-column
store it changes *everything* — the table structure is a transcription of the
query list, and storing the same data three times under three keys to serve
three access patterns is normal there and insane in Postgres. The practical
rule: write the query list with frequencies, and for each ask *what would have
to be true on disk for this to be one cheap lookup?* — any conflict between two
queries is an index, a denormalisation, or a second store you need.

### Deep-dive: consistency per operation, and making stale reads safe

Relates to [Transactions and isolation](#transactions-and-isolation).

**The three-path split.** The same application routes three operations down
three different paths based on one question — *if this value is 200ms out of
date, what breaks?*

| Operation | Requirement | Where it runs |
| --- | --- | --- |
| Decrement inventory | Linearizable | Primary, in a transaction — no cache, no replica |
| Show product page | Seconds stale is fine | Cache or read replica, TTL = staleness budget |
| Recommendation feed | Minutes stale is fine | Precomputed KV store, rebuilt by a batch job |

**The inventory race and its fix.** Read-modify-write in application code
(`qty = get(); if qty > 0: set(qty-1)`) lets two threads both read 1 and both
write 0 — two units sold, one in stock. The fix pushes the check into a single
atomic statement the database evaluates: `UPDATE inventory SET quantity =
quantity - 1 WHERE product_id = ? AND quantity > 0`, then check the affected
row count. This works cheaply because inventory is **single-key** — all
contention is on one row. *Multi-key* linearizability (decrement stock AND
charge the card AND create the order, atomically) is what gets expensive, which
is why real checkout flows use a **reservation**: hold stock for 10 minutes
(one cheap linearizable write), then take payment, then confirm — one hard
distributed transaction traded for two easy local ones plus a timeout.

**You don't fix a stale read by making the read fresh — you fix it by making
the write refuse to commit against state that changed.** Even a read from the
primary is stale by the time the application acts on it; the bug is *trusting*
the value at write time, not the staleness of the read itself. Two cases:

- **Case A — the write can be expressed relative to current state** (inventory,
  balances, counters): `WHERE quantity > 0` never uses the stale number the
  replica returned, so both users can read stale values and the outcome is
  still correct. Cheapest, most robust — use it whenever the write can be
  phrased as a relative change with a guard condition.
- **Case B — the write depends on what the user actually saw** (editing a
  product, approving a displayed price): carry a version/etag through from read
  to write — `WHERE id = ? AND version = 7` — so a concurrent update makes the
  write affect zero rows and fail loudly instead of silently overwriting. This
  is optimistic concurrency control (JPA's `@Version`, HTTP `ETag`/`If-Match`).

**When the check fails:** Case A (inventory) — don't retry, the answer is
genuinely "sold out." Case B (concurrent edit) — a background job can retry
with backoff; a human editing a form should be *shown* the conflict, not
silently overwritten. Make the operation idempotent (a client-generated
idempotency key) so a retry after a timeout doesn't double-decrement. Pessimistic
locking (`SELECT ... FOR UPDATE`) is correct but holds a lock across the whole
read-decide-write span — reserve it for frequent conflicts where retrying is
expensive; for normal checkout the Case A conditional update holds the lock for
microseconds instead of milliseconds. **The rule:** never let application code
perform check-then-act across a network boundary — express the condition
inside the write, or hold a lock for the entire span.

### Deep-dive: write-heavy patterns and why LSM trees exist

Relates to [Indexing and query performance](#indexing-and-query-performance)
and [Data modeling](#data-modeling).

**The ladder — work down it in order, each step costs more than the last:**

1. **Don't write it** — coalesce and sample. "Last seen at" becomes one update a
   minute kept in memory and flushed periodically; view counts get buffered and
   flushed as `+247` instead of 247 increments. Routinely removes 90% of write
   volume.
2. **Batch** — one statement writing 1,000 rows beats 1,000 statements writing
   one; Postgres `COPY` is roughly 10x faster than `INSERT`.
3. **Cut the cost of each write** — every index is a write tax; audit and drop
   unused ones. Synchronous replication doubles write latency if durability
   allows async instead.
4. **Buffer with a queue (write-behind)** — absorbs spikes (5k writes/sec
   sustained DB, 40k/sec for 30 seconds during a flash sale) but doesn't raise
   long-run capacity, and turns the write asynchronous (needs idempotency keys).
5. **Append instead of update** — update-in-place is expensive (find the page,
   lock, modify, WAL, and in Postgres a whole new row version autovacuum must
   clean up). **Event sourcing** takes this to its conclusion: never update the
   balance, append `Deposited(100)`, derive it — which is why event sourcing and
   CQRS pair so often.
6. **Spread the writes** — partitioning/sharding, designed for early because
   it's hard to retrofit. The entire game is the partition key; a classic
   mistake is partitioning event data by timestamp, so all of today's writes
   land on one partition.
7. **The single-row contention case** — 10,000 writes/sec to *one* row (a
   viral post's like counter) can't be fixed by sharding, because it's one key.
   Fix with **sharded counters** (split into N rows, write to one at random,
   sum all N to read) or aggregate in a stream (Kafka + Flink, flush a periodic
   total).

Steps 1–3 are tuning and don't change the architecture — that's where effort
should go first. A well-tuned Postgres does tens of thousands of writes/sec;
confirm you're near that ceiling before reaching past step 3.

**LSM trees are the storage engine built for step 5.** The design comes from
one hardware fact: sequential disk writes are fast, random ones are slow. A
B-tree (Postgres, MySQL) updates in place — a random I/O. An LSM tree **never
modifies anything on disk, only appends**: writes go to a write-ahead log, then
an in-memory sorted memtable; when the memtable fills it flushes as an
immutable, sorted **SSTable**. An update just writes the new value again (newest
copy wins); a delete writes a **tombstone**. Reads check the memtable, then each
SSTable newest-to-oldest — the hard part — rescued by **bloom filters** ("is
this key definitely not here?", skip the file) and sparse indexes. A background
**compaction** process merges SSTables, drops tombstoned rows, and reclaims
space. The trade is named directly: **write amplification** (one logical write
rewritten several times by compaction, 10–30x is normal), **read
amplification** (one read may touch several files), **space amplification**
(old versions sit until compaction runs) — versus a B-tree's low read/space
amplification but expensive random writes. LSM is right for sustained
high-volume writes with a known key pattern (event ingestion, time series,
messaging); wrong for ad-hoc analytical queries, heavy joins, or volume a
single relational database handles comfortably — which is most systems.

### Deep-dive: partitioning vs sharding, and the partition-key/unique-constraint trap

Relates to [Partitioning and sharding](#partitioning-and-sharding).

**The toy-box version.** One toy box, everything piles in, the lid won't shut.
Split into several boxes by a rule (dinosaurs in box 1, cars in box 2) — finding
a dinosaur means opening one box. **That rule is the only thing that matters**;
a bad rule (before/after my birthday) leaves one box growing forever.
**Partitioning** = several boxes in your own room — one machine, several
tables, the database can still join and transact across them for free.
**Sharding** = the boxes are in different houses — a machine boundary, so joins,
transactions and uniqueness stop being free. Hence: **partition early, shard
late.**

**Postgres range partitioning, concretely:**

```sql
CREATE TABLE orders (
    order_id bigserial, customer_id bigint NOT NULL,
    order_date date NOT NULL, amount numeric(12,2),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

A query with `order_date` in the `WHERE` clause prunes to one partition; a query
on `customer_id` alone scans **every** partition — partitioning made that query
worse, because you optimise for the access patterns you wrote down. Pruning
needs the raw column (`date_trunc(order_date)` defeats it) and a `DEFAULT`
partition is a safety net that becomes a silent dumping ground if relied on.
The benefit people actually partition for is lifecycle: `DETACH PARTITION` +
`DROP TABLE` replaces an hours-long `DELETE FROM ... WHERE date < X` with
near-instant metadata work — no vacuum storm.

**Why the partition key must be in the unique key — and it's the reverse of
what it looks like.** Postgres only has *local* indexes, one per partition,
knowing nothing about the others. Including the partition key in the primary
key doesn't let Postgres check uniqueness across all partitions — it lets
Postgres **avoid** having to, by guaranteeing two rows that could collide are
routed to the same partition. It's a routing guarantee, not a global check —
which is why `UNIQUE (order_id, order_date)` happily accepts the same
`order_id` twice under two different dates. Four real ways to get a genuinely
unique `order_id` anyway: (1) generate IDs that can't collide — a shared
sequence or UUIDv7 (covers ~95% of cases, but is a convention, not a
constraint); (2) partition by something the ID already implies (`PARTITION BY
HASH (order_id)`, giving up date pruning and cheap retention drops); (3) derive
the partition key from a time-ordered ID (`PARTITION BY RANGE (order_id)`,
keeping both properties approximately); (4) a separate unpartitioned registry
table enforcing the constraint directly, at the cost of a shared contention
point that can't be aged out. **The real choice: an enforced global unique key,
or date-range partitioning — Postgres won't give both on the same column**
(Oracle's global indexes offer both, at the cost of an expensive partition
drop).

### Deep-dive: DynamoDB, partitioning as the data model

Relates to [The datastore families](#the-datastore-families) and
[choosing a datastore](#choosing-a-datastore--a-decision-framework).

In Postgres, partitioning is an optimisation added later. In DynamoDB it **is**
the data model, mandatory and permanent: every item has a partition key, hashed
to decide which physical partition holds it — no range or list option, no
strategy choice. Vocabulary maps directly: table → table, **item** → row,
**attribute** → column. The table designer decides *which attribute* is the
partition key, once, at table creation; the writer decides *what value* it has,
per write — the same as choosing what goes in a column.

**Single-table design** puts related items in the same partition so the sort
key does the work a join would have done:

| PK | SK | attributes |
| --- | --- | --- |
| `CUSTOMER#42` | `PROFILE` | name, email |
| `CUSTOMER#42` | `ORDER#2026-01-15#1001` | amount, status |

"Get customer 42 and their recent orders" becomes one query on one partition,
because DynamoDB has no joins and this co-locates the data instead. This only
works because access patterns were enumerated first — a query without the
partition key isn't a slow query like in Postgres, it's a `Scan`: reads the
whole table, bills for everything read, with no "add an index later" escape (a
GSI is a full second copy of the data, paid for permanently).

**Hot-partition limits that bite:** 10GB per item collection (all items sharing
a partition key, when an LSI exists) and 3,000 RCU / 1,000 WCU **per
partition** — a table provisioned for 100,000 WCU is still capped at 1,000 if
every write targets one key. `status = 'PENDING'` or `date = today` as a
partition key puts all matching writes on one machine. **Write sharding** — a
random numeric suffix (`PENDING#0`…`PENDING#9`), reads query all and merge —
trades read cost for write throughput deliberately.

**The deal, plainly:** DynamoDB says "tell me exactly which questions you'll
ask — those are instant forever, at any size, but I'll only answer those."
Postgres says "ask me anything, whenever — some answers will be slower, and
past a certain size I'll struggle." Choose DynamoDB when access is by a known
key, you need consistent single-digit-ms latency at any scale (not just
average-case), and write volume genuinely spreads across many keys. The tell
that you actually wanted Postgres: a product manager eventually asks "can we
see this broken down by region?" — a question DynamoDB's key design didn't
anticipate.

### Deep-dive: sharding, what it actually costs, and picking a shard key

Relates to [Partitioning and sharding](#partitioning-and-sharding) and
[Distributed transactions across stores](#distributed-transactions-across-stores).

**The one difference that creates every other difference:** partitioning splits
data across tables on one machine; sharding splits it across **separate
database servers that don't know about each other**. Cross that line and the
database stops helping: no joins, no transactions, no unique constraints, no
`ORDER BY` across shards — every one becomes the application's problem. You're
not buying a feature, you're giving up features to get write scaling.

**What actually forces sharding:** write throughput beyond one primary, dataset
size beyond one machine, regulatory data residency (EU data must live in the
EU), or blast-radius isolation. **"Our database is slow" isn't on the list** —
that's usually missing indexes, N+1 queries, or analytics hitting the primary.

**Strategies:** range (skew-prone on sequential keys like timestamps), hash
(even spread, but `% N` means adding a server reshuffles almost everything),
directory/lookup (total flexibility, but the lookup service becomes a
dependency that must never go down — common in multi-tenant SaaS). The fix for
`% N` is **consistent hashing / virtual buckets**: map keys to a large fixed
number of buckets (never changed), then map buckets to physical shards —
`hash(key) % 1024` never changes, only the bucket→shard assignment does, so
adding a shard moves ~1/N of the data instead of all of it.

**What you lose, concretely:**

- **Cross-shard joins** — mitigated by co-locating tables on the same shard key
  (shard both `orders` and `customers` by `customer_id`) and replicating small
  lookup tables to every shard.
- **Cross-shard transactions** — restructure so transactions stay within a
  shard, or adopt a saga (local transactions + compensating actions).
- **Global unique constraints and auto-increment IDs** — need a registry table
  or collision-proof IDs (UUIDv7, Snowflake, per-shard sequence offsets).
- **Aggregations and pagination** — `SUM` becomes scatter-gather across every
  shard; deep offset pagination becomes impractical, cursor pagination is the
  workaround.
- **Schema migrations** — must run on every shard; tooling is mandatory.

**Resharding a live system:** stand up the new shard, backfill the moving
buckets while traffic continues, double-write old and new, **verify the two
sides agree** (the step people underestimate — without it, drift surfaces from
a customer months later), cut reads over bucket by bucket, then stop
double-writing.

**Exhaust these first, cheapest to most expensive:** fix queries/indexes,
vertical scaling (a 128-core/2TB machine is real and cheaper than an
engineering team), read replicas, caching, move analytics off the primary,
partition within one database, a **functional split** (move a whole subsystem
like billing to its own database — often enough on its own), archive cold data.

**Picking the shard key:** same criteria as a partition key with higher stakes
— high cardinality, even distribution, present in nearly every query, and
stable. For most business systems it's `customer_id`/`tenant_id`, because it
co-locates everything one customer touches so most queries stay single-shard;
a tenant too large for one shard gets a dedicated shard or is sub-sharded.
**Honest summary:** sharding converts a database problem into an application
problem — write scaling and blast-radius isolation are bought with joins,
transactions, constraints, and migrations, permanently.

### Deep-dive: replicas vs shards, and CQRS on top

Relates to [Replication](#replication) and
[CQRS and Event Sourcing](#cqrs-and-event-sourcing).

**Replicas copy writes. Shards divide writes.** A read replica is a full copy —
every write to the primary must also apply on every replica, so adding
replicas adds *read* capacity but never reduces anyone's write load; the
primary now ships WAL to more replicas and, under synchronous replication,
waits longer for acks. With N shards, each is a *different* database holding a
*different* slice of the data — 4 shards at 10,000 writes/sec means each does a
quarter of the work, and adding a fifth raises the ceiling again. **The
asymmetry in one line: reads can be duplicated, writes must be divided.** They
compose: shards give write capacity, replicas within each shard give read
capacity on top.

**Two different read problems, easy to conflate:**

| Problem | Solved by | Why |
| --- | --- | --- |
| Read volume *within* a shard | Replicas | Shard B's replicas hold shard B's data — serve any read that includes the shard key |
| Reads that don't fit the shard key | CQRS projection | Replicas do nothing here — a replica of shard B still only knows shard B |

If shards are keyed by `customer_id` and someone asks "all orders shipped last
Tuesday," every shard holds part of the answer — that's what CQRS is for here,
not replicas. **How the projection actually gets built:** tap each shard's
replication log (Postgres logical decoding via Debezium, MySQL binlog, DynamoDB
Streams), publish to Kafka, and a consumer merges the streams into whatever
store answers the query well (Elasticsearch for search, a warehouse for
aggregation, a denormalised table keyed for the read). **Why CDC and not dual
writes from the app:** dual writes have no atomicity — the DB commit can
succeed while the Kafka publish fails, permanently out of sync with nothing to
detect it; CDC reads the committed log, so it can't disagree with what
committed. The projection is eventually consistent (seconds, occasionally
minutes behind), so the routing rule from the consistency deep-dive applies
directly: linearizable decisions go to the shard primary, reads tolerating
staleness go to shard replicas, cross-shard/analytical reads go to the
projection. **One correction to how this is usually phrased:** "read replicas
**or** read views" is the wrong framing — it's usually both, and they're not
alternatives. Replicas copy the write model at the same keying; projections are
a different model with different keying. CQRS is justified once read patterns
genuinely don't fit the write partitioning (usually true once you've sharded),
but it costs a pipeline to operate and a reconciliation path to build — build
that rebuild path early, before an incident forces it.

### Deep-dive: why relational gives up write scaling

Relates to [The datastore families](#the-datastore-families).

**Partitioning doesn't break queries.** Within one Postgres, joins across
partitions still work, transactions spanning partitions are still atomic, and
foreign keys still hold. `WHERE customer_id = ?` on a date-partitioned table
returns the right answer — it just scans all 24 partitions instead of one.
**Slower, not broken.** So partitioning is a *tuning* step, not a *scaling*
step: it never raises the write ceiling, because every partition still shares
one machine's CPU, disk and WAL.

**Sharding is where it actually breaks, and relational specifically hurts.**
Cassandra and DynamoDB shard happily. Relational doesn't, because the things
that break under sharding are exactly the things that make a database
relational: joins, multi-row transactions, foreign keys, unique constraints.
Every one of them assumes a single coordinator that sees all the data and can
order operations against it. A key-value store never promised joins, so
distributing it costs nothing; relational promised all of it, so distributing
it means handing all of it back.

> **Relational can be scaled horizontally for writes, but only by giving up the
> properties that made it relational.**

**Replicas don't help writes at all — not "a bit less," zero.** Ten replicas
means ten machines each carrying the *full* write load, because every replica
must apply every write to stay a faithful copy.

**There is no escape hatch inside the relational model:** ACID needs one
authority ordering writes → replicas copy that authority's output, they don't
divide the work → partitioning rearranges data on that same one authority →
sharding actually divides the work, and that's exactly what costs the
guarantees.

**It's a trade, not a law.** Spanner and CockroachDB do give SQL, joins and
distributed transactions across machines — they just show the price:
consensus per shard means every write waits for a quorum, milliseconds where
a single Postgres primary takes microseconds, and far worse across regions.
**They paid in latency instead of in lost guarantees.** Relational's
guarantees are *cross-row*, and cross-row guarantees need a single
coordinator — split the rows and either the guarantees go, or the latency
goes up. There is no third option.

### Deep-dive: four families compared, and the locality principle

Relates to [The datastore families](#the-datastore-families) and
[Data modeling](#data-modeling).

**Every family optimises one access pattern and taxes the rest** — nothing
here is generally "faster," only faster at the thing it was built for.

| | Key-value | Relational | Document | Wide-column |
|---|---|---|---|---|
| Access | Only by known key | Any field, joins across tables | Any field in a collection; joins are the weak spot | Partition key required; range-slice on clustering key |
| Core strength | Fastest lookup, trivial scaling | Enforced correctness — FKs, constraints, multi-row transactions | Locality — one aggregate, one read | Leaderless writes — any node accepts |
| Write scaling | Easy — nothing to give up | Hard — needs one coordinator | Easier (joins already surrendered); still one primary per shard | Easiest — no primary at all |
| Gave up | Every non-key question | Write scaling without sharding pain | Joins, cross-document constraints | Ad-hoc queries; data stored once per query |

**Choosing, in order:** only ever fetch by a known key → key-value. Queries
span entities — joins, aggregation, questions not yet asked → relational. One
self-contained document answers it, and records are genuinely heterogeneous →
document. "Everything for one entity, over time, by recency" at very high
write volume → wide-column. If both relational and document feel true, pick
relational — an unexpected join costs some SQL there; in a document store it
costs a data migration.

**Locality is the point worth remembering — it's not about schema
flexibility, it's about how many places the disk has to visit:**

| | Relational | Document |
|---|---|---|
| Fetching one order | 4 tables — header, lines, address, history | 1 document |
| Physical work | 4 index lookups + join | 1 seek, 1 contiguous read |
| Why | Normalised into separate places | Whole aggregate as one blob on disk |

The mirror image: the same locality that makes reads cheap makes shared data
expensive — a school name embedded in 500 student documents means 500 updates
when it changes, where relational writes it once. Wide-column has its own
version of the same principle at the *partition* rather than the *aggregate*
level: a partition is contiguous and pre-sorted, so "latest 20 for this
entity" is one sequential read with no sorting at query time.

**Key-value vs wide-column** isn't about whether you *could* jam everything
into one blob — it's what happens next: a key-value value is one opaque blob
(read part of it → fetch and parse the whole thing; append one item → rewrite
the blob), while a wide-column row is many cells sorted by a clustering key
(range-scan a slice; append one small write). Hence *wide* — not many rows,
but a single row that is enormously wide, with different columns per row.

**Wide-column is not a warehouse**, despite both being associated with "big
data": wide-column (Cassandra) is OLTP at scale — queries name a key
("Anna's last 20 events"), single-digit-ms latency, row-oriented within a
partition. Analytical/columnar (BigQuery) is OLAP — no key ("avg session by
country, last quarter"), seconds-to-minutes latency, column-oriented across
all rows. "Columnar" means two different things here and it's easy to
conflate them.

**The collapse case:** Postgres `JSONB` with a GIN index gives flexible
documents with real indexing, *plus* joins, transactions and constraints —
which is why the genuine reasons to reach for MongoDB tend to be operational
(team knowledge, a managed platform, a measured sharding need) rather than
modeling ones.

### Deep-dive: document databases, embedding vs referencing

Relates to [The datastore families](#the-datastore-families).

**The picture:** relational is a filing cabinet — one drawer for names, one
for hobbies, one for addresses, three drawers to learn everything about Anna.
A document store is one envelope per person — one trip. **The entire design
rule: put things together that you always look at together.** The warning
that comes with it: an envelope is only a good idea while it's about one
thing — stuff every receipt into it and it grows forever and gets slow to open
and update.

**Document querying is closer to relational than to key-value.** A document
store can query by any field, including nested fields and array contents, not
just a key — the real gap from key-value. But "can" isn't "should": a query
on a non-indexed field scans every document, fine at 50,000 docs, a disaster
at 50 million, so the same indexing discipline as relational applies. What
document actually gave up is **joins** and **cross-document constraints**, not
field querying.

**The rule that decides embed vs reference is boundedness, not
convenience:**

> **Embed what is bounded and read together. Reference what is unbounded or
> read separately.**

A customer's addresses are bounded — a handful, reached a size and stayed
there — so they embed. A customer's orders are unbounded — ten years is
thousands — so they're separate documents linked by `customerId`, at the cost
of no foreign key, no cascade, and "customer with recent orders" becoming two
round trips (MongoDB's `$lookup` exists but is slower than a relational join
and behaves poorly on sharded clusters). The checklist per piece of data:
grows without limit → separate collection; always read with the parent →
embed; read on its own or shared → separate; would push past a few hundred KB
→ separate.

**MongoDB can partly imitate the wide-column strategy, but only the
modeling half.** A compound index like `{ userId: 1, ts: -1 }` gives "Anna's
last 20 events" — the same partition-key-plus-clustering-key idea — and the
bucket pattern (one document per user per month, events pushed into an array)
is the closer analogue, formalized as MongoDB's time-series collections
(5.0+). What doesn't translate is the *scaling* model: writes in Cassandra go
to any node with no election on failover; in MongoDB writes go to the shard's
primary, and adding capacity means adding a shard and rebalancing chunks.
Cassandra earns its keep at hundreds of thousands of sustained writes/sec or
when active-active writes across regions are required; below that, MongoDB or
Postgres (with TimescaleDB for time-series shapes) serve the same pattern with
far less operational pain.

### Deep-dive: wide-column and Cassandra, query-first modelling

Relates to [The datastore families](#the-datastore-families) and
[Partitioning and sharding](#partitioning-and-sharding).

**The picture: a ring of houses.** Hashing a key says which house holds it,
and any house will take a write — there's no headmaster's office. The write
is copied to the next two houses clockwise, so if one burns down nothing is
lost and nobody holds a meeting about who's in charge. In Postgres or MongoDB
one machine is the boss for any given piece of data, and if it dies, everyone
pauses for an election; **in Cassandra there is no boss, ever** — which is
why it keeps taking writes while machines die, and why adding nodes adds
write capacity in a straight line. The cost: you must say *whose* data you
want. "Whose notes mention a red bicycle?" has no answer without knocking on
every door — so when a fact needs a second way to be found, it gets written a
second time, on purpose, as the design rather than a workaround.

**A Cassandra table isn't a place to store entities — it's a stored answer
to one specific query,** with the primary key built from a **partition key**
(hashed to place the row) followed by one or more **clustering keys**
(sorting rows *within* that partition, which is how they sit on disk). This
makes non-contiguous reads not just slower but often **not allowed**: CQL
rejects a `WHERE` clause that doesn't start with the partition key, and
`ALLOW FILTERING` — which would permit it by scanning every partition on
every node — should be read as a syntax error, not an option. The rule that
must not be broken: partition key plus *all* clustering columns must uniquely
identify a row, or writes silently overwrite each other with no error.

**Needing a second access pattern means a second table, kept in sync by
hand** — no trigger, no cascade, the application performs every write, often
inside a `LOGGED BATCH` for atomicity (though not isolation: a concurrent
reader can see one table updated and not the other). This is affordable
because Cassandra writes are cheap — no read-before-write, no coordination,
pure LSM appends — so three writes here can genuinely cost less than one
write plus an index update in a B-tree database. The two tempting
alternatives are both traps in practice: **materialized views** (Cassandra
maintains the second table for you, but are flagged experimental for years
and known to drift without self-repairing) and **secondary indexes** (look
like a normal index but are local to each node, so a lookup fans out to
every node — workable only for high-cardinality columns queried within a
known partition). Nothing enforces agreement between hand-kept tables, so the
practical discipline is: put all writes for one logical event in one place in
the code (a single repository method), rely on writes being naturally
idempotent (every write is an upsert) so retries are safe, and run periodic
reconciliation — drift is a *when*, not an *if*.

**The mental shift from Postgres:** there, you model the data once and add
indexes for new queries. **In Cassandra, a table *is* an index** — one
materialization shaped for one query, so adding a query means adding a table,
which means adding a write path. "Write down all your access patterns first"
isn't advice here; it's the entire design process, the same idea as
[access-patterns-first modeling](#deep-dive-access-patterns-first-not-entities)
taken to its logical extreme.
