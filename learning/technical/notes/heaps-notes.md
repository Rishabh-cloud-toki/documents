# Heaps Notes: Priority Queues, Sift Up and Down, Build-Heap

Oct 9, 2026 · @Rishabh Toki

The teaching companion to [heaps.md](../data-structures/heaps.md). Same order as the
[trees notes](trees-notes.md): **the problem** → **the idea** → **how it works** →
**why it works** → **the trade-off**, then questions to check yourself. Code is Java.

> **Status: preview — not yet read.** Moved here from the heap section of the trees
> notes. This is a map of what to expect so the reading lands quickly; it will be
> expanded as you work through the topic.

Where heaps sit: a binary heap is a **complete binary tree** (see
[the map of trees](trees-notes.md#the-map-how-all-the-trees-relate)), but it behaves
very differently from the search trees — it lives in an array, and it keeps only the
minimum or maximum at the top. Rotations and balance factors do not appear here; the
tools are *sift up* and *sift down*.

## Contents

1. [The problem](#the-problem)
2. [The idea](#the-idea)
3. [The array trick](#the-array-trick)
4. [How the operations work](#how-the-operations-work)
5. [Java: PriorityQueue](#java-priorityqueue)
6. [Classic uses](#classic-uses)
7. [The trade-off](#the-trade-off)
8. [Check your understanding](#check-your-understanding)

---

## The problem

Many tasks only ever ask for **the smallest (or largest) item right now**: the next
job to run, the closest unvisited node in Dijkstra's algorithm, the top 10 scores.
A sorted array makes insertion `O(n)`. A balanced BST works in `O(log n)` but does
far more than needed — it keeps *everything* sorted when you only care about the
extreme.

## The idea

A **binary heap** only guarantees that the extreme item is at the top. Two rules:

1. **Shape:** it is a **complete** binary tree (every level full, the last filled
   left to right).
2. **Order (heap property):** in a **min-heap** every parent ≤ its children, so the
   root is the minimum. In a **max-heap** every parent ≥ its children.

A heap is **not** a BST. Siblings have no order relative to each other, and an
in-order walk is not sorted. It promises less, so it can do it more cheaply.

## The array trick

Because the tree is *complete*, there are no gaps, so it can live in a plain array
with no pointers. For a 0-indexed element `i`:

```
parent(i) = (i − 1) / 2
left(i)   = 2i + 1
right(i)  = 2i + 2
```

```
index:   0  1  2  3  4
array: [ 2, 5, 3, 9, 6 ]        as a tree:      2
                                              /   \
                                             5     3
                                            / \
                                           9   6
```

Check it: the children of index 1 (value 5) are indexes 3 and 4 (9 and 6), and the
parent of index 4 is `(4 − 1) / 2 = 1`.

## How the operations work

**`add`:** put the new item at the end (keeps the tree complete), then **sift up**:
while it is smaller than its parent, swap with the parent.

Example, min-heap `[2, 5, 3, 9, 6]`, add 1:

```
append → [2, 5, 3, 9, 6, 1]     1 is at index 5, parent index 2 (value 3): swap
       → [2, 5, 1, 9, 6, 3]     1 is at index 2, parent index 0 (value 2): swap
       → [1, 5, 2, 9, 6, 3]     1 is the root: done
```

**`poll` (remove the minimum):** take the root; move the **last** item into the root
(keeps the tree complete), then **sift down**: while it is bigger than a child, swap
with the *smaller* child.

```
[1, 5, 2, 9, 6, 3]  → remove 1, move 3 to the root → [3, 5, 2, 9, 6]
children of index 0 are 5 and 2: swap with the smaller (2) → [2, 5, 3, 9, 6]  done
```

| Operation | Cost |
|---|---|
| `peek` (see the min) | `O(1)`: it is `a[0]` |
| `add` | `O(log n)`: climbs at most the height |
| `poll` | `O(log n)`: descends at most the height |
| **build-heap** from `n` items | **`O(n)`** |
| find an arbitrary item | `O(n)`: no ordering to exploit |
| heap sort | `O(n log n)`, in place |

**Why build-heap is `O(n)`, not `O(n log n)`:** sift down from index `n/2 − 1` back
to 0. Half the nodes are leaves and need no work; a quarter sift down at most one
level; an eighth at most two. The total, `n/4·1 + n/8·2 + n/16·3 + …`, converges to
a constant times `n`.

## Java: PriorityQueue

- A binary **min**-heap by default; pass `Comparator.reverseOrder()` for a max-heap.
- `offer`/`poll` are `O(log n)`, `peek` is `O(1)`, `remove(Object)` is `O(n)`.
- **Iterating it does not give sorted order**; only repeated `poll()` does.
- Not thread-safe; use `PriorityBlockingQueue`.

## Classic uses

- **Top-K:** keep a min-heap of size K; any item bigger than the root replaces it.
  `O(n log K)`.
- Merge K sorted lists; Dijkstra's and Prim's algorithms.
- Task schedulers and event simulation.
- Running median (a max-heap and a min-heap together).

## The trade-off

| | Heap | Balanced BST |
|---|---|---|
| Find min | `O(1)` | `O(log n)` |
| Insert | `O(log n)` | `O(log n)` |
| Find an arbitrary key | `O(n)` | `O(log n)` |
| Sorted iteration | No | Yes |
| Memory | Compact array, no pointers | Pointers per node |

A heap is the right tool exactly when you only ever need the extreme.

## Check your understanding

*(After you read it.)*

1. Why is a heap not a BST?
2. Why can a complete tree be stored in an array but an arbitrary tree cannot?
3. Walk through adding 0 to the min-heap `[2, 5, 3, 9, 6]`.
4. Why is build-heap `O(n)` and not `O(n log n)`?
5. How would you find the 3 largest items in a stream of a million numbers using
   O(3) memory?
