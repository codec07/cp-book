# Chapter 2 — CP Level 1

> **11 topics · ~51,800 words · 140 code blocks · 7,372 lines · ≈ 4.3 hours of reading**

The algorithm layer. Chapter 1 gave you a language you can trust; this chapter gives you the six techniques that solve almost every problem rated 800–1600: complete search, sorting and maps, casework, greedy, graph traversal, and ad hoc geometry.

By the end you will read a constraint and know the technique before you know the solution. That single reflex is what separates a contestant who finishes Div. 2 A–C from one who spends forty minutes writing a brute force and watching it TLE.

---

## Prerequisites

[Chapter 1 — CPP for CP](../CPP%20for%20CP/README.md) in full. Specifically:

- a starter template you can type from memory,
- `vector`, `pair`, `set`, `map`, and `sort` used without hesitation,
- the habit of checking for overflow and stack limits before submitting.

If those are shaky, finish Chapter 1 first. Nothing here re-teaches them.

---

## What you will be able to do

By the end of Chapter 2:

- [ ] Read $n \le 2 \times 10^5$ and immediately rule out any nested loop.
- [ ] Read $n \le 20$ and immediately write a bitmask enumeration.
- [ ] Count the four shapes of search space and know which one a statement describes.
- [ ] Replace a pairwise scan with sort plus two pointers.
- [ ] Use prefix sums and a map to count subarrays with a given property.
- [ ] Split a problem into disjoint cases and prove the split is exhaustive.
- [ ] Prove a greedy by exchange argument — and recognise when it cannot be proved.
- [ ] Store a graph as an adjacency list and run BFS or DFS from memory.
- [ ] Choose BFS or DFS by the question, not by habit.
- [ ] Flood-fill a grid iteratively, at any size.
- [ ] Compute rectangle intersection, union, and distance without a boundary bug.
- [ ] Classify an unseen Level 1 problem in under a minute.

---

## Table of contents

| # | Topic | What it gives you | §§ | Lines | ~Time |
|---|---|---|---|---|---|
| 01 | [Introduction to CP and ICPC](Topic%2001%20-%20Introduction%20to%20CP%20and%20ICPC.md) | How contests actually work, ICPC format, verdicts, rating, the practice loop | 10 | 435 | 21 min |
| 02 | [Time Complexity (Basics to Advanced)](Topic%2002%20-%20Time%20Complexity%20(Basics%20to%20Advanced).md) | Big-O in contest terms, the $10^8$ rule, reading constraints, amortised analysis | 14 | 596 | 27 min |
| 03 | [Pairs, Tuples and Simulation](Topic%2003%20-%20Pairs%2C%20Tuples%20and%20Simulation.md) | `pair`/`tuple`, structured bindings, and the discipline of faithful simulation | 14 | 799 | 26 min |
| 04 | [Complete Searching](Topic%2004%20-%20Complete%20Searching.md) | Bitmask subsets, permutations, combinations, backtracking, pruning, N-Queens | 15 | 633 | 22 min |
| 05 | [Introduction to Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md) | The five things sorting buys, two pointers, prefix + map, live order with `set` | 12 | 662 | 23 min |
| 06 | [Casework Problems](Topic%2006%20-%20Casework%20Problems.md) | Disjoint/exhaustive splits, the six boundary cases, five worked problems | 13 | 694 | 26 min |
| 07 | [Introduction to Greedy Algorithms and Graphs](Topic%2007%20-%20Introduction%20to%20Greedy%20Algorithms%20and%20Graphs.md) | Exchange arguments, adjacency lists, BFS, DFS, grids as graphs | 14 | 626 | 21 min |
| 08 | [Problems on Greedy Algorithms and Graphs](Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md) | 5 greedy + 5 graph problems solved, plus 20 unsolved | 14 | 972 | 30 min |
| 09 | [Rectangle Geometry and Ad Hoc Problems](Topic%2009%20-%20Rectangle%20Geometry%20and%20Ad%20Hoc%20Problems.md) | Intersection, union, distance, and the six ad hoc techniques | 11 | 527 | 18 min |
| 10 | [Level 1 Solved Problem Set](Topic%2010%20-%20Level%201%20Solved%20Problem%20Set.md) | 20 problems solved end to end, each with the reasoning that chose the technique | 23 | 1,209 | 32 min |
| 11 | [Level 1 Unsolved Problem Set](Topic%2011%20-%20Level%201%20Unsolved%20Problem%20Set.md) | 50 problems with constraints, grouped by technique, plus a grading rubric | 11 | 219 | 13 min |

**Totals:** 11 topics · 151 sections · 7,372 lines · 51,764 words · 140 `cpp` blocks.

---

## Dependency graph

Read in order the first time. The arrows show what genuinely blocks what.

```
01 Intro to CP
 └─→ 02 Time Complexity
      └─→ 03 Pairs / Tuples / Simulation
           │
           ├─→ 04 Complete Search ──→ 05 Sorting / Sets / Maps
           │                              │
           │                              ├──→ 06 Casework
           │                              │
           └──────────────────────────────┴──→ 07 Greedy + Graphs
                                                  │
                                                  ↓
                                        08 Greedy / Graph Problems
                                                  │
                                                  ↓
                    09 Rectangle Geometry + Ad Hoc ←──┘
                                                  │
                                                  ↓
                                    10 Solved Set (20 problems)
                                                  │
                                                  ↓
                                    11 Unsolved Set (50 problems)
```

**If you are short on time**, the highest-leverage four are **02 (Time Complexity)**, **05 (Sorting/Sets/Maps)**, **06 (Casework)** and **07 (Greedy + Graphs)**. Those four account for most of the gap between solving A–B and solving C reliably.

Topics 10 and 11 are not new material — they are where the material becomes skill. Skipping them and moving to Chapter 3 is the most common way to arrive at Chapter 3 unprepared.

---

## Three ways to read this chapter

### Sprint — 4 days

| Day | Topics | Focus |
|---|---|---|
| 1 | 01, 02, 03 | How contests work, how to read constraints, how to simulate without bugs |
| 2 | 04, 05 | Enumeration and the sorting reflex |
| 3 | 06, 07, 08 | Casework discipline, then greedy and graphs in one sitting |
| 4 | 09, 10, 11 | Geometry and ad hoc, then work the solved set |

Skim the `Common Mistakes & Pitfalls` table of anything you skip. Those tables are the compressed form of the whole file.

### Standard — 2 weeks

One topic per weekday, two on weekends, and do the practice problems before moving on. This is the recommended pace. Topics 10 and 11 deserve a full week between them; they are the difference between reading the chapter and owning it.

### Reference

Use the table of contents as an index. Every file is self-contained enough to read cold, and every file opens with a `Prerequisites` block and ends with a `Key Takeaways` block readable in 60 seconds. When a later chapter uses a technique you have forgotten, come back to the `Common Mistakes` table — that is what it is for.

---

## Self-check before Chapter 3

If you can answer all twelve without looking, you are ready.

1. Given $n \le 2 \times 10^5$, what is the slowest complexity that still passes, and why is the constant factor often irrelevant at that size?
2. How many leaves does a search over all subsets of $n$ elements have, and at what $n$ does it stop being feasible with $\mathcal{O}(1)$ work per leaf?
3. Why is `sum[mask] = sum[mask ^ low] + a[i]` better than looping over the bits of every mask?
4. What two bound changes turn a permutation generator into a combination generator?
5. What does sorting buy you, in one sentence, that a scan does not?
6. Why is `seen[0] = 1` mandatory in the prefix-sum-plus-map pattern?
7. What are the two obligations every case split must satisfy, and how do you check the first?
8. Which of these is the correct interval-scheduling sort key: start time or end time — and why?
9. In a greedy proof, what does the exchange argument actually exchange?
10. Why does BFS give shortest paths only on unweighted graphs?
11. Where must a vertex be marked visited in BFS — on enqueue or on dequeue — and what breaks otherwise?
12. For two rectangles, when is `w <= 0 || h <= 0` the right disjoint test rather than four inequality comparisons?

---

## What this chapter deliberately leaves out

Scope discipline matters as much as coverage. These belong to later chapters and are only *teased* here:

| Deferred topic | Where it lives |
|---|---|
| Prefix sums from basics to 2D/3D and beyond | Chapter 3, *Prefix Sums* |
| Two pointers, binary search on unsorted arrays, binary search on the answer | Chapter 3, *Advanced Searching & Sorting* |
| Custom comparators, coordinate compression, sorting with greedy | Chapter 3, *Advanced Searching & Sorting Part 2* |
| Dijkstra, DSU, topological sort, MST | Chapter 4, *Graph Theory Intermediate* |
| Segment trees, Fenwick trees, sweep line over many rectangles | Chapters 6 and 7, *Range Queries* |
| Dynamic programming in all its forms | Chapter 5, *Dynamic Programming* |
| Number theory, modular arithmetic, combinatorics | Chapter 4, *Math for CP* |

Chapter 2 gives you the six techniques that cover the first third of every contest. Those chapters give you the rest.

---

## Conventions in this chapter

- C++17, contest style: `#include <bits/stdc++.h>`, `using namespace std;`.
- Every complete program in this chapter was compiled and run against the output shown.
- `$...$` is inline maths, `$$...$$` is display maths — both render on GitHub.
- Greedy solutions always carry a proof. A greedy without a proof in this chapter is presented as a *wrong* greedy, on purpose.
- Each file closes with `Key Takeaways` → `Common Mistakes & Pitfalls` → `Practice Problems` → `Next Up`.

---

## Where to go next

→ [Chapter 3 — CP Level 2](../README.md#chapter-map), starting with **Prefix Sums**.

Chapter 3 raises the ceiling rather than filling the toolkit: prefix sums as a first-class tool, two pointers and binary search on the answer, custom comparators and coordinate compression, then graph traversal in full.

Before starting it, hit the target in [Topic 11 §9](Topic%2011%20-%20Level%201%20Unsolved%20Problem%20Set.md): **classify 40 of 50 problems in one minute, solve 30 of 50 correctly.** Below that, the gap is classification — and Chapter 3 will expose it quickly.
