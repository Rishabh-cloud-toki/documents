# Graphs — Reference

Quick reference for graphs: what they are, how to store them, and the standard algorithms
with their costs. The step-by-step explanations, traces and code are in the layered
companion, [graphs-notes.md](../notes/graphs-notes.md), in seven levels:
**1** the simple version · **2** storing and walking a graph · **3** traversal in depth ·
**4** directed graphs and ordering · **5** weighted graphs and shortest paths (Dijkstra) ·
**6** spanning trees (Prim, Kruskal, Union-Find) · **7** interview playbook.

A tree is a special graph (connected, no cycles); see [trees.md](trees.md). The heap
notes' [lazy deletion](heaps.md#8-advanced-patterns) pattern is what Dijkstra's algorithm
relies on.

> **Status: preview — not yet read.** To be expanded as you work through the topic.

Running example: vertices `0`–`5`, edges `0–1, 0–2, 1–3, 2–3, 3–4`, with `5` isolated.

## Contents

1. [What a graph is](#1-what-a-graph-is)
2. [Storing a graph](#2-storing-a-graph)
3. [Traversal: BFS and DFS](#3-traversal-bfs-and-dfs)
4. [Directed graphs and topological sort](#4-directed-graphs-and-topological-sort)
5. [Shortest paths](#5-shortest-paths)
6. [Minimum spanning trees](#6-minimum-spanning-trees)
7. [Interview playbook](#7-interview-playbook)

---

## 1. What a graph is

A set of **vertices** (things) and **edges** (connections). Edges can be **undirected**
or **directed**, and **unweighted** or **weighted**.

| Term | Meaning |
|---|---|
| Degree | Number of edges at a vertex (in-degree / out-degree when directed) |
| Path / cycle | A route along edges / a path that returns to its start |
| Connected | Every vertex can reach every other |
| Component | A maximal connected group |
| DAG | Directed graph with no cycles |

**Tree = connected graph with no cycles** (`V − 1` edges, one path between any two
vertices). Degrees sum to `2E`.

**Types of graphs.** A graph is classified on several independent axes, each with an
answer (a road network is undirected, weighted, cyclic, connected and sparse):

| Axis | Types |
|---|---|
| Edge direction | **Undirected** · **Directed** (digraph) |
| Edge weights | **Unweighted** · **Weighted** |
| Cycles | **Cyclic** · **Acyclic**: **tree** (connected), **forest** (several trees), **DAG** (directed) |
| Connectivity | **Connected** · **Disconnected** (components) · **Strongly connected** (directed, follow the arrows) vs **weakly connected** |
| Number of edges | **Sparse** (few) · **Dense** (near the maximum) · **Complete** (every pair; `V(V−1)/2` edges undirected) |
| Structure | **Simple** (no parallel edges or self-loops) vs **multigraph** · **Bipartite** (two groups, edges only between them; no odd cycle; BFS 2-colouring, `O(V + E)`) · grid |

Assume **simple, undirected, unweighted** if a problem does not say.

> Deep-dives: [what is a graph](../notes/graphs-notes.md#what-is-a-graph) ·
> [vocabulary](../notes/graphs-notes.md#vocabulary) ·
> [graph vs tree](../notes/graphs-notes.md#graph-vs-tree) ·
> [directed, undirected, weighted](../notes/graphs-notes.md#directed-undirected-weighted) ·
> [types of graphs](../notes/graphs-notes.md#types-of-graphs) ·
> [types in depth: edge counts, multigraphs, strong connectivity, bipartite](../notes/graphs-notes.md#types-of-graphs-in-depth) ·
> [Level 1 check](../notes/graphs-notes.md#level-1-check)

## 2. Storing a graph

| | Adjacency list | Adjacency matrix |
|---|---|---|
| Memory | `O(V + E)` | `O(V²)` |
| Edge `u–v`? | `O(degree(u))` | **`O(1)`** |
| Neighbours of `u` | **`O(degree(u))`** | `O(V)` |
| Best for | **Sparse** graphs (default) | Dense graphs |

An **edge list** (`(u, v, w)` triples) is convenient as input and for Kruskal. For grids
and puzzle states, compute neighbours on the fly instead of storing them.

> Deep-dives: [three ways to store a graph](../notes/graphs-notes.md#storing-a-graph-three-ways) ·
> [adjacency list in Java](../notes/graphs-notes.md#adjacency-list-in-java)

## 3. Traversal: BFS and DFS

| | BFS | DFS |
|---|---|---|
| Container | Queue | Stack / recursion |
| Explores | Ring by ring | Deep path first |
| Tree analogue | Level-order | Pre-order |
| Shortest path (unweighted) | **Yes** | No |

Both are **`O(V + E)`** with an adjacency list (`O(V²)` with a matrix) and need a
`visited` record because graphs have cycles. Mark visited **when enqueuing**. Count
components by starting a traversal from every unvisited vertex. Rebuild a route with a
`parent` array. Undirected cycle: DFS finds an already-visited vertex that is not the
parent. A grid is a graph of cells (number of islands = components).

> Deep-dives: [BFS with a trace](../notes/graphs-notes.md#breadth-first-search-bfs) ·
> [DFS with a trace](../notes/graphs-notes.md#depth-first-search-dfs) ·
> [components](../notes/graphs-notes.md#connected-components) ·
> [complexity](../notes/graphs-notes.md#complexity) ·
> [why BFS gives shortest paths](../notes/graphs-notes.md#why-bfs-finds-shortest-paths) ·
> [rebuilding the path](../notes/graphs-notes.md#rebuilding-the-path) ·
> [cycle detection](../notes/graphs-notes.md#cycle-detection-undirected-graphs) ·
> [grids](../notes/graphs-notes.md#grids-are-graphs) ·
> Checks: [Level 2](../notes/graphs-notes.md#level-2-check) ·
> [Level 3](../notes/graphs-notes.md#level-3-check)

## 4. Directed graphs and topological sort

A **topological order** lists vertices so every edge `u → v` has `u` before `v`; it exists
only for a **DAG**. **Kahn's algorithm:** repeatedly take a vertex with in-degree 0,
output it, and lower its neighbours' in-degrees. `O(V + E)`. If fewer than `V` vertices
come out, there is a **cycle**.

> Deep-dives: [directed graphs and DAGs](../notes/graphs-notes.md#directed-graphs-and-dags) ·
> [Kahn's algorithm with a trace](../notes/graphs-notes.md#topological-sort-kahns-algorithm) ·
> [cycles in directed graphs](../notes/graphs-notes.md#detecting-a-cycle-in-a-directed-graph) ·
> [Level 4 check](../notes/graphs-notes.md#level-4-check)

## 5. Shortest paths

| Situation | Algorithm | Cost |
|---|---|---|
| Unweighted | BFS | `O(V + E)` |
| Non-negative weights | **Dijkstra** | `O((V + E) log V)` |
| Negative weights | Bellman–Ford | `O(V·E)` |

**Dijkstra:** keep the best known distance to each vertex; repeatedly take the closest
unfinished vertex (a **min-heap** of `(distance, vertex)`), whose distance is now final,
and try to improve its neighbours. `PriorityQueue` has no decrease-key, so push a *new*
entry and skip **stale** ones (`if (d > dist[u]) continue;`) — the heap notes' lazy
deletion. Fails with negative weights.

> Deep-dives: [why BFS is not enough](../notes/graphs-notes.md#why-bfs-is-not-enough) ·
> [the idea](../notes/graphs-notes.md#dijkstras-algorithm-the-idea) ·
> [a worked trace with stale entries](../notes/graphs-notes.md#worked-example) ·
> [the Java code](../notes/graphs-notes.md#dijkstra-in-java) ·
> [why it works and when it fails](../notes/graphs-notes.md#why-it-works-and-when-it-fails) ·
> [Level 5 check](../notes/graphs-notes.md#level-5-check)

## 6. Minimum spanning trees

A **spanning tree** connects every vertex with `V − 1` edges and no cycles; the **MST**
has the smallest total weight (not the same as shortest paths from one vertex).

| | Prim | Kruskal |
|---|---|---|
| Idea | Grow one tree: add the cheapest edge leaving it | Take edges cheapest-first, skip any that form a cycle |
| Needs | Min-heap (ordered by edge weight) | Sorted edges + **Union-Find** |
| Cost | `O((V + E) log V)` | `O(E log E)` |

**Union-Find** answers "are these two vertices in the same group?" in almost `O(1)`
(path compression + union by rank).

> Deep-dives: [the idea](../notes/graphs-notes.md#minimum-spanning-tree-the-idea) ·
> [Prim, traced](../notes/graphs-notes.md#prims-algorithm) ·
> [Kruskal and Union-Find, traced](../notes/graphs-notes.md#kruskals-algorithm-and-union-find) ·
> [Prim vs Kruskal](../notes/graphs-notes.md#prim-vs-kruskal) ·
> [Level 6 check](../notes/graphs-notes.md#level-6-check)

## 7. Interview playbook

| Goal | Algorithm |
|---|---|
| Reachability, components, "islands" | DFS / BFS / Union-Find |
| Fewest steps (unweighted), word ladder | **BFS** |
| Cheapest route (weights ≥ 0), network delay | **Dijkstra** |
| Prerequisites, build order, deadlock | **Topological sort** |
| Connect everything cheaply | **MST** (Prim / Kruskal) |

**Also:** "split into two groups" / "no odd cycle" ⇒ bipartite check.

**Common mistakes:** no `visited`; marking visited too late in BFS; wrong edge direction;
assuming connectivity; BFS on weighted graphs; negative weights with Dijkstra;
`Integer.MAX_VALUE + w` overflow; using the undirected cycle test on a directed graph.
**Approach:** name the vertices and edges first (for puzzles, a vertex is a *state*),
then directed/weighted/connected, then the question, then the algorithm and its cost.

> Deep-dives: [recognising a graph problem](../notes/graphs-notes.md#recognising-a-graph-problem) ·
> [choosing the algorithm](../notes/graphs-notes.md#choosing-the-algorithm) ·
> [common mistakes](../notes/graphs-notes.md#common-mistakes) ·
> [a ten-problem practice set](../notes/graphs-notes.md#mixed-practice-set) ·
> [how to approach a new problem](../notes/graphs-notes.md#how-to-approach-a-new-problem) ·
> [going further](../notes/graphs-notes.md#going-further) ·
> [Level 7 check](../notes/graphs-notes.md#level-7-check)
