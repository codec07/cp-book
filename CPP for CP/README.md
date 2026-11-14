# Chapter 1 — CPP for CP

> **15 topics · ~97,600 words · 510 code blocks · ~13,450 lines · ≈ 8 hours of reading**

The language layer of the book. By the end of this chapter you will have a contest template you trust, an I/O setup that never TLEs on input size, a mental model of where your memory actually lives, and every STL container and algorithm you will reach for in the next twelve chapters.

This chapter does **not** teach algorithms. It teaches the tool you will implement them with — thoroughly enough that you stop losing points to the tool.

---

## Prerequisites

None. This is the starting point of the book.

You should be able to type, run a terminal command, and install a compiler. If you have written even a hundred lines of any programming language before, you are ready.

---

## What you will be able to do

By the end of Chapter 1:

- [ ] Paste a starter template into a judge and know what every line of it does.
- [ ] Read a g++ error message and locate the actual mistake in under a minute.
- [ ] Choose between `cin`, `scanf` and a custom `fread` reader based on the input size.
- [ ] Predict integer overflow before it happens, and know when to reach for `__int128`.
- [ ] Budget memory for an array against a 256 MB limit without running the program.
- [ ] Write a comparator that cannot segfault `std::sort`.
- [ ] Use `lower_bound` and binary-search-on-the-answer without an off-by-one.
- [ ] Explain why a local `int a[1000000]` crashes and a global one does not.
- [ ] Pick the right container from eleven options and justify it with complexity.
- [ ] Debug a wrong answer with `assert`, `cerr`, and a brute-force stress test.

---

## Table of contents

| # | Topic | What it gives you | §§ | Lines | ~Time |
|---|---|---|---|---|---|
| 01 | [CPP Basics and Syntax Introduction](Topic%2001%20-%20CPP%20Basics%20and%20Syntax%20Introduction.md) | Judging, program anatomy, `<bits/stdc++.h>`, the compilation pipeline, the contest template | 12 | 676 | 30 min |
| 02 | [Understanding Comments, Errors, Warnings](Topic%2002%20-%20Understanding%20Comments%2C%20Errors%2C%20Warnings.md) | Four diagnostic classes, reading g++ output, sanitizers, stress testing, UB hall of fame | 16 | 1,177 | 35 min |
| 03 | [Input, Output and Efficient I/O](Topic%2003%20-%20Input%2C%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20(VERY%20IMPORTANT).md) | Every I/O tier from `cin` to a custom reader, with measured timings and every trap | 15 | 913 | 30 min |
| 04 | [Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems%2C%20Variables%20and%20Datatypes.md) | Ranges, two's complement, overflow, `__int128`, floating point, bitwise idioms, `bitset` | 17 | 693 | 28 min |
| 05 | [String Manipulation Methods & Statements](Topic%2005%20-%20String%20Manipulation%20Methods%20%26%20Statements.md) | `std::string` end to end, `stringstream`, conversions, the quadratic `erase` trap, recipes | 21 | 791 | 25 min |
| 06 | [Conditional Statements](Topic%2006%20-%20Conditional%20Statements%20(If%2C%20else%2C%20else%20if%2C%20ternary%20operators).md) | Truthiness, short-circuit guards, `switch`, ternary, chained-comparison bug, branchless code | 13 | 768 | 31 min |
| 07 | [Loops & 'Auto'](Topic%2007%20-%20Loops%20%26%20'Auto'.md) | Three loop forms, off-by-one analysis, the $10^8$ rule, `auto`'s two silent changes, patterns | 16 | 745 | 31 min |
| 08 | [Useful Math Functions & Operations on Data](Topic%2008%20-%20Useful%20Math%20functions%20%26%20Operations%20on%20Data.md) | `abs`/`sqrt`/`pow` traps, negative modulo, `gcd`/`lcm`, modular arithmetic, bit builtins | 17 | 771 | 31 min |
| 09 | [Arrays (1D, 2D and 3D)](Topic%2009%20-%20Arrays%20(1D%2C%202D%20and%203D).md) | Memory layout, stack limits, budget arithmetic, `memset` lies, cache order, prefix sums | 14 | 714 | 30 min |
| 10 | [Memory Management in C++ and Pointers](Topic%2010%20-%20Memory%20Management%20in%20C%2B%2B%20and%20Pointers.md) | Stack/heap/globals, pointers vs references, `const` decoding, the segfault playbook | 16 | 1,151 | 43 min |
| 11 | [Functions, Classes and Useful Lambdas](Topic%2011%20-%20Functions%2C%20Classes%20(Basics)%20and%20Useful%20Lambda%20Methods.md) | Pass-by semantics, recursion, structs, operator overloading, lambdas including recursive | 15 | 1,546 | 50 min |
| 12 | [STL Part 1 — Vectors, DeQue, Lists, Forward Lists](Topic%2012%20-%20STL%20Part%201%20(Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists).md) | `vector` in full, amortised growth, iterator invalidation, why `list` is a trap | 15 | 788 | 29 min |
| 13 | [STL Part 2 — Set, Multiset, Map, Unordered](Topic%2013%20-%20STL%20Part%202%20(Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps).md) | Balanced BSTs, the member-vs-algorithm `lower_bound` TLE, the anti-hash attack, custom hashes | 14 | 913 | 31 min |
| 14 | [STL Part 3 — Stacks, Queue, Priority Queue](Topic%2014%20-%20STL%20Part%203%20(Stacks%2C%20Queue%20and%20Priority%20Queue).md) | Adaptors, monotonic stack, BFS frontier, why `greater<int>` is a min-heap, lazy deletion | 15 | 751 | 27 min |
| 15 | [STL Part 4 — Essential Prebuilt Algorithms](Topic%2015%20-%20STL%20Part%204%20(Essential%20Prebuilt%20Algorithms%20-%20Sort%2C%20Binary%20Search%2C%20Min%20Max%2C%20Array%20Manipulation%2C%20Permutations%2C%20Math%20%26%20Accumulators).md) | `sort` and comparators, `lower_bound`, binary search on the answer, erase–remove, `<numeric>` | 15 | 1,059 | 38 min |

**Totals:** 15 topics · 231 sections · 13,456 lines · 97,583 words · 510 `cpp` blocks.

---

## Dependency graph

Read in order the first time. The arrows show what genuinely blocks what.

```
01 Basics
 └─→ 02 Errors ──→ 03 I/O
      │              │
      └─→ 04 Types ──┼─→ 05 Strings
           │         │
           ├─→ 06 Conditionals ─→ 07 Loops
           │                       │
           ├─→ 08 Math ────────────┤
           │                       │
           └─→ 09 Arrays ──→ 10 Memory ──→ 11 Functions/Lambdas
                                              │
                          ┌───────────────────┴──────────────┐
                          ↓                                  ↓
                    12 vector/deque                    13 set/map
                          └──────────→ 14 stack/queue ←──────┘
                                            │
                                            ↓
                                    15 Algorithms
```

**If you are short on time**, the highest-leverage four are **03 (I/O)**, **09 (Arrays)**, **13 (set/map)** and **15 (Algorithms)**. Those four account for most of the difference between a TLE and an AC on Div. 2 C–D problems.

---

## Three ways to read this chapter

### Sprint — 3 days

| Day | Topics | Focus |
|---|---|---|
| 1 | 01, 02, 03, 04 | Get a template, learn to read errors, never lose to I/O, stop overflowing |
| 2 | 05, 06, 07, 08, 09 | The core language surface you touch in every solution |
| 3 | 10, 11, 12, 13, 14, 15 | Memory model, then all four STL parts back to back |

Skim the `Common Mistakes` table of anything you skip. Those tables are the compressed form of the whole file.

### Standard — 2 weeks

One topic per weekday, two on weekends. Do the practice table at the end of each file before moving on. This is the recommended pace: the chapter is designed so each file's practice problems use only material from files already read.

### Reference

Use the table of contents above as an index. Every file is self-contained enough to be read cold, and every file ends with a `Key Takeaways` block you can read in 60 seconds. When you hit a bug in a later chapter, come back to the relevant `Common Mistakes` table — that is what it is for.

---

## Self-check before Chapter 2

If you can answer all twelve without looking, you are ready.

1. Why does `ios_base::sync_with_stdio(false)` speed up `cin`, and what breaks if you then use `scanf`?
2. What is `sizeof(int)` on the judge, and at what $n$ does `n*(n+1)/2` overflow it?
3. Why is a local `int a[1000000]` dangerous and the global version not?
4. What does `memset(a, 1, sizeof a)` actually put in an `int` array, and why?
5. Why does `0 < x < 10` compile, and what does it mean?
6. What is the difference between `lower_bound` and `upper_bound`, and how do you get a count from them?
7. Why must a `sort` comparator return `false` for equal elements?
8. When is `std::lower_bound` on a `std::set` the wrong call, and why is it $\mathcal{O}(n)$?
9. What does `v.erase(remove(v.begin(), v.end(), x), v.end())` do that `remove` alone does not?
10. Why does `accumulate(v.begin(), v.end(), 0)` overflow where `0LL` does not?
11. What is the difference between `push_back` and `emplace_back`?
12. Give the two binary-search-on-the-answer templates and the one character that distinguishes them.

---

## What this chapter deliberately leaves out

Scope discipline matters as much as coverage. These belong to later chapters and are only *teased* here:

| Deferred topic | Where it lives |
|---|---|
| Big-O analysis, amortised analysis, the $10^8$ rule in full | Chapter 2, *Time Complexity* |
| Prefix sums from basics to 2D/3D and beyond | Chapter 3, *Prefix Sums* |
| Two pointers, binary search on unsorted arrays, coordinate compression | Chapter 3, *Advanced Searching & Sorting* |
| Modular exponentiation, inverses, combinatorics | Chapter 4, *Math for CP* |
| Graph traversal, DSU, shortest paths | Chapters 3 and 5 |
| Segment trees, Fenwick trees, sparse tables | Chapters 5 and 7 |
| String hashing, suffix arrays, KMP | Chapters 6 and 11 |

Chapter 1 gives you the primitives; those chapters give you the algorithms.

---

## Conventions in this chapter

- C++17, contest style: `#include <bits/stdc++.h>`, `using namespace std;`.
- Every code block compiles. Output is shown in a `text` block immediately after.
- `$...$` is inline maths, `$$...$$` is display maths — both render on GitHub.
- *Contest story* asides are real failure modes with the cost attached.
- Each file closes with `Key Takeaways` → `Common Mistakes & Pitfalls` → `Practice Problems` → `Next Up`.

---

## Related

- [← Project README](../README.md)
- [Chapter 2 — CP Level 1 →](../README.md#chapter-map) *(planned)*
- [Master syllabus](../content.md)

---

**Status:** ✅ Complete — 15/15 topics, all code compiled, core algorithms stress-tested against brute force.