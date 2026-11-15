# Chapter 3 - CP Level 2

> **9 topics · ~47,900 words · 110 code blocks · 7,329 lines · ≈ 4 hours of reading**

The ceiling layer. Chapter 2 gave you techniques that solve problems; this chapter gives you the techniques that make hard problems *tractable* - the ones that turn a nested loop into a scan, an impossible search into a binary search, and an arbitrary graph into a structure you can reason about.

Three things change here. Prefix sums stop being a trick and become a way of thinking about ranges. Binary search stops meaning "search a sorted array" and starts meaning "search the answer space". And the graph material stops being "BFS and DFS" and starts being "recognise the graph, then choose the traversal that answers the question".

By the end you will read a statement, find the phrase that names the technique, and know which boundary condition that technique fails on.

---

## Prerequisites

[Chapter 2 - CP Level 1](../CP%20Level%201/README.md) in full, especially:

- sorting as a reflex, and `set` / `map` used without hesitation,
- binary search from the standard library (`lower_bound`, `upper_bound`),
- BFS and DFS on adjacency lists, and flood fill on grids,
- the habit of proving a greedy with an exchange argument.

If any of those is shaky, finish Chapter 2 first. Nothing here re-teaches them - it extends them.

---

## What you will be able to do

By the end of Chapter 3:

- [ ] Build 1D, 2D and difference arrays from memory.
- [ ] Count subarrays with a given sum, or a sum divisible by $K$, using prefixes plus a map.
- [ ] Replace a pairwise scan with sort plus two pointers.
- [ ] Recognise when a window may shrink, and when it may not.
- [ ] Write a binary search on the answer with the correct `lo`/`hi` update for minimise *and* maximise.
- [ ] Binary search over values, not indices, when the counting function is monotone.
- [ ] Write a strict weak ordering comparator on the first attempt.
- [ ] Compress coordinates and know exactly what compression preserves.
- [ ] Prove greedy interval problems with the right sort key.
- [ ] Store any graph and traverse it iteratively, at $2 \times 10^5$ vertices.
- [ ] Detect undirected cycles with the parent test and directed cycles with three colours.
- [ ] Root a tree, compute subtree aggregates, and find its diameter in two passes.
- [ ] Peel a functional graph to its cycles, then propagate values backwards.
- [ ] Use a bitmask as a set, and count per bit instead of per pair.

---

## Table of contents

| # | Topic | What it gives you | §§ | Lines | ~Time |
|---|---|---|---|---|---|
| 01 | [Prefix Sums (From Basics to Advanced Level)](Topic%2001%20-%20Prefix%20Sums%20%28From%20Basics%20to%20Advanced%20Level%29.md) | 1D/2D/3D prefixes, subarray counting, difference arrays, XOR prefixes | 14 | 614 | 20 min |
| 02 | [Advanced Searching and Sorting Part 1](Topic%2002%20-%20Advanced%20Searching%20and%20Sorting%20Part%201%20%28Two%20Pointers%2C%20Binary%20Search%20on%20Sorted%20and%20Unsorted%20Arrays%29.md) | Two pointers, binary search on sorted and unsorted arrays, search on the answer | 17 | 704 | 25 min |
| 03 | [Advanced Searching and Sorting Part 2](Topic%2003%20-%20Advanced%20Searching%20and%20Sorting%20Part%202%20%28Custom%20Comparators%2C%20Coordinate%20Compression%20and%20Greedy%20Sorting%29.md) | Comparators, multi-key sorts, coordinate compression, sorting as the greedy | 16 | 684 | 22 min |
| 04 | [Advanced Searching and Sorting Part 3](Topic%2004%20-%20Advanced%20Searching%20and%20Sorting%20Part%203%20%2820%20Solved%20Examples%20and%2010%20Unsolved%20Problems%29.md) | 20 solved problems across every searching/sorting shape, plus 10 unsolved | 24 | 1,452 | 42 min |
| 05 | [Graph Theory Beginner Part 1](Topic%2005%20-%20Graph%20Theory%20Beginner%20Part%201%20%28Graph%20Traversal%20and%20Flood%20Fill%29.md) | Adjacency lists, BFS distances, DFS regions, grids as graphs, multi-source BFS | 14 | 714 | 24 min |
| 06 | [Graph Theory Beginner Part 2](Topic%2006%20-%20Graph%20Theory%20Beginner%20Part%202%20%28Intro%20to%20Tree%20Algos%20and%20Intro%20to%20Functional%20Graphs%29.md) | Rooting, subtree aggregates, diameter, LCA, binary lifting, functional graphs | 14 | 716 | 24 min |
| 07 | [Bitwise Operators](Topic%2007%20-%20Bitwise%20Operators.md) | The six operators, low-bit tricks, per-bit counting, subset masks, prefix XOR | 14 | 585 | 23 min |
| 08 | [Level 2 Solved Problem Set](Topic%2008%20-%20Level%202%20Solved%20Problem%20Set.md) | 20 problems solved end to end, one per technique, each with its trap | 23 | 1,507 | 40 min |
| 09 | [Level 2 Unsolved Problem Set](Topic%2009%20-%20Level%202%20Unsolved%20Problem%20Set.md) | 50 problems with constraints, grouped by family, with a grading rubric | 12 | 353 | 19 min |

**Totals:** 9 topics · 148 sections · 7,329 lines · 47,859 words · 110 `cpp` blocks · **70 complete programs compiled and run against the output shown**.

---

## Dependency graph

Read in order the first time. The two branches after Topic 04 are independent, and both feed the problem sets.

```
01 Prefix Sums
 └─→ 02 Two Pointers / Binary Search
      └─→ 03 Comparators / Compression / Greedy Sorting
           └─→ 04 Twenty Solved Problems
                │
                ├─→ 05 Graph Traversal & Flood Fill
                │    └─→ 06 Trees & Functional Graphs
                │
                └─→ 07 Bitwise Operators
                     │
                     └──────┬──→ 08 Level 2 Solved Set (20)
                            └──→ 09 Level 2 Unsolved Set (50)
```

**If you are short on time**, the highest-leverage three are **02 (binary search on the answer)**, **03 (comparators)** and **04 (the problem set)**. Those three are where Div. 2 C is decided. Topic 05 is unavoidable for anything graph-shaped, and Topic 07 is short but appears inside other problems more often than its length suggests.

Topics 08 and 09 are not new material - they are where the material becomes skill. Moving to Chapter 4 without attempting the fifty is the most common way to arrive at Level 3 unprepared.

---

## Three ways to read this chapter

### Sprint - 4 days

| Day | Topics | Focus |
|---|---|---|
| 1 | 01, 02 | Prefixes and two pointers; write both from memory |
| 2 | 03, 04 | Comparators, then the twenty solved problems |
| 3 | 05, 06 | Every traversal, iterative, on $10^5$-vertex inputs |
| 4 | 07, then start 09 | Bits, then the first ten unsolved problems |

### Standard - 2 weeks

One topic per weekday, one solved problem from Topic 04 or 08 per evening, and ten unsolved problems at the weekend. This is the recommended pace: the reading is short, and the retention comes from the problems.

### Reference

Every file opens with a `Prerequisites` block and closes with `Key Takeaways` → `Common Mistakes & Pitfalls` → `Practice Problems` → `Next Up`. When a later chapter uses a technique you have forgotten, read the `Common Mistakes` table first - it is the compressed form of the file.

---

## Self-check before Chapter 4

If you can answer all twelve without looking, you are ready.

1. What are the two loops that build a 2D prefix sum, and what is the inclusion–exclusion formula for a rectangle query?
2. Why does counting subarrays with sum $K$ need a map when negatives are allowed, but only a window when all values are positive?
3. State the monotonicity condition a sliding window requires, and give one problem that violates it.
4. In "minimise the maximum", which side moves when the predicate is true - `lo` or `hi` - and why does the other choice loop forever?
5. How do you decide whether the duplicate in an array of length $n+1$ lies below `mid`, and what makes that predicate monotone?
6. Why must a comparator return `false` for equal elements, and what breaks in `sort` when it returns `true`?
7. What does coordinate compression preserve, and what does it destroy?
8. Why is the correct sort key for maximum non-overlapping intervals the **end** time, not the start?
9. At any midpoint of a rotated sorted array, why is at least one half sorted, and how do you detect which?
10. Why must a tree be rooted iteratively at $n = 2 \times 10^5$, and what is reverse pre-order used for?
11. Why does two-pass BFS find a diameter, and what assumption about weights does the argument need?
12. What does Kahn's peel leave behind in a functional graph, and in which direction is the second pass run?

---

## What this chapter deliberately leaves out

Scope discipline matters as much as coverage. These belong to later chapters and are only *teased* here:

| Deferred topic | Where it lives |
|---|---|
| Fenwick trees, segment trees, sparse tables | Chapter 4, *Range Queries* |
| Disjoint set union, Dijkstra, MST, topological sort as a tool | Chapter 4, *Graph Theory Intermediate* |
| Dynamic programming in all its forms, including bitmask DP | Chapter 5, *Dynamic Programming* |
| Number theory, modular arithmetic, combinatorics | Chapter 4, *Math for CP* |
| Strings, hashing, tries, string algorithms | Chapters 6–7 |
| Advanced tree techniques: heavy-light, centroid decomposition, Euler tours | Chapter 8 |

Chapter 3 gives you the shapes. Those chapters give you the structures that hold the shapes under updates.

---

## Conventions in this chapter

- C++17, contest style: `#include <bits/stdc++.h>`, `using namespace std;`, fast I/O.
- Every complete program was compiled with `g++ -std=c++17 -O2` and run against the input shown; the output shown is the output it produced.
- `$...$` is inline maths, `$$...$$` is display maths - both render on GitHub.
- Greedy solutions carry an argument. A greedy without one is presented as a *wrong* greedy, on purpose.
- Topic 04 is the heaviest file in the chapter and is best read in two sittings.
- Each file closes with `Key Takeaways` → `Common Mistakes & Pitfalls` → `Practice Problems` → `Next Up`.

---

## Where to go next

→ [Chapter 4 - CP Level 3](../README.md#chapter-map), starting with **Fenwick trees and segment trees**.

Chapter 4 adds structures that survive updates: point-update/range-query and range-update/range-query in $O(\log n)$, disjoint sets for connectivity under merges, Dijkstra for weighted shortest paths, and dynamic programming over states rather than values.

Before starting it, hit the target in [Topic 09](Topic%2009%20-%20Level%202%20Unsolved%20Problem%20Set.md): **attempt all fifty, solve thirty without reading an editorial, and re-solve every failure from scratch a day later.** Below that, the gap is recognition - and Chapter 4 will expose it quickly.
