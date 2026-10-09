# Heaps Notes: From the Simple Idea to the Deep Dive

Oct 9, 2026 · @Rishabh Toki

The teaching companion to [heaps.md](../data-structures/heaps.md). A heap is part of
the tree family (see [the map of trees](trees-notes.md#the-map-how-all-the-trees-relate)),
but it behaves very differently from the search trees: it lives in an array, it keeps
only the minimum or maximum at the top, and it uses *sift up* and *sift down* instead
of rotations.

> **Status: preview — not yet read.** Written so the reading lands quickly; expand it
> as you work through the topic.

## How this file is organised

The same example is used throughout, so each level builds on the one before.

| Level | Goal | What you get |
|---|---|---|
| [**Level 1: The simple version**](#level-1-the-simple-version) | Know what a heap is and picture it | Plain words, one small example, no comparisons or jargon you have not met |
| [**Level 2: How it works**](#level-2-how-it-works) | Do it by hand | Step-by-step traces of insert, remove and build, with short code |
| [**Level 3: Deep dive**](#level-3-deep-dive) | Know why, and when to use it | Complexity reasoning, alternatives, heap sort, `PriorityQueue`, real uses |
| [**Level 4: Working with `PriorityQueue`**](#level-4-working-with-priorityqueue) | Use it on real objects | Comparators, tie-breaking, mistakes to avoid, heap sort with code |
| [**Level 5: Classic problems**](#level-5-classic-problems) | Solve the standard problems | Kth largest, top-K frequent, merge K sorted lists, and which heap to pick |
| [**Level 6: Advanced patterns**](#level-6-advanced-patterns) | Combine heaps with other ideas | Two heaps (median), scheduling, greedy with heaps, lazy deletion |
| [**Level 7: Interview playbook**](#level-7-interview-playbook) | Recognise and choose | Spotting heap problems, min vs max rules, a mixed practice set |

**The running example (Levels 1–3):** the numbers `10, 4, 15, 2, 8, 20`.
Levels 4–7 use small problem-specific examples.

> **Dijkstra's shortest-path algorithm is deliberately not in this file.** It needs
> graph basics (nodes, edges, weights), so it is saved for the graphs topic, where it
> builds on Level 6's *lazy deletion*. It is recorded in
> [to-be-added.md](../../to-be-added.md#data-structures--graphs) so it is not forgotten.

---

## Level 1: The simple version

### What is a heap?

A **heap** is a way of arranging items so that **the smallest one (or the largest one)
is always right at the top, instantly available.**

- **Min-heap:** the smallest item is at the top.
- **Max-heap:** the largest item is at the top.

Say you have the numbers `10, 4, 15, 2, 8, 20` and you keep needing the smallest one.
You could search through all of them every time. A min-heap keeps the smallest at the
top, so you just read it. Think of a hospital waiting room where the most urgent
patient is always at the front, even as new patients arrive.

### Seeing it as a tree

Here are those six numbers arranged as a min-heap:

```
          2
        /   \
      10     4
     /  \   /
   15   20 8
```

Two things to notice:

1. The **root** holds `2`, the smallest value.
2. Every **parent is smaller than or equal to its children**: 2 ≤ 10 and 2 ≤ 4;
   10 ≤ 15 and 10 ≤ 20; 4 ≤ 8.

That second rule is the **heap property**. A **max-heap** follows the opposite rule:
every parent is greater than or equal to its children.

### A heap is not sorted

The heap property only compares a **parent with its own children**. It says nothing
about siblings or cousins. In the tree above, `10` sits to the left of `4` even though
10 is bigger, and the values read left to right (`2, 10, 4, 15, 20, 8`) are not in
order.

The same six numbers can also form `[2, 4, 8, 10, 15, 20]`, which happens to be sorted,
and that is *also* a valid heap. A heap is **not unique and not fully sorted**; it
just guarantees that the top is the minimum.

### Is a heap a binary search tree (BST)?

No. This is a very common interview question.

| | Heap | Binary search tree |
|---|---|---|
| Main purpose | Quickly get the min (or max) | Quickly search for any value |
| The rule | Parent ≤ children (min-heap) | Left subtree < node < right subtree |
| Find the minimum | Instant (the root) | Walk down the left side |
| Search for an arbitrary value | Slow: may check every item | Fast, if the tree is balanced |

**Key takeaway:** a heap gives up searching and full ordering in exchange for getting
the minimum or maximum as cheaply as possible.

### Why "complete"?

A heap has a shape rule on top of the ordering rule: it must be a **complete binary
tree**.

- Every level is full, except possibly the last.
- The last level fills **from left to right**, with no gaps.

```
VALID shape                    INVALID shape
(last level filled from        (a gap on the left of the
 the left)                      last level)

       o                             o
      / \                           / \
     o   o                         o   o
    / \ /                           \
   o   o o                           o
```

Why this rule? Two reasons:

1. It keeps the tree **short**, only about `log₂ n` levels tall, so walking from top to
   bottom is quick.
2. With no gaps, the tree can be stored in a plain **array**, which is the next idea.

### How a heap is stored: an array

Because the shape has no gaps, we can write the tree out **level by level** into an
array and need no pointers at all:

```
index:   0   1   2   3   4   5
array: [ 2, 10,  4, 15, 20,  8 ]
```

For the item at index `i` (counting from 0):

```
parent(i)      = (i − 1) / 2      (whole-number division)
left child(i)  = 2i + 1
right child(i) = 2i + 2
```

Test it on index 1 (value 10):

- parent: `(1 − 1) / 2 = 0` → value 2 ✓
- left child: `2·1 + 1 = 3` → value 15 ✓
- right child: `2·1 + 2 = 4` → value 20 ✓

And on index 2 (value 4): parent `(2 − 1) / 2 = 0` (whole-number division) → value 2; left
child `2·2 + 1 = 5` → value 8; right child would be index 6, which is past the end of the
array, so there is none.

### What can you do with a heap?

| Operation | What it does | Cost |
|---|---|---|
| **Peek** | Read the top without removing it | Instant, `O(1)` |
| **Insert** | Add a new value | `O(log n)` |
| **Extract min / max** | Remove and return the top | `O(log n)` |
| **Build heap** | Turn an unordered array into a heap | `O(n)` |
| **Search** | Find an arbitrary value | `O(n)` |

You do not need to know *why* those costs are what they are yet. Level 2 shows how
insert and extract work, and Level 3 explains the costs.

### Level 1 check

1. In `[2, 10, 4, 15, 20, 8]`, what is the parent of the value `8`? (Find its index first.)
2. Is `[3, 1, 2]` a valid min-heap? Why or why not?
3. Is `[1, 3, 2, 7, 4]` a valid min-heap? It is not sorted: does that matter?
4. Give two differences between a heap and a BST.
5. Why can a heap be stored in an array, while an arbitrary tree cannot?

---

## Level 2: How it works

Every change to a heap does the same thing in two steps: **(1)** make the change in the
way that keeps the shape complete, **(2)** repair the ordering by swapping along one path.

### Insert: sift up

**Idea:** put the new value in the next free spot at the end (this keeps the shape
complete), then move it **up** while it is smaller than its parent.

Insert `1` into `[2, 10, 4, 15, 20, 8]`:

```
append at the end:      [2, 10, 4, 15, 20, 8, 1]     1 is at index 6; parent index (6−1)/2 = 2 (value 4)
1 < 4, swap:            [2, 10, 1, 15, 20, 8, 4]     1 is at index 2; parent index 0 (value 2)
1 < 2, swap:            [1, 10, 2, 15, 20, 8, 4]     1 is at index 0: it is the root, so stop
```

As a tree, the result is:

```
          1
        /   \
      10     2
     /  \   / \
   15   20 8   4
```

Every parent is still ≤ its children. The new value only ever swapped with its
*parent*, so it travelled up one path.

### Remove the minimum: sift down

**Idea:** the minimum is the root. You cannot just delete it (that leaves a hole), so:
take the root out, move the **last** item into the root (keeps the shape complete), then
move it **down** while it is bigger than a child, always swapping with the **smaller**
child.

Remove the minimum from `[1, 10, 2, 15, 20, 8, 4]`:

```
take out 1, move last item (4) to the root:   [4, 10, 2, 15, 20, 8]
children of index 0 are 10 and 2; 2 is smaller and 4 > 2, so swap:
                                              [2, 10, 4, 15, 20, 8]
4 is now at index 2; its only child is index 5 (value 8); 4 < 8, so stop.
```

We are back to the original heap. Why swap with the *smaller* child? Because that child
becomes the new parent of the other one, so it must be the smaller of the two to keep
"parent ≤ child" true.

### In code

```java
class MinHeap {
    int[] a = new int[16];
    int size = 0;

    void add(int x) {
        a[size] = x;                  // 1. put it in the next free spot
        siftUp(size);                 // 2. repair upward
        size++;
    }

    int poll() {                      // assumes size > 0
        int min = a[0];
        a[0] = a[--size];             // 1. move the last item to the root
        siftDown(0);                  // 2. repair downward
        return min;
    }

    void siftUp(int i) {
        while (i > 0) {
            int p = (i - 1) / 2;
            if (a[i] >= a[p]) break;  // already ≥ parent: done
            swap(i, p);
            i = p;
        }
    }

    void siftDown(int i) {
        while (true) {
            int l = 2 * i + 1, r = 2 * i + 2, smallest = i;
            if (l < size && a[l] < a[smallest]) smallest = l;
            if (r < size && a[r] < a[smallest]) smallest = r;
            if (smallest == i) break; // no child is smaller: done
            swap(i, smallest);
            i = smallest;
        }
    }

    void swap(int i, int j) { int t = a[i]; a[i] = a[j]; a[j] = t; }
}
```

(The array would need to grow when it fills up; omitted to keep the idea visible.)

### Build a heap from an unordered array

You can add the numbers one at a time, but there is a faster way (**heapify**): treat
the array as an already-shaped tree and fix it from the bottom up. Start at the **last
parent**, index `n/2 − 1`, and sift each one down, moving back to index 0.

Build a heap from `[10, 4, 15, 2, 8, 20]` (`n = 6`, so start at index `6/2 − 1 = 2`):

```
start:                [10,  4, 15,  2,  8, 20]
index 2 (15): its only child is 20 (index 5); 15 < 20, nothing to do.
index 1 (4):  children are 2 (index 3) and 8 (index 4); smaller is 2; 4 > 2, swap
                      [10,  2, 15,  4,  8, 20]
index 0 (10): children are 2 and 15; smaller is 2; 10 > 2, swap
                      [ 2, 10, 15,  4,  8, 20]
              10 is now at index 1; its children are 4 and 8; smaller is 4; swap
                      [ 2,  4, 15, 10,  8, 20]
              10 is at index 3, which has no children: done.
```

Result: `[2, 4, 15, 10, 8, 20]`. As a tree, `2` is the root with children `4` and `15`; `4`
has children `10` and `8`; `15` has the child `20`. Every parent ≤ its children ✓.

### Java: `PriorityQueue` (a first look)

You rarely write the heap yourself. Java's `PriorityQueue` is a ready-made binary heap:

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();       // min-heap
pq.offer(10);  pq.offer(4);  pq.offer(15);
pq.peek();    // 4   (look at the minimum, do not remove)
pq.poll();    // 4   (remove and return the minimum)

PriorityQueue<Integer> max = new PriorityQueue<>(Comparator.reverseOrder());  // max-heap
```

### Level 2 check

1. Insert `3` into `[2, 10, 4, 15, 20, 8]`. Show the array after each swap.
2. Remove the minimum from `[2, 10, 4, 15, 20, 8]`. Show every step.
3. Why does `poll` move the *last* item to the root instead of, say, pulling up the
   smaller child repeatedly?
4. Heapify `[7, 3, 9, 1, 5]` and write the array after each swap.
5. When sifting down, why swap with the smaller child rather than the larger?

---

## Level 3: Deep dive

### The problem heaps solve, compared with the alternatives

Many tasks only ever ask for **the smallest (or largest) item right now**, over and over,
while new items keep arriving: the next job a scheduler should run, the closest
unvisited place in a route-finding algorithm (Dijkstra's shortest-path algorithm), the
top 10 scores. What are the options for storing the items?

| Structure | Insert | Get the min | Verdict |
|---|---|---|---|
| Unsorted list | `O(1)` | `O(n)`, scan everything | Slow to read |
| Sorted array | `O(n)`, shift items to make room | `O(1)` | Slow to write |
| Balanced BST (AVL / Red-Black) | `O(log n)` | `O(log n)` | Works, but keeps *everything* sorted when you only need the extreme |
| **Binary heap** | **`O(log n)`** | **`O(1)`** | Cheapest for exactly this job |

The heap wins by promising *less*: only the top is guaranteed, not the order of the rest.

### Why the operations are `O(log n)`

Insert and extract each move along **one path**, swapping at most once per level, so
their cost is bounded by the **height** of the tree. A complete tree with `n` nodes is
as short as a tree can be: height `⌊log₂ n⌋`. (Why a tree with `n` nodes has height about
`log₂ n`: [why searches take log n steps](trees-notes.md#why-searches-take-log-n-steps).)
So `add` and `poll` are `O(log n)`, and `peek` is `O(1)` because it only reads `a[0]`.

The completeness rule is what guarantees the short height *without any rebalancing*.
A search tree needs rotations to stay short; a heap gets it free from its shape.

### Why build-heap is `O(n)`, not `O(n log n)`

Building by `n` inserts costs `O(n log n)`. Heapify is cheaper because the work is
concentrated where the nodes are:

- About **half** the nodes are leaves and need no work at all.
- About a **quarter** are one level above the leaves and sift down at most 1 level.
- An **eighth** sift down at most 2 levels, and so on up to the single root, which may
  sift down `log n` levels.

Total work ≈ `n/4·1 + n/8·2 + n/16·3 + …`, which adds up to a constant multiple of `n`.
The few expensive nodes are outweighed by the huge number of cheap ones.

### Heap sort

A heap can sort in place:

1. Build a **max-heap** from the array, `O(n)`.
2. Swap the root (the maximum) with the last element, and treat the array as one item
   shorter.
3. Sift the new root down to restore the heap, `O(log n)`.
4. Repeat until nothing is left.

Total `O(n log n)`, with `O(1)` extra memory. It is **not stable** (equal items may be
reordered).

### `PriorityQueue` in detail

- A binary **min**-heap by default; pass a `Comparator` (for example
  `Comparator.reverseOrder()`) for a max-heap or a custom priority.
- `offer` / `poll`: `O(log n)`. `peek`: `O(1)`. `remove(Object)`: `O(n)` (it must find
  the item first).
- Building one from an existing collection uses heapify, `O(n)`, rather than `n` inserts.
- **Iterating it does not give sorted order.** You see the internal array order; only
  repeated `poll()` returns items in order.
- `null` elements are not allowed. Items with equal priority come out in no guaranteed order.
- Not thread-safe; use `PriorityBlockingQueue` across threads.

### Classic uses

- **Top-K:** keep a min-heap of size `K`. For each new item, add it, and if the size
  exceeds `K`, `poll()` the smallest. What remains is the `K` largest. `O(n log K)`
  time, `O(K)` memory, even for a stream you cannot store.

  ```java
  PriorityQueue<Integer> top = new PriorityQueue<>();
  for (int x : stream) {
      top.offer(x);
      if (top.size() > k) top.poll();      // drop the smallest of the K+1
  }
  ```

- **Merge `K` sorted lists:** keep the current head of each list in a min-heap.
- **Dijkstra's and Prim's algorithms:** repeatedly take the "closest" unvisited item.
  (Saved for the graphs topic; see the note near the top of this file.)
- **Schedulers and event simulation:** always run the earliest or most urgent task.
- **Running median:** a max-heap for the lower half and a min-heap for the upper half.

### Heap vs balanced BST

| | Heap | Balanced BST |
|---|---|---|
| Find min | `O(1)` | `O(log n)` |
| Insert | `O(log n)` | `O(log n)` |
| Find an arbitrary key | `O(n)` | `O(log n)` |
| Sorted iteration | No | Yes |
| Memory | Compact array, no pointers | Pointers per node |
| Needs rebalancing | No (shape does the work) | Yes (rotations) |

A heap is the right tool exactly when you only ever need the extreme.

### Level 3 check

1. Why does a complete tree guarantee `O(log n)` operations without any rebalancing?
2. Why is build-heap `O(n)` and not `O(n log n)`?
3. How would you find the 3 largest numbers in a stream of a million values using
   `O(3)` memory?
4. Why does a `PriorityQueue` iterate in a different order from the one `poll()` uses?
5. You need "the minimum" *and* "everything between 10 and 20". Heap or `TreeMap`? Why?
6. Why is heap sort not stable?

---

## Level 4: Working with PriorityQueue

### The simple version

A `PriorityQueue` decides who comes out first by **comparing** items. For numbers and
strings it already knows how. For your own objects you tell it how, with a
**comparator**.

```java
record Task(String name, int priority) {}

// smallest priority number comes out first (a min-heap on priority)
PriorityQueue<Task> pq = new PriorityQueue<>(Comparator.comparingInt(Task::priority));

pq.offer(new Task("email", 3));
pq.offer(new Task("deploy", 1));
pq.offer(new Task("lunch", 2));
pq.poll();      // Task[name=deploy, priority=1]
```

### Choosing the order

| You want | Comparator |
|---|---|
| Smallest priority number first | `Comparator.comparingInt(Task::priority)` |
| **Largest** priority number first | `Comparator.comparingInt(Task::priority).reversed()` |
| Priority, then name as a tie-breaker | `Comparator.comparingInt(Task::priority).thenComparing(Task::name)` |
| Plain numbers, largest first | `Comparator.reverseOrder()` |

### Ties: equal priorities have no guaranteed order

If three tasks all have priority 1, a `PriorityQueue` does **not** promise they come out
in the order you added them. A heap only guarantees the top; among equals, anything may
come first. If you need first-in-first-out among equals, add a **sequence number** to
the comparison:

```java
record Job(String name, int priority, long seq) {}

long counter = 0;
PriorityQueue<Job> pq = new PriorityQueue<>(
    Comparator.comparingInt(Job::priority).thenComparingLong(Job::seq));

pq.offer(new Job("A", 1, counter++));
pq.offer(new Job("B", 1, counter++));   // A is always polled before B
```

### Mistakes to avoid

- **Comparing with subtraction.** `(a, b) -> a - b` can overflow for large or negative
  values and silently give the wrong order. Use `Integer.compare(a, b)` or
  `Comparator.comparingInt`.
- **Changing an item's priority after adding it.** The heap does not notice, so the
  ordering breaks (the same trap as mutating a `TreeMap` key). Remove and re-add, or use
  the *lazy deletion* trick in Level 6.
- **Expecting sorted iteration.** Looping over a `PriorityQueue` shows the internal
  array order, not sorted order. Call `poll()` repeatedly.
- **Peeking an empty queue.** `peek()` and `poll()` return `null`; they do not throw.
  Check `isEmpty()` first.
- **Comparing boxed `Integer`s with `==`.** Use `.equals`, or compare their values.

### Heap sort, with code and a trace

Heap sort sorts an array **in place** using a heap built inside the array itself. To
sort in **ascending** order it uses a **max-heap**: the largest item is repeatedly moved
to the end, where it belongs.

```java
void heapSort(int[] a) {
    int n = a.length;
    for (int i = n / 2 - 1; i >= 0; i--)        // 1. build a max-heap (heapify)
        siftDownMax(a, i, n);
    for (int end = n - 1; end > 0; end--) {     // 2. repeatedly:
        swap(a, 0, end);                        //    the max goes to its final place
        siftDownMax(a, 0, end);                 //    repair the shrunken heap
    }
}

void siftDownMax(int[] a, int i, int size) {
    while (true) {
        int l = 2 * i + 1, r = 2 * i + 2, largest = i;
        if (l < size && a[l] > a[largest]) largest = l;
        if (r < size && a[r] > a[largest]) largest = r;
        if (largest == i) break;
        swap(a, i, largest);
        i = largest;
    }
}
```

Trace on `[4, 10, 3, 5, 1]`:

```
build the max-heap:
  index 1 (10): children 5 and 1, it is already the largest        [4, 10, 3, 5, 1]
  index 0 (4):  children 10 and 3; swap with 10                     [10, 4, 3, 5, 1]
                4 is now at index 1; children 5 and 1; swap with 5  [10, 5, 3, 4, 1]

sort:
  swap root with last, heap size 4:   [1, 5, 3, 4 | 10]   sift 1 down → [5, 4, 3, 1 | 10]
  swap root with last, heap size 3:   [1, 4, 3 | 5, 10]   sift 1 down → [4, 1, 3 | 5, 10]
  swap root with last, heap size 2:   [3, 1 | 4, 5, 10]   3 ≥ 1, nothing to fix
  swap root with last, heap size 1:   [1 | 3, 4, 5, 10]   done

result: [1, 3, 4, 5, 10]
```

The `|` separates the shrinking heap (left) from the sorted part (right). It is
`O(n log n)`, uses no extra memory, and is not stable.

### Level 4 check

1. Write a comparator that orders `Task`s by priority descending, then by name ascending.
2. Three jobs with the same priority are added in the order A, B, C. Why might they come
   out as B, A, C, and how do you guarantee A, B, C?
3. Why is `(a, b) -> a - b` a risky comparator?
4. Heap-sort `[3, 1, 2]` by hand and show the array after each swap.
5. Why does ascending heap sort use a *max*-heap?

---

## Level 5: Classic problems

For each problem: the question in plain words, the idea in one line, the code, and why
that kind of heap.

### Kth largest element

**Question:** given numbers, find the kth largest (for `k = 2` in `[3, 2, 1, 5, 6, 4]`
the answer is `5`).

**Idea:** keep a **min-heap holding the k largest numbers seen so far**. When it grows
to `k + 1`, throw out the smallest. At the end the smallest of the k survivors, the
root, is the kth largest.

```java
int kthLargest(int[] nums, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>();      // min-heap
    for (int x : nums) {
        pq.offer(x);
        if (pq.size() > k) pq.poll();                       // evict the smallest
    }
    return pq.peek();
}
```

Trace for `[3, 2, 1, 5, 6, 4]`, `k = 2`:

```
add 3 → [3]        add 2 → [2, 3]       add 1 → [1, 2, 3] → evict 1 → [2, 3]
add 5 → [2, 3, 5] → evict 2 → [3, 5]    add 6 → evict 3 → [5, 6]
add 4 → evict 4 → [5, 6]                answer = root = 5
```

**Why a min-heap for the largest?** Because the thing we must repeatedly find and
throw out is the *smallest* of the current top-k, and a min-heap hands that over in
`O(1)`. **Cost:** `O(n log k)` time, `O(k)` memory. Sorting would be `O(n log n)`, and
this also works on a stream you cannot store.

### Top K frequent elements

**Question:** return the k values that occur most often in an array.

**Idea:** two steps. Count occurrences with a `HashMap`; then keep a **min-heap of size k
ordered by count**, evicting the least frequent whenever it grows past k.

```java
List<Integer> topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int x : nums) freq.merge(x, 1, Integer::sum);

    PriorityQueue<Map.Entry<Integer, Integer>> pq =
        new PriorityQueue<>(Map.Entry.comparingByValue());  // min-heap by count

    for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
        pq.offer(e);
        if (pq.size() > k) pq.poll();                       // drop the least frequent
    }

    List<Integer> result = new ArrayList<>();
    for (Map.Entry<Integer, Integer> e : pq) result.add(e.getKey());
    return result;                                          // not in sorted order
}
```

**Cost:** `O(n + m log k)`, where `m` is the number of distinct values. This is a
**heap plus a hash map**, a very common combination: the map does the counting, the
heap does the "best k".

### Merge K sorted lists

**Question:** given k lists, each already sorted, produce one sorted list.

**Idea:** at any moment the next output must be the smallest *front* item of the k lists.
Keep those k fronts in a **min-heap**. Take the smallest, append it to the result, and
push that list's next item.

```java
ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>(Comparator.comparingInt(n -> n.val));
    for (ListNode head : lists)
        if (head != null) pq.offer(head);

    ListNode dummy = new ListNode(0), tail = dummy;
    while (!pq.isEmpty()) {
        ListNode smallest = pq.poll();
        tail.next = smallest;
        tail = smallest;
        if (smallest.next != null) pq.offer(smallest.next);  // refill from the same list
    }
    return dummy.next;
}
```

For lists `[1,4,5]`, `[1,3,4]`, `[2,6]` the heap starts as the three heads `1, 1, 2`; each
`poll` yields the next output and is replaced by its successor. **Cost:** `O(N log k)`
for `N` items in total, because the heap never holds more than `k` items. Merging the
lists pairwise one after another would be slower, `O(N·k)`.

### Level 5 check

1. In "kth largest", why is the heap size capped at `k`, and what would go wrong if you
   let it grow?
2. How would you find the **kth smallest** element instead? Which heap type, and what
   gets evicted?
3. In "top K frequent", what are the two data structures and what is each one's job?
4. In "merge K lists", why does the heap hold at most `k` items, and what does that do
   to the cost?

---

## Level 6: Advanced patterns

### Two heaps: the median of a data stream

**Question:** numbers arrive one at a time; after each, report the median (the middle
value, or the average of the two middle values).

**Idea:** split the numbers seen so far into a **lower half** and an **upper half**. The
median sits at the border. Keep the lower half in a **max-heap** (its top is the largest
of the low numbers) and the upper half in a **min-heap** (its top is the smallest of the
high numbers). The two tops are exactly the middle elements.

```java
class MedianFinder {
    private final PriorityQueue<Integer> lower = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap
    private final PriorityQueue<Integer> upper = new PriorityQueue<>();                          // min-heap

    void addNum(int x) {
        lower.offer(x);
        upper.offer(lower.poll());             // move the largest of the lower half across
        if (upper.size() > lower.size())       // keep lower equal in size, or one bigger
            lower.offer(upper.poll());
    }

    double findMedian() {
        if (lower.size() > upper.size()) return lower.peek();       // odd count
        return (lower.peek() + upper.peek()) / 2.0;                  // even count
    }
}
```

Trace for the stream `5, 2, 8, 1`:

| After adding | lower (max-heap) | upper (min-heap) | Median |
|---|---|---|---|
| 5 | {5} | {} | 5 |
| 2 | {2} | {5} | (2 + 5) / 2 = 3.5 |
| 8 | {2, 5} | {8} | 5 |
| 1 | {1, 2} | {5, 8} | (2 + 5) / 2 = 3.5 |

**Cost:** `O(log n)` per number, `O(1)` for the median. Re-sorting after every number
would cost `O(n log n)` each time.

### Scheduling: always take the most urgent

A scheduler repeatedly needs "the task that must happen next". That is a heap on
deadline, priority, or finish time. Example, **minimum number of meeting rooms**:

**Idea:** sort the meetings by start time. Keep a **min-heap of the end times** of rooms
in use. For each meeting, if the earliest-ending room has already finished, reuse it
(pop); then add this meeting's end time. The heap size is the number of rooms needed.

```java
int minMeetingRooms(int[][] meetings) {                     // each = {start, end}
    Arrays.sort(meetings, Comparator.comparingInt(m -> m[0]));
    PriorityQueue<Integer> ends = new PriorityQueue<>();    // end times of rooms in use
    for (int[] m : meetings) {
        if (!ends.isEmpty() && ends.peek() <= m[0]) ends.poll();   // a room is free: reuse it
        ends.offer(m[1]);
    }
    return ends.size();
}
```

For `{0,30}, {5,10}, {15,20}`: after `{0,30}` the heap is `[30]`; `{5,10}` starts before 30,
so it needs a new room, `[10,30]`; `{15,20}` starts after 10, so it reuses that room,
`[20,30]`. Two rooms. **Cost:** `O(n log n)`.

### Greedy algorithms with heaps

A **greedy** algorithm makes the best-looking choice at every step. A heap supplies
"the best choice right now" cheaply. Example, **connect ropes at minimum cost**: joining
two ropes costs their combined length, and you must join them all into one.

**Idea:** long ropes should be re-counted as few times as possible, so always join the
**two shortest** first. This is the same idea behind Huffman coding.

```java
int minCostToConnect(int[] ropes) {
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    for (int r : ropes) pq.offer(r);
    int cost = 0;
    while (pq.size() > 1) {
        int merged = pq.poll() + pq.poll();        // the two shortest
        cost += merged;
        pq.offer(merged);                          // the new rope goes back in
    }
    return cost;
}
```

For `[4, 3, 2, 6]`: join 2+3 = 5 (cost 5), then 4+5 = 9 (cost 14), then 6+9 = 15
(cost 29). Total **29**. **Cost:** `O(n log n)`.

### Lazy deletion and stale entries

**The problem.** A `PriorityQueue` is bad at changing or removing something in the
middle: `remove(Object)` is `O(n)`, and changing a priority in place breaks the heap.

**The trick.** Don't remove or update. Instead **leave the old entry in the heap, mark it
stale, and skip it when it eventually reaches the top.** To update an item, simply add a
fresh entry with the new priority.

```java
record Entry(String id, int priority, int version) {}

Map<String, Integer> currentVersion = new HashMap<>();
PriorityQueue<Entry> pq = new PriorityQueue<>(Comparator.comparingInt(Entry::priority));

void upsert(String id, int priority) {
    int v = currentVersion.merge(id, 1, Integer::sum);   // bump the version: older entries go stale
    pq.offer(new Entry(id, priority, v));
}

void cancel(String id) {
    currentVersion.remove(id);                           // every entry for this id is now stale
}

Entry pollValid() {
    while (!pq.isEmpty()) {
        Entry e = pq.poll();
        Integer v = currentVersion.get(e.id());
        if (v != null && v == e.version()) return e;     // current: use it
        // otherwise stale: discard and keep looking
    }
    return null;
}
```

**Cost:** each entry is added once and removed at most once, so the work is still
`O(log n)` per entry, amortised. **Trade-off:** stale entries use memory until they
surface.

> **Remember for graphs:** this is exactly how Dijkstra's algorithm is normally written
> in Java. `PriorityQueue` has no "decrease priority", so a shorter route to a node is
> pushed as a *new* entry and the old, longer one is skipped when it comes up. That is why
> this pattern is here; the algorithm itself waits for the graphs topic.

### Level 6 check

1. In the two-heaps median, which heap is the max-heap and why? What invariant do the
   sizes keep?
2. Walk the stream `7, 3, 9` through `addNum` and give the median after each number.
3. In minimum meeting rooms, why sort by start time and keep end times in the heap?
4. Why is joining the two shortest ropes first optimal, in plain words?
5. What is lazy deletion, and why is it needed with a `PriorityQueue`?

---

## Level 7: Interview playbook

### Recognising a heap problem

| If the problem says… | Think |
|---|---|
| "kth largest / smallest" | Heap of size k |
| "top K", "K most frequent", "K closest" | Heap of size k, plus a map or distance function |
| "merge K sorted ..." | Min-heap of the k current fronts |
| "median", "running / streaming" | Two heaps |
| "schedule", "earliest deadline", "next available", "meeting rooms" | Heap on time or priority |
| "combine two smallest, repeatedly" / "minimum total cost" | Greedy with a min-heap |
| "next smallest / best, over and over, while items change" | A priority queue |

The common thread: **you repeatedly need the best (extreme) item from a collection that
keeps changing.**

### Min-heap or max-heap?

| Task | Heap | Why |
|---|---|---|
| k **largest** | **Min**-heap of size k | The root is the smallest of the k, the one to evict |
| k **smallest** | **Max**-heap of size k | The root is the largest of the k, the one to evict |
| Median | Max-heap (lower half) + min-heap (upper half) | The two roots are the middle values |
| Always process the smallest next | Min-heap | |
| Sort ascending in place | Max-heap (heap sort) | The max goes to the end |

**Rule of thumb:** for "top K", use the *opposite* heap to the one you'd expect, because
the root must be the item you throw away.

### Optimising time and space

| Situation | Better approach |
|---|---|
| `k` is small compared with `n` | Heap of size k: `O(n log k)`, `O(k)` memory |
| `k` is close to `n` | Just sort: `O(n log n)`, simpler |
| You need the min or max **once** | Scan: `O(n)`, no heap needed |
| You have all items up front and want a heap | Heapify: `O(n)`, not `n` inserts at `O(n log n)` |
| Single kth element, no stream, average speed matters | Quickselect: `O(n)` average |
| Data arrives as a stream you cannot store | Heap of size k (you never need all of it) |

### Combining heaps with other tools

- **Heap + hash map:** count with a map, pick the best with a heap (top-K frequent);
  track current versions for lazy deletion.
- **Heap + sorting:** sort first, then use a heap for "what's active now" (meeting
  rooms).
- **Heap vs binary search:** for something like "kth smallest in a sorted matrix", a
  heap solves it in `O(k log n)` by repeatedly taking the smallest front; binary
  searching on the *value* solves it in `O(n log(range))`. Know both and pick by the
  constraints: small `k` favours the heap, a huge `k` favours binary search.

### Mixed practice set

For each, name the pattern and the heap type *before* coding.

| # | Problem | Pattern | Heap |
|---|---|---|---|
| 1 | Kth largest element in a stream | Size-k heap | Min |
| 2 | K closest points to the origin | Size-k heap on distance | Max |
| 3 | Sort characters by frequency | Map + heap | Max (by count) |
| 4 | Merge K sorted arrays | Heap of fronts | Min |
| 5 | Median of a data stream | Two heaps | Max + Min |
| 6 | Minimum meeting rooms | Sort + heap of end times | Min |
| 7 | Connect ropes at minimum cost | Greedy | Min |
| 8 | Task scheduler with cooldown | Greedy with a count heap | Max |
| 9 | Kth smallest in a sorted matrix | Heap vs binary search | Min |
| 10 | Reorganise a string so no two letters touch | Greedy | Max (by count) |

### How to approach a new problem

1. **Brute force first.** Say it aloud (usually "sort everything", `O(n log n)`).
2. **What do I need repeatedly?** If it's the extreme of a changing set, think heap.
3. **How big is k, and does the data stream?** That picks heap size and whether sorting
   still wins.
4. **Which heap, and what is the comparator?** Use the min/max table above.
5. **State the cost:** `O(n log k)` time, `O(k)` space, and compare it to the brute force.
6. **Check the edges:** empty input, `k > n`, ties, duplicates.

### Level 7 check

1. You are asked for "the 10 smallest values in a stream of a billion numbers". Which
   heap, what size, and what are the time and space costs?
2. Why does top-K use the *opposite* heap type to the obvious one?
3. When is sorting better than a heap for "kth largest"?
4. Which of the ten practice problems use a hash map, and which use sorting first?
5. Pick any three practice problems and, without coding, say the heap type, what's
   stored in it, and the cost.
