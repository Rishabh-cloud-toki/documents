# Data Structures & Algorithms — Study Plan

The order in which the DSA notes are written and studied, why that order, and what each
step will contain. Update the status column as steps are written and read.

**Status legend:** ✅ written and read · ✍️ written (preview — not yet read) · 📝 planned

## How every step is written

Each topic gets two files, as with trees, heaps and graphs:

- a **reference** (concise tables, costs and decision guides), and
- a **layered companion** in [notes/](notes/) that teaches it in levels: **simple version →
  how it works (hand traces) → deep dive → practice problems → interview playbook**.

Rules every companion follows:

1. **Simple first.** Level 1 uses plain words and one small running example, with no
   jargon or comparisons the reader has not met.
2. **Pseudocode before code.** Each algorithm is given as step-by-step pseudocode, then
   explained line by line ("Reading it"), then shown in Java.
3. **Check questions** at the end of every level.
4. Cross-links to related notes instead of repeating them.

## Where things stand

| # | Topic | Files | Status |
|---|---|---|---|
| — | Trees (BST, AVL, Red-Black, traversals, `TreeMap`) | [trees.md](data-structures/trees.md) · [trees-notes.md](notes/trees-notes.md) | ✅ |
| — | Heaps and priority queues | [heaps.md](data-structures/heaps.md) · [heaps-notes.md](notes/heaps-notes.md) | ✅ |
| — | Graphs (BFS, DFS, topological sort, Dijkstra, MST) | [graphs.md](data-structures/graphs.md) · [graphs-notes.md](notes/graphs-notes.md) | ✍️ |
| 1 | **Tries** | [tries.md](data-structures/tries.md) · [tries-notes.md](notes/tries-notes.md) | ✍️ |
| 2 | **Stacks and queues** | `data-structures/stacks-and-queues.md` · `notes/stacks-and-queues-notes.md` | 📝 |
| 3 | **Binary search, two pointers, sliding window** | `algorithms/binary-search-and-windows.md` · `notes/binary-search-and-windows-notes.md` | 📝 |
| 4 | **Recursion and backtracking** | `algorithms/recursion-and-backtracking.md` · `notes/recursion-and-backtracking-notes.md` | 📝 |
| 5 | **Dynamic programming** | `algorithms/dynamic-programming.md` · `notes/dynamic-programming-notes.md` | 📝 |
| 6 | **Data structures inside real systems** | [systems-data-structures.md](data-structures/systems-data-structures.md) · [systems-data-structures-notes.md](notes/systems-data-structures-notes.md) | ✍️ |

## The steps, and why in this order

### 1. Tries
Closes the trees chapter, which already previewed them. Small, and a good warm-up for
the layered style. Covers: the prefix idea, insert/search/startsWith, autocomplete,
deletion, memory trade-offs and compressed tries, classic problems (wildcard search,
longest common prefix), and when a trie beats a hash set or `TreeMap`.

### 2. Stacks and queues
The most basic linear structures, and already used throughout the notes without a chapter
of their own: stacks drive iterative DFS and the recursion call stack, queues drive BFS,
level-order and topological sort. Placed here because the next steps lean on them
(sliding-window maximum uses a deque; recursion is the call stack). Covers: **stack**
(LIFO) and **queue** (FIFO) with array and linked-list implementations, the circular
queue, the **deque**, and Java's `ArrayDeque` (and why not the legacy `Stack` class);
classic problems (balanced parentheses, postfix evaluation, undo, min stack, queue from
two stacks and stack from queues); the **monotonic stack** (next greater element) and
**monotonic deque** (sliding-window maximum); and the architect angle, queues as buffers
between systems (producer–consumer, blocking queues, a pointer to the
[messaging notes](backend-and-messaging/messaging-and-event-driven-architecture.md)).

### 3. Binary search, two pointers, sliding window
Quick to learn, constantly used in coding rounds, and binary search already appears in
the heap and graph notes. Covers: the binary-search invariant and its off-by-one traps,
first/last occurrence, rotated arrays, *binary search on the answer*; two pointers on
sorted data; fixed and variable sliding windows (the maximum-in-window case uses the deque
from step 2); a pattern-recognition playbook.

### 4. Recursion and backtracking
Builds on the recursion already met in tree and graph DFS. Covers: the call stack, base
case and recursion trees, the cost of recursion, and the **choose → explore → undo**
template for subsets, permutations, combinations, N-queens and word search, with
pruning. Ends by showing that repeated work in a recursion tree is exactly what DP fixes.

### 5. Dynamic programming
Recursion plus remembering results, so it follows step 4. Covers: overlapping
subproblems and optimal substructure, top-down (memoisation) vs bottom-up (tabulation),
designing the state, 1-D problems (climbing stairs, house robber), knapsack, coin
change, grids, LCS / edit distance, LIS, space optimisation, and a recognition playbook.

### 6. Data structures inside real systems *(written 2026-10-14, ahead of steps 2–5)*
The architect-facing step, tying into [data architecture](data/data-architecture.md) and
[distributed systems](notes/distributed-systems-interview-notes.md). Covers: **B+ trees**
(why databases use wide nodes), **LSM trees** (memtable, SSTables, compaction), **Bloom
filters**, **consistent hashing** (the ring, virtual nodes), **skip lists** and **Merkle
trees**, each as problem → idea → how it works → trade-off, with the interview questions
architects actually get. May be split into more than one file if it grows.

## Flexibility

- If **architect interviews** come before coding rounds, jump to step 6 right after
  step 1 (it is already written).
- **Stacks and queues** can move earlier or later freely; they have no prerequisites.
  They sit at step 2 only because steps 3 and 4 use them.
- **Hash table internals** (collisions, resizing, treeification) can slot in before step
  3 or step 6; it completes the `HashMap` vs `TreeMap` comparison in the trees notes and
  leads into consistent hashing.
- Deferred extensions are tracked in [../to-be-added.md](../to-be-added.md) (for example
  Bellman–Ford, strongly connected components, A\*, segment trees).
