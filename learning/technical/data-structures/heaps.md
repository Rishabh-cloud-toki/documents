# Heaps — Reference

Quick reference for binary heaps and priority queues: the problem, the idea, the
numbers. Full explanations (array indexing, worked sift-up and sift-down, why
build-heap is `O(n)`, check-yourself questions) are in the deep-dive companion,
[heaps-notes.md](../notes/heaps-notes.md). Part of the tree family — see
[the map of trees](trees.md#the-map-of-trees), where the heap sits beside the search
trees, not under them.

> **Status: preview — not yet read.** Moved here from the trees reference; to be
> expanded as you work through the topic.

## Contents

1. [The problem](#1-the-problem)
2. [The idea](#2-the-idea)
3. [Operations and cost](#3-operations-and-cost)
4. [Java PriorityQueue](#4-java-priorityqueue)
5. [Uses](#5-uses)
6. [Heap vs balanced BST](#6-heap-vs-balanced-bst)

---

## 1. The problem

Many tasks only need the **smallest (or largest) item right now**: next job, nearest
node in Dijkstra, top-K scores. A sorted array inserts in `O(n)`; a balanced BST does
`O(log n)` but keeps *everything* sorted, which is more than needed.

> Deep-dive: [the problem](../notes/heaps-notes.md#the-problem)

## 2. The idea

A **binary heap** is a **complete** binary tree (all levels full except possibly the
last, filled left to right) with the **heap property**: min-heap ⇒ each parent ≤ its
children, so the root is the minimum (max-heap: the reverse). It is **not** a BST:
siblings are unordered and in-order traversal is not sorted.

Because it is complete, it is stored in an **array** with no pointers, 0-indexed:

```
parent(i) = (i − 1) / 2     left(i) = 2i + 1     right(i) = 2i + 2
```

> Deep-dives: [the idea](../notes/heaps-notes.md#the-idea) ·
> [the array trick](../notes/heaps-notes.md#the-array-trick)

## 3. Operations and cost

| Operation | How | Cost |
|---|---|---|
| `peek` | return `a[0]` | `O(1)` |
| `add` | append at the end, **sift up** | `O(log n)` |
| `poll` | move last to root, **sift down** (swap with the smaller child) | `O(log n)` |
| build-heap | sift down from index `n/2 − 1` to 0 | **`O(n)`** |
| find arbitrary item | scan | `O(n)` |
| heap sort | build-heap, then poll repeatedly | `O(n log n)`, in place |

> Deep-dive: [worked add and poll, and why build-heap is O(n)](../notes/heaps-notes.md#how-the-operations-work)

## 4. Java PriorityQueue

A binary **min**-heap by default (`Comparator.reverseOrder()` for max). `offer`/`poll`
`O(log n)`, `peek` `O(1)`, `remove(Object)` `O(n)`. **Iteration is not sorted**; only
repeated `poll()` is. Not thread-safe (`PriorityBlockingQueue`).

> Deep-dive: [PriorityQueue](../notes/heaps-notes.md#java-priorityqueue)

## 5. Uses

Top-K (min-heap of size K, `O(n log K)`), merge K sorted lists, Dijkstra / Prim,
schedulers and event simulation, running median (two heaps).

> Deep-dive: [classic uses](../notes/heaps-notes.md#classic-uses)

## 6. Heap vs balanced BST

| | Heap | Balanced BST |
|---|---|---|
| Find min | `O(1)` | `O(log n)` |
| Insert | `O(log n)` | `O(log n)` |
| Find arbitrary key | `O(n)` | `O(log n)` |
| Sorted iteration | No | Yes |
| Memory | Compact array | Pointers per node |

A heap is right exactly when you only ever need the extreme.

> Deep-dive: [the trade-off](../notes/heaps-notes.md#the-trade-off) ·
> [check-yourself questions](../notes/heaps-notes.md#check-your-understanding)
