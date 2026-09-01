# Data Architecture

Architect-level reference on how to store, model, replicate, partition, evolve,
and cache data in a distributed system. Companion to the unanswered question list
in [database-and-data-architecture-questions.md](database-and-data-architecture-questions.md)
and to [../architecture/cap-theorem.md](../architecture/cap-theorem.md).

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

---

## OLTP vs OLAP vs streaming

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
