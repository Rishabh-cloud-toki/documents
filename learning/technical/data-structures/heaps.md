# Heaps — Reference

Quick reference for binary heaps and priority queues. The step-by-step explanations are
in the layered companion, [heaps-notes.md](../notes/heaps-notes.md):
**Level 1** (the simple version), **Level 2** (how it works, with traces) and
**Level 3** (the deep dive). A heap belongs to the tree family but sits *beside* the
search trees, not under them — see [the map of trees](trees.md#the-map-of-trees).

> **Status: preview — not yet read.** To be expanded as you work through the topic.

Running example: `10, 4, 15, 2, 8, 20` as the min-heap `[2, 10, 4, 15, 20, 8]`.

The notes continue past the fundamentals: **Level 4** working with `PriorityQueue`,
**Level 5** classic problems, **Level 6** advanced patterns and **Level 7** an interview
playbook. Sections 6–9 below summarise them. Dijkstra's algorithm is deliberately saved
for the graphs topic ([backlog](../../to-be-added.md#data-structures--graphs)).

## Contents

1. [What a heap is](#1-what-a-heap-is)
2. [Shape and storage](#2-shape-and-storage)
3. [Operations and cost](#3-operations-and-cost)
4. [Java PriorityQueue](#4-java-priorityqueue)
5. [Why and when](#5-why-and-when)
6. [PriorityQueue in practice](#6-priorityqueue-in-practice)
7. [Classic problems](#7-classic-problems)
8. [Advanced patterns](#8-advanced-patterns)
9. [Interview playbook](#9-interview-playbook)

---

## 1. What a heap is

A structure that keeps the **smallest (min-heap) or largest (max-heap) item at the
top**, instantly available. **Heap property:** every parent ≤ its children (min-heap).
It compares only parent with child, so it is **not sorted** and **not a BST**.

```
          2            array: [ 2, 10, 4, 15, 20, 8 ]
        /   \
      10     4         10 sits left of 4 although it is bigger:
     /  \   /          siblings are unordered
   15   20 8
```

> Deep-dives: [what is a heap](../notes/heaps-notes.md#what-is-a-heap) ·
> [seeing it as a tree](../notes/heaps-notes.md#seeing-it-as-a-tree) ·
> [a heap is not sorted](../notes/heaps-notes.md#a-heap-is-not-sorted) ·
> [heap vs BST](../notes/heaps-notes.md#is-a-heap-a-binary-search-tree-bst)

## 2. Shape and storage

A heap is a **complete binary tree**: every level full except possibly the last, which
fills left to right. That keeps it short (about `log₂ n` levels) and gap-free, so it is
stored in an **array** with no pointers, 0-indexed:

```
parent(i) = (i − 1) / 2     left(i) = 2i + 1     right(i) = 2i + 2
```

> Deep-dives: [why "complete"](../notes/heaps-notes.md#why-complete) ·
> [the array layout, with worked checks](../notes/heaps-notes.md#how-a-heap-is-stored-an-array)

## 3. Operations and cost

| Operation | How | Cost |
|---|---|---|
| `peek` | return `a[0]` | `O(1)` |
| `add` | put at the end, **sift up** | `O(log n)` |
| `poll` | move the last item to the root, **sift down** (swap with the *smaller* child) | `O(log n)` |
| build-heap | sift down from index `n/2 − 1` to 0 | **`O(n)`** |
| find arbitrary item | scan | `O(n)` |
| heap sort | build a max-heap, then swap the root to the end and sift down, repeatedly | `O(n log n)`, in place, not stable |

> Deep-dives: [insert: sift up](../notes/heaps-notes.md#insert-sift-up) ·
> [remove: sift down](../notes/heaps-notes.md#remove-the-minimum-sift-down) ·
> [build a heap](../notes/heaps-notes.md#build-a-heap-from-an-unordered-array) ·
> [why `O(log n)`](../notes/heaps-notes.md#why-the-operations-are-olog-n) ·
> [why build-heap is `O(n)`](../notes/heaps-notes.md#why-build-heap-is-on-not-on-log-n) ·
> [heap sort](../notes/heaps-notes.md#heap-sort)

## 4. Java PriorityQueue

A binary **min**-heap by default (`Comparator.reverseOrder()` for max). `offer`/`poll`
`O(log n)`, `peek` `O(1)`, `remove(Object)` `O(n)`. **Iteration is not sorted**; only
repeated `poll()` is. No `null`s. Not thread-safe (`PriorityBlockingQueue`).

> Deep-dives: [a first look](../notes/heaps-notes.md#java-priorityqueue-a-first-look) ·
> [in detail](../notes/heaps-notes.md#priorityqueue-in-detail)

## 5. Why and when

Use a heap when you repeatedly need only the extreme: top-K (min-heap of size K,
`O(n log K)`), merge K sorted lists, Dijkstra / Prim, schedulers, running median.

| | Heap | Balanced BST |
|---|---|---|
| Find min | `O(1)` | `O(log n)` |
| Insert | `O(log n)` | `O(log n)` |
| Find arbitrary key | `O(n)` | `O(log n)` |
| Sorted iteration | No | Yes |
| Rebalancing | None: the shape does the work | Rotations |

> Deep-dives: [the problem, vs the alternatives](../notes/heaps-notes.md#the-problem-heaps-solve-compared-with-the-alternatives) ·
> [classic uses](../notes/heaps-notes.md#classic-uses) ·
> [heap vs BST](../notes/heaps-notes.md#heap-vs-balanced-bst) ·
> Check yourself: [Level 1](../notes/heaps-notes.md#level-1-check) ·
> [Level 2](../notes/heaps-notes.md#level-2-check) ·
> [Level 3](../notes/heaps-notes.md#level-3-check)

---

## 6. PriorityQueue in practice

Give your own objects an order with a **comparator**:
`new PriorityQueue<>(Comparator.comparingInt(Task::priority))` (add `.reversed()` for
largest-first, `.thenComparing(...)` for tie-breakers).

- **Equal priorities come out in no guaranteed order.** For first-in-first-out among
  equals, add a sequence number to the comparator.
- Don't use `(a, b) -> a - b` (overflow); don't change an item's priority after adding
  it; don't expect sorted iteration; `peek`/`poll` return `null` when empty.
- **Heap sort in place:** build a **max**-heap, then repeatedly swap the root to the end
  and sift down. `O(n log n)`, `O(1)` extra memory, not stable.

> Deep-dives: [comparators and ordering](../notes/heaps-notes.md#choosing-the-order) ·
> [ties](../notes/heaps-notes.md#ties-equal-priorities-have-no-guaranteed-order) ·
> [mistakes to avoid](../notes/heaps-notes.md#mistakes-to-avoid) ·
> [heap sort with code and a trace](../notes/heaps-notes.md#heap-sort-with-code-and-a-trace) ·
> [Level 4 check](../notes/heaps-notes.md#level-4-check)

## 7. Classic problems

| Problem | Heap | Idea | Cost |
|---|---|---|---|
| Kth largest | **Min**, size k | Evict the smallest; the root is the answer | `O(n log k)` |
| Top K frequent | **Min** by count, size k | Count with a `HashMap`, then keep the k best | `O(n + m log k)` |
| Merge K sorted lists | **Min** of the k fronts | Take the smallest, push its successor | `O(N log k)` |

> Deep-dives: [kth largest, with a trace](../notes/heaps-notes.md#kth-largest-element) ·
> [top K frequent](../notes/heaps-notes.md#top-k-frequent-elements) ·
> [merge K sorted lists](../notes/heaps-notes.md#merge-k-sorted-lists) ·
> [Level 5 check](../notes/heaps-notes.md#level-5-check)

## 8. Advanced patterns

| Pattern | Idea |
|---|---|
| **Two heaps** (median of a stream) | Max-heap for the lower half, min-heap for the upper half; keep sizes equal or lower one bigger; median from the tops. `O(log n)` per number |
| **Scheduling** (meeting rooms) | Sort by start; min-heap of end times; reuse the earliest-ending room if it is free |
| **Greedy** (connect ropes) | Always join the two shortest; a heap gives them cheaply |
| **Lazy deletion** | Don't remove or update in the heap: add a fresh entry and skip stale ones when they surface (`PriorityQueue.remove` is `O(n)`) |

Lazy deletion is how Dijkstra is normally written in Java (no decrease-key), which is
why it matters for the graphs topic.

> Deep-dives: [two heaps](../notes/heaps-notes.md#two-heaps-the-median-of-a-data-stream) ·
> [scheduling](../notes/heaps-notes.md#scheduling-always-take-the-most-urgent) ·
> [greedy](../notes/heaps-notes.md#greedy-algorithms-with-heaps) ·
> [lazy deletion](../notes/heaps-notes.md#lazy-deletion-and-stale-entries) ·
> [Level 6 check](../notes/heaps-notes.md#level-6-check)

## 9. Interview playbook

| Task | Heap |
|---|---|
| k **largest** | **Min**-heap of size k (the root is what you evict) |
| k **smallest** | **Max**-heap of size k |
| Median of a stream | Max-heap (lower half) + min-heap (upper half) |
| Merge K sorted / schedule by earliest | Min-heap |

**Trigger phrases:** "kth largest/smallest", "top K", "K closest", "merge K sorted",
"median / streaming", "schedule", "combine the two smallest". **Choose by size:** small
`k` ⇒ heap of size k (`O(n log k)`); `k` near `n` ⇒ just sort; need the min once ⇒ scan;
all items up front ⇒ heapify (`O(n)`).

> Deep-dives: [recognising a heap problem](../notes/heaps-notes.md#recognising-a-heap-problem) ·
> [min or max?](../notes/heaps-notes.md#min-heap-or-max-heap) ·
> [time and space trade-offs](../notes/heaps-notes.md#optimising-time-and-space) ·
> [combining heaps with other tools](../notes/heaps-notes.md#combining-heaps-with-other-tools) ·
> [a mixed practice set of ten](../notes/heaps-notes.md#mixed-practice-set) ·
> [how to approach a new problem](../notes/heaps-notes.md#how-to-approach-a-new-problem) ·
> [Level 7 check](../notes/heaps-notes.md#level-7-check)
