# Tries — Reference

Quick reference for tries (prefix trees). The step-by-step explanations, pseudocode,
traces and code are in the layered companion, [tries-notes.md](../notes/tries-notes.md):
**Level 1** the simple version · **2** how it works · **3** deep dive · **4** classic
problems · **5** interview playbook. A trie is a tree built for strings; see
[the map of trees](trees.md#the-map-of-trees), where it sits under *prefix trees*.

> **Status: preview — not yet read.** Moved here from the trees reference and expanded.

Running example: the words `cat`, `car`, `cart`, `do`, `dog`.

## Contents

1. [What a trie is](#1-what-a-trie-is)
2. [Operations and cost](#2-operations-and-cost)
3. [Memory and variants](#3-memory-and-variants)
4. [Trie vs the alternatives](#4-trie-vs-the-alternatives)
5. [Classic problems](#5-classic-problems)
6. [Interview playbook](#6-interview-playbook)

---

## 1. What a trie is

A tree that stores words **one character per edge**, so words with the same beginning
share the same path. Each node means "the prefix spelled so far" and carries an
**end-of-word flag**. The point: answer *"does any word start with `ca`?"* and *"list
all words starting with `ca`"* without scanning every word.

```
root ── c ── a ──┬── t ✔            cat
                 └── r ✔ ── t ✔     car, cart
     └─ d ── o ✔ ── g ✔             do, dog          (✔ = a word ends here)
```

The flag matters: `ca` is on a path but is not a word; `car` is a word *and* a
stepping stone to `cart`.

> Deep-dives: [what is a trie](../notes/tries-notes.md#what-is-a-trie) ·
> [seeing it](../notes/tries-notes.md#seeing-it) ·
> [vocabulary](../notes/tries-notes.md#vocabulary) ·
> [why not a list or hash set](../notes/tries-notes.md#why-not-just-use-a-list-or-a-hash-set) ·
> [Level 1 check](../notes/tries-notes.md#level-1-check)

## 2. Operations and cost

`L` = word length.

| Operation | How | Cost |
|---|---|---|
| `insert` | Walk down, creating missing nodes; flag the last | **`O(L)`** |
| `search` | Walk down; path must exist **and** end on a flagged node | **`O(L)`** |
| `startsWith` | Walk down; path must exist | **`O(L)`** |
| Autocomplete | Jump to the prefix node, then DFS below it (alphabetical for free) | `O(L + sub-trie size)` |
| `delete` | Unflag the word, then prune now-useless nodes on the way back up | `O(L)` |
| Count by prefix | Keep a pass-through counter on each node | `O(L)` |

The cost depends on the **word**, not on how many words are stored.

```java
class TrieNode { TrieNode[] next = new TrieNode[26]; boolean end; }
```

> Deep-dives: [the node](../notes/tries-notes.md#the-node) ·
> [insert](../notes/tries-notes.md#insert) ·
> [search and prefix check](../notes/tries-notes.md#search-and-prefix-check) ·
> [autocomplete](../notes/tries-notes.md#autocomplete-every-word-with-a-given-prefix) ·
> [complexity](../notes/tries-notes.md#complexity) ·
> [deletion](../notes/tries-notes.md#deletion) ·
> [counting by prefix](../notes/tries-notes.md#counting-words-by-prefix) ·
> Checks: [Level 2](../notes/tries-notes.md#level-2-check) ·
> [Level 3](../notes/tries-notes.md#level-3-check)

## 3. Memory and variants

Memory is the weak spot.

| Children stored as | Memory | Use when |
|---|---|---|
| Array `TrieNode[26]` | High (26 slots per node, mostly `null`) | Small fixed alphabet, dense trie |
| `HashMap<Character, TrieNode>` | Low (only real children) | Large alphabet (Unicode), sparse trie |

A **compressed (radix / Patricia) trie** merges chains of single-child nodes into one
edge labelled with a string: fewer nodes, less memory, harder insert and delete. Used in
IP routing and some key-value stores.

> Deep-dives: [array vs map](../notes/tries-notes.md#memory-and-the-array-vs-map-choice) ·
> [compressed tries](../notes/tries-notes.md#compressed-radix--patricia-tries)

## 4. Trie vs the alternatives

| | Trie | `HashSet` | `TreeMap` | Sorted array |
|---|---|---|---|---|
| Exact lookup | `O(L)` | `O(L)` | `O(L log N)` | `O(L log N)` |
| **Prefix query** | **`O(L + results)`** | `O(N·L)` scan | `O(L log N + results)` | `O(L log N + results)` |
| Sorted output | Free | No | Yes | Yes |
| Memory | **High** | Low | Medium | Lowest |
| Data per prefix | **Easy** | Awkward | Awkward | Awkward |

A `TreeMap` can also answer prefix queries, so the trie is not the only choice; it wins on
`O(L)` independent of `N`, shared storage and per-prefix data.

> Deep-dive: [trie vs the alternatives](../notes/tries-notes.md#trie-vs-the-alternatives)

## 5. Classic problems

| Problem | Idea |
|---|---|
| Implement a trie | Node + `insert` / `search` / `startsWith` |
| Wildcard search (`.` = any letter) | Recurse into **every** child on `.`; follow one path otherwise |
| Longest common prefix | Walk while the node has exactly one child and no word ends |
| Autocomplete / top-k suggestions | Prefix node + DFS, with a heap or a cached list per node |
| Word Search II | Trie + backtracking on a grid (after the backtracking topic) |
| Maximum XOR | Trie over the bits of each number |

> Deep-dives: [wildcard search](../notes/tries-notes.md#search-with-wildcards--matches-any-letter) ·
> [longest common prefix](../notes/tries-notes.md#longest-common-prefix-of-a-list-of-words) ·
> [problems that build on tries](../notes/tries-notes.md#problems-that-build-on-tries) ·
> [Level 4 check](../notes/tries-notes.md#level-4-check)

## 6. Interview playbook

**Trigger phrases:** "starts with", "prefix", "autocomplete", "type-ahead", a dictionary
queried many times by prefix, wildcard search, longest-prefix match, maximise XOR.
**Pick:** exact lookup only ⇒ `HashSet`; prefix queries on a big dictionary ⇒ trie;
small data ⇒ sorted array or `TreeMap`.
**Mistakes:** forgetting the end flag, mixing up `search` and `startsWith`, assuming
`a`–`z`, deleting without pruning, ignoring memory.

> Deep-dives: [recognising a trie problem](../notes/tries-notes.md#recognising-a-trie-problem) ·
> [choosing](../notes/tries-notes.md#choosing-trie-hash-set-or-sorted-structure) ·
> [real-world uses](../notes/tries-notes.md#where-tries-are-used-in-real-systems) ·
> [common mistakes](../notes/tries-notes.md#common-mistakes) ·
> [a ten-problem practice set](../notes/tries-notes.md#mixed-practice-set) ·
> [how to approach a new problem](../notes/tries-notes.md#how-to-approach-a-new-problem) ·
> [Level 5 check](../notes/tries-notes.md#level-5-check)
