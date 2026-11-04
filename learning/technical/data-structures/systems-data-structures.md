# Systems Data Structures — Reference

Quick reference for the data structures **inside** databases, caches and distributed
stores: B+ trees, LSM trees, Bloom filters, consistent hashing, skip lists and Merkle
trees. Step-by-step explanations, pseudocode, traces and code are in the layered
companion, [systems-data-structures-notes.md](../notes/systems-data-structures-notes.md):
**Level 1** the simple version · **2** how each one works · **3** deep dive · **4**
architect interview playbook.

The system-level trade-offs (indexing, LSM, sharding, caching) are in
[data-architecture.md](../data/data-architecture.md); this reference is about the
structures themselves. The tree structures link back to [trees.md](trees.md).

> **Status: preview — not yet read.** Written ahead of reading, like the other DSA notes.

## Contents

1. [The six at a glance](#1-the-six-at-a-glance)
2. [B+ tree](#2-b-tree)
3. [LSM tree](#3-lsm-tree)
4. [Bloom filter](#4-bloom-filter)
5. [Consistent hashing](#5-consistent-hashing)
6. [Skip list](#6-skip-list)
7. [Merkle tree](#7-merkle-tree)
8. [How they fit together](#8-how-they-fit-together)
9. [Interview playbook](#9-interview-playbook)

---

## 1. The six at a glance

| Structure | Problem it solves | Key cost | Main limitation | Seen in |
|---|---|---|---|---|
| **B+ tree** | Ordered index on disk, range queries | 3–4 page reads per lookup | In-place random writes; splits | Postgres, MySQL/InnoDB, Oracle, SQLite |
| **LSM tree** | Very high write throughput | Sequential writes; reads check several files | Write, read and space amplification; compaction I/O | Cassandra, RocksDB, HBase, LevelDB |
| **Bloom filter** | "Definitely not here" in a few bits per item | `O(k)` per query; ~10 bits/item at 1% | False positives; no delete | LSM stores, caches, crawlers |
| **Consistent hashing** | Spread keys over changing servers | Adding a server moves ~`1/N` of keys | Hot keys; poor range queries | Dynamo, Cassandra, memcached clients, CDNs |
| **Skip list** | Simple sorted structure, concurrency-friendly | Expected `O(log n)` | Not worst-case guaranteed | Redis sorted sets, RocksDB memtable, `ConcurrentSkipListMap` |
| **Merkle tree** | Find differences between big copies | ~`log n` hash comparisons per difference | Hashing and tree upkeep | Git, Dynamo/Cassandra repair, blockchains |

> Deep-dives: [six problems, six tools](../notes/systems-data-structures-notes.md#six-problems-six-tools) ·
> [in plain words](../notes/systems-data-structures-notes.md#in-plain-words) ·
> [the one-minute picture](../notes/systems-data-structures-notes.md#the-one-minute-picture-of-how-they-fit-together) ·
> [Level 1 check](../notes/systems-data-structures-notes.md#level-1-check)

---

## 2. B+ tree

A balanced search tree whose **node size equals the disk page** (4–16 KB). Internal nodes
hold only separator keys; **leaves hold all the data and are linked** left to right. All
leaves are at the same depth.

- **Search:** descend one page per level. **Range scan:** one descent, then walk the leaf
  links.
- **Insert:** put the key in its leaf; if the leaf overflows, **split** it and **copy** the
  separator up; if the parent overflows, split it and **move** the middle key up; if the
  root splits, the tree grows one level at the top.
- **Depth:** with ~500 entries per 8 KB page, 3 levels address ~125 million rows and 4
  levels ~62 billion; the top levels stay cached.
- **Clustered** index (InnoDB primary key): leaves hold the whole row. **Secondary**
  index: leaves hold the primary key, so a lookup may need a second lookup unless the index
  is **covering**.
- **Costs:** in-place random writes plus a write-ahead log; every extra index is another
  tree to update; random keys (UUIDv4) cause splits and fragmentation, sequential keys do
  not.

> Deep-dives: [structure, search, range scan](../notes/systems-data-structures-notes.md#the-b-tree-step-by-step) ·
> [B+ tree in depth](../notes/systems-data-structures-notes.md#b-tree-in-depth) ·
> [indexing and query performance](../data/data-architecture.md#indexing-and-query-performance)

## 3. LSM tree

Never edits on disk; it **appends**. Writes go to a **commit log** and a sorted in-memory
**memtable**; a full memtable flushes to an immutable sorted **SSTable** (with a sparse
index and a Bloom filter); background **compaction** merges SSTables, keeps the newest
version of each key and drops tombstones. A delete is a **tombstone** write.

| | LSM tree | B+ tree |
|---|---|---|
| Write | **Sequential, very fast** | Random in-place |
| Read | Memtable, then SSTables newest-first (Bloom filters skip files) | 3–4 pages |
| Write amplification | **High** (10–30x) | Moderate |
| Read amplification | Higher | Low |
| Space amplification | Higher | Low |

**Compaction:** *size-tiered* (lower write amp, higher read and space amp; write-heavy)
vs *leveled* (higher write amp, lower read and space amp; read-heavy).

> Deep-dives: [the write, read and compaction logic, with a trace](../notes/systems-data-structures-notes.md#the-lsm-tree-step-by-step) ·
> [amplification and compaction strategies](../notes/systems-data-structures-notes.md#lsm-tree-in-depth) ·
> [write-heavy patterns and why LSM trees exist](../data/data-architecture.md#deep-dive-write-heavy-patterns-and-why-lsm-trees-exist) ·
> [LSM stores (Q&A)](../notes/data-architecture-notes.md#9-lsm-stores)

## 4. Bloom filter

A bit array of `m` bits and `k` hash functions. **Add:** set `k` bits. **Query:** if any of
the `k` bits is 0 ⇒ **definitely not present**; if all are 1 ⇒ **probably present**. No
false negatives; tunable false positives; **no delete, no listing**.

```
m = − n·ln(p) / (ln 2)²        k = (m/n)·ln 2        (1M items at 1% ⇒ ~1.2 MB, k ≈ 7)
```

The false-positive rate climbs if you add more than the `n` it was sized for. Use only where
a false positive is a wasted check: skipping SSTables, rejecting non-existent keys before
the database, URL de-duplication. Variants: counting Bloom filter and cuckoo filter (both
support delete).

> Deep-dives: [the logic, a toy trace and Java](../notes/systems-data-structures-notes.md#the-bloom-filter-step-by-step) ·
> [false-positive maths and uses](../notes/systems-data-structures-notes.md#bloom-filters-in-depth)

## 5. Consistent hashing

Hash servers **and** keys onto a ring; a key belongs to the **first server clockwise**.
Adding or removing a server moves only ~`1/N` of keys (versus ~all with `hash % N`), and
only to or from its neighbours. **Virtual nodes** (many ring positions per server) even out
the load and spread a failed server's keys. **Replication:** the next `R − 1` distinct
servers clockwise. A sorted map (`TreeMap.ceilingEntry`, with wrap-around) implements it.

| Scheme | Notes |
|---|---|
| Ring + virtual nodes | Dynamo, Cassandra, caches, load balancers |
| Fixed hash slots | Redis Cluster: 16,384 slots moved between nodes |
| Rendezvous hashing | Highest `hash(key, node)` wins; `O(N)` per lookup |
| Jump consistent hash | Numbered buckets only; tiny and fast |

Limits: does not fix **hot keys**; adjacent keys scatter, so **range queries** need range
partitioning.

> Deep-dives: [ring, virtual nodes, trace and Java](../notes/systems-data-structures-notes.md#consistent-hashing-step-by-step) ·
> [alternatives and limits](../notes/systems-data-structures-notes.md#consistent-hashing-in-depth) ·
> [partitioning and sharding](../data/data-architecture.md#partitioning-and-sharding)

## 6. Skip list

A sorted linked list with **express lanes**: level 0 holds every item, each higher level
about half the previous, with promotion decided by **coin flips**. **Search:** start at the
top, move right while the next key is smaller, else drop a level. **Insert:** search,
choose a random height, splice in on each level. **No rotations.**

| | Skip list | Red-Black / AVL |
|---|---|---|
| Cost | Expected `O(log n)` | `O(log n)` guaranteed |
| Code | **Simple** | Rotations / balance rules |
| Concurrency | **Easy** (local pointer changes) | Hard |
| Range scan | Walk level 0 | In-order traversal |

Used in Redis sorted sets (with a hash table), the LevelDB/RocksDB memtable and Java's
`ConcurrentSkipListMap`/`Set`.

> Deep-dives: [structure, search, insert, Java](../notes/systems-data-structures-notes.md#the-skip-list-step-by-step) ·
> [skip lists in depth](../notes/systems-data-structures-notes.md#skip-lists-in-depth) ·
> [the Red-Black alternative](trees.md#5-red-black-tree)

## 7. Merkle tree

A tree of hashes: leaves hash data blocks, each parent hashes its children's hashes, the
root fingerprints everything. **Equal roots ⇒ identical data.** To find differences, compare
children and descend **only into differing branches**: ~`log n` comparisons per differing
block, and only those blocks are copied. An **inclusion proof** is the `log n` sibling hashes
on the path to the root.

Used for replica **anti-entropy repair** (Dynamo, Cassandra), **Git** (a Merkle DAG),
blockchains (block headers), Certificate Transparency and verified downloads.

> Deep-dives: [build, diff, trace and inclusion proof](../notes/systems-data-structures-notes.md#the-merkle-tree-step-by-step) ·
> [Merkle trees in depth](../notes/systems-data-structures-notes.md#merkle-trees-in-depth)

---

## 8. How they fit together

A **Cassandra-style write**: **consistent hashing** picks the replicas → each appends to a
**commit log** and inserts into a **memtable** (skip-list-like) → flushes to an **SSTable**
with a **Bloom filter** → background **compaction**; replicas periodically compare **Merkle
trees** to repair drift. A **relational** database instead keeps rows and indexes in **B+
trees**, updated in place.

> Deep-dive: [a Cassandra-style system, with a diagram](../notes/systems-data-structures-notes.md#how-they-fit-together-a-cassandra-style-system) ·
> [choosing a datastore](../data/data-architecture.md#choosing-a-datastore--a-decision-framework)

## 9. Interview playbook

| You need… | Use |
|---|---|
| Ordered on-disk index, ranges, balanced reads and writes | **B+ tree** |
| Very high write throughput | **LSM tree** |
| Cheap "definitely not here" | **Bloom filter** |
| Spread keys over changing servers | **Consistent hashing** |
| Sorted in-memory structure, concurrent | **Skip list** |
| Locate differences between big copies | **Merkle tree** |

**Answer shape:** the constraint that forces it → the one-line idea → the cost → the
trade-off → a real system. **Mistakes:** treating a Bloom "yes" as certain; undersizing
the filter; `hash % N` for changing shard counts; too few virtual nodes; expecting
consistent hashing to fix hot keys; random UUID clustered keys; too many indexes; ignoring
compaction cost.

> Deep-dives: [twelve model Q&As](../notes/systems-data-structures-notes.md#model-qa) ·
> [choosing the structure](../notes/systems-data-structures-notes.md#choosing-the-structure) ·
> [common mistakes](../notes/systems-data-structures-notes.md#common-mistakes) ·
> [how to answer](../notes/systems-data-structures-notes.md#how-to-answer-in-an-interview) ·
> [practice prompts](../notes/systems-data-structures-notes.md#practice-prompts) ·
> Checks: [Level 2](../notes/systems-data-structures-notes.md#level-2-check) ·
> [Level 3](../notes/systems-data-structures-notes.md#level-3-check) ·
> [Level 4](../notes/systems-data-structures-notes.md#level-4-check)
