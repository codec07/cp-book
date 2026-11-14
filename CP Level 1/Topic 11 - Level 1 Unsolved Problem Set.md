# Topic 11 — Level 1 Unsolved Problem Set

Fifty problems, no solutions, constraints given. Every technique from Chapter 2 appears at least four times, and none of the problems is a restatement of a solved one.

This is the set that converts reading into skill. The instruction is deliberately minimal: **classify before you code.** If you can name the technique and its complexity within a minute of reading a statement, you are ready for Chapter 3. If you find yourself writing a loop and hoping, stop and go back to the constraint.

> **Prerequisites:** all of Chapter 2. Attempt this set only after [Topic 10](Topic%2010%20-%20Level%201%20Solved%20Problem%20Set.md).
> **You will be able to:**
> - Classify an unseen Level 1 problem in under a minute.
> - Choose between complete search, sorting, casework, greedy, graph traversal, and closed form.
> - Recognise the four ways a problem hides its real difficulty.

---

## 1. How to use this set

| Step | Time | What you do |
|---|---|---|
| 1 | 1 min | Write the complexity the constraints permit |
| 2 | 1 min | Name the technique and the data structure |
| 3 | 15 min | Code it |
| 4 | 5 min | Test the sample, then the degenerate case |
| 5 | — | Only then submit |

If step 1 or 2 takes longer than a minute, the problem is not the code — it is the classification, and re-reading the statement will not fix that. Work the technique key in §8 instead.

**Groupings below are by intended technique.** That is a hint you would not get in a contest, so use it on a first pass and then re-attempt the set blind.

---

## 2. Group A — Complete Search and Enumeration

Constraints here are deliberately tiny. If you reach for anything other than enumeration, you have missed the point.

| # | Problem | Constraints |
|---|---|---|
| 1 | **Subset Divisibility.** Count the non-empty subsets whose product of elements is divisible by $K$. | $n \le 20$, $a_i \le 10^6$, $K \le 10^6$ |
| 2 | **Sum of Subset Sums.** Output the sum, over all $2^n$ subsets, of the sum of that subset's elements. | $n \le 20$, $a_i \le 10^4$ |
| 3 | **N-Queens Count.** Count the ways to place $n$ non-attacking queens on an $n \times n$ board. | $n \le 10$ |
| 4 | **K-th Permutation.** Output the $k$-th permutation of $1 \dots n$ in lexicographic order. | $n \le 8$, $1 \le k \le n!$ |
| 5 | **Maximum XOR Subset.** Choose a subset maximising the XOR of its elements. | $n \le 20$, $a_i < 2^{20}$ |
| 6 | **Monotone Paths.** Count paths from $(0,0)$ to $(n,m)$ moving only right or down, avoiding blocked cells. | $n, m \le 8$ |
| 7 | **Plus or Minus.** Insert `+` or `-` between the digits of a number so the expression equals a target. Report any valid expression or `-1`. | at most 8 digits |
| 8 | **Coin Combinations.** Count the ways to form an amount using unlimited coins of the given denominations (order irrelevant). | $n \le 10$, amount $\le 100$ |

---

## 3. Group B — Sorting, Sets and Maps

Every problem here is $\mathcal{O}(n \log n)$ or better. If your first idea is a double loop, sort something.

| # | Problem | Constraints |
|---|---|---|
| 9 | **Distinct Count.** Output the number of distinct values. | $n \le 2 \times 10^5$ |
| 10 | **Find the Duplicate.** An array of $n+1$ values, each in $[1, n]$, contains exactly one repeated value. Find it without modifying the array. | $n \le 2 \times 10^5$ |
| 11 | **Closest Pair on a Line.** Output the minimum $\lvert a_i - a_j \rvert$ over $i \ne j$. | $n \le 2 \times 10^5$ |
| 12 | **K-th Distinct Smallest.** Output the $k$-th smallest distinct value. | $n \le 2 \times 10^5$ |
| 13 | **Common Elements.** Output the distinct values present in both arrays, in ascending order. | $n, m \le 2 \times 10^5$ |
| 14 | **Zero-Sum Subarray.** Decide whether some contiguous subarray has sum exactly 0. | $n \le 2 \times 10^5$ |
| 15 | **First Unique Character.** Output the index of the first character that occurs exactly once, or `-1`. | $\lvert s \rvert \le 2 \times 10^5$ |
| 16 | **Rank Transform.** Replace each element by its rank among the distinct values (smallest rank 1). | $n \le 2 \times 10^5$ |
| 17 | **Count Pairs With Sum $T$.** Count index pairs $i < j$ with $a_i + a_j = T$, counting each index pair separately. | $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$ |

---

## 4. Group C — Casework and Construction

Each of these has at least one input where the obvious approach is wrong. Find it before submitting.

| # | Problem | Constraints |
|---|---|---|
| 18 | **Alternating Flips.** Find the minimum number of character flips so that no two adjacent characters are equal. | $\lvert s \rvert \le 2 \times 10^5$ |
| 19 | **Three Consecutive Evens.** Decide whether $n$ can be written as the sum of three consecutive even integers. | $n \le 10^9$ |
| 20 | **Sum and XOR.** Construct $n$ positive integers with total sum $S$ and XOR $X$, or report impossible. | $n \le 10^5$, $S, X \le 10^9$ |
| 21 | **Exactly $k$ Peaks.** Construct a permutation of $1 \dots n$ with exactly $k$ peaks (an element greater than both neighbours), or report impossible. | $n \le 10^5$ |
| 22 | **Product Sign.** Count the triples whose product is negative. | $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$ |
| 23 | **Alternating Digit Parity.** Construct an $n$-digit string (no leading zero) such that the sums of every two adjacent digits alternate in parity. | $n \le 10^5$ |
| 24 | **One Swap Sort.** Decide whether the array can be sorted with at most one swap of two elements. | $n \le 2 \times 10^5$ |
| 25 | **Palindrome After One Deletion.** Decide whether $s$ becomes a palindrome after deleting at most one character. | $\lvert s \rvert \le 2 \times 10^5$ |

---

## 5. Group D — Greedy

Each of these is passed by a sort plus a single pass, but proving *which* key is the whole problem.

| # | Problem | Constraints |
|---|---|---|
| 26 | **Minimum Notes.** With denominations 1, 2, 5, 10, 20, 50, 100, 200, 500, 2000, find the minimum number of notes for an amount. | amount $\le 10^9$ |
| 27 | **Ice Cream Bars.** Given costs and a budget, maximise the number of bars bought (one of each). | $n \le 10^5$ |
| 28 | **Minimum Difference Pairs.** Count the pairs realising the minimum absolute difference. | $n \le 2 \times 10^5$ |
| 29 | **Maximum Dot Product.** Rearrange both arrays so that the sum of element-wise products is maximised. | $n \le 10^5$ |
| 30 | **Minimum Cost to Meet.** Given positions on a line, move points one unit at a cost of one to make all coincide. Minimise the total cost. | $n \le 2 \times 10^5$ |
| 31 | **Two Meeting Rooms.** Given intervals, assign each to one of two rooms so that no room hosts overlapping meetings. Maximise the number scheduled. | $n \le 2 \times 10^5$ |
| 32 | **Maximum Chunks.** Split the array into the maximum number of contiguous chunks such that sorting each chunk and concatenating gives the sorted array. | $n \le 2 \times 10^5$ |
| 33 | **Unique After Increment.** In one step, increase any element by 1. Find the minimum steps so all elements are distinct. | $n \le 2 \times 10^5$ |
| 34 | **Largest Perimeter Triangle.** Choose three lengths forming a non-degenerate triangle with maximum perimeter. | $n \le 2 \times 10^5$ |

---

## 6. Group E — Graphs and Grids

Half of these do not look like graphs. Finding the vertices and edges is the problem.

| # | Problem | Constraints |
|---|---|---|
| 35 | **Count Islands.** Count 4-connected regions of `.` in a grid of `.` and `#`. | $R, C \le 1000$ |
| 36 | **Shortest Unit Path.** Find the fewest edges from $s$ to $t$ in an unweighted graph. | $n, m \le 2 \times 10^5$ |
| 37 | **Cycle Exists.** Decide whether an undirected graph contains a cycle. | $n, m \le 2 \times 10^5$ |
| 38 | **Two-Colourable.** Decide whether the vertices can be split into two groups with no edge inside a group. | $n, m \le 2 \times 10^5$ |
| 39 | **Nearest Facility.** Given several facility cells in a grid, output every cell's distance to the nearest one. | $R, C \le 1000$ |
| 40 | **Reachability.** Decide whether $t$ is reachable from $s$ in a directed graph. | $n, m \le 2 \times 10^5$ |
| 41 | **Eight-Direction Grid.** Find the shortest path between two cells when diagonal moves are allowed. | $R, C \le 1000$ |
| 42 | **Tree Depth.** Given a tree, output the maximum number of edges on a root-to-leaf path from vertex 1. | $n \le 2 \times 10^5$ |

---

## 7. Group F — Rectangle Geometry and Ad Hoc

Coordinates are integers up to $10^9$ unless stated. Watch for overflow in every area computation.

| # | Problem | Constraints |
|---|---|---|
| 43 | **Union Perimeter.** Find the perimeter of the union of two axis-aligned rectangles. | coordinates $\le 10^9$ |
| 44 | **Containment Count.** Given $n$ rectangles and a point, count how many rectangles strictly contain it. | $n \le 2 \times 10^5$ |
| 45 | **Three-Rectangle Union Area.** Find the total area covered by at least one of three rectangles. | coordinates $\le 10^9$ |
| 46 | **Largest Square in a Rectangle.** Find the side length of the largest axis-aligned square fitting inside a $w \times h$ rectangle. | $w, h \le 10^9$ |
| 47 | **King Moves.** Find the minimum moves for a king on an infinite board from one square to another. | coordinates $\le 10^9$ |
| 48 | **Touching Rectangles.** Decide whether two rectangles share at least one point, counting boundary contact. | coordinates $\le 10^9$ |
| 49 | **Border Lattice Points.** Count the integer points on the boundary of an axis-aligned rectangle with corners at integer coordinates. | coordinates $\le 10^9$ |
| 50 | **Rotate 90°.** Rotate a point by $90^\circ$ clockwise about the origin. | coordinates $\le 10^9$ |

---

## 8. Technique key

Cover this table until you have attempted the set. It maps each group to the Chapter 2 topic that answers it.

| Group | Primary topic | Secondary topic | The dead giveaway |
|---|---|---|---|
| A | [04](Topic%2004%20-%20Complete%20Searching.md) | [06](Topic%2006%20-%20Casework%20Problems.md) | $n \le 20$, $n \le 10$, "at most 8 digits" |
| B | [05](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md) | [04](Topic%2004%20-%20Complete%20Searching.md) | $n \le 2 \times 10^5$ and no graph in sight |
| C | [06](Topic%2006%20-%20Casework%20Problems.md) | [09](Topic%2009%20-%20Rectangle%20Geometry%20and%20Ad%20Hoc%20Problems.md) | "decide whether", "construct any", "or $-1$" |
| D | [08](Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md) | [05](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md) | "maximum/minimum number", "minimum cost", intervals |
| E | [07](Topic%2007%20-%20Introduction%20to%20Greedy%20Algorithms%20and%20Graphs.md) | [08](Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md) | grid, "path", "connected", "distance" |
| F | [09](Topic%2009%20-%20Rectangle%20Geometry%20and%20Ad%20Hoc%20Problems.md) | [06](Topic%2006%20-%20Casework%20Problems.md) | coordinates, area, perimeter, corners |

### The four hiding places

Across the fifty, difficulty comes from exactly four sources. Naming which one applies is half the classification.

| Hiding place | Problems |
|---|---|
| The constraint forces a technique you would not otherwise choose | 1, 5, 17, 39, 45 |
| A degenerate case decides correctness | 10, 19, 20, 21, 24, 46, 48 |
| The graph must be discovered, not given | 6, 31, 32, 35, 41, 42 |
| The answer is a formula, not a simulation | 26, 30, 46, 47, 49, 50 |

---

## 9. Self-grading

Score each attempt on two axes, because they fail independently.

| | Correct | Wrong | No attempt |
|---|---|---|---|
| **Under 25 min** | solid — pattern learned | silent bug — find the degenerate case | classification gap |
| **25–60 min** | slow but sound — practise recall | expected at this stage | revisit the topic |
| **Over 60 min** | over-engineering — reread the constraint | stop, read the solution idea, retry cold in a week | go back to the topic |

A solution that is **correct but slow to produce** is worth more than one that is **fast but wrong**, because the second kind is invisible until the judge finds it.

**Target before Chapter 3:** 40 of 50 at least correctly classified in one minute, and 30 of 50 solved correctly. Below that, the issue is classification, not coding — and Chapter 3 will be harder, not easier.

---

## 10. Pitfalls to watch for in this set

These are the traps the fifty were chosen to contain.

1. **All-negative input** breaks any maximum initialised to 0 (problems 2, 11, 28).
2. **Duplicate values** break two-pointer counting when $K = 0$ (problems 11, 17, 28).
3. **Boundary contact** is not area overlap (problems 43, 48).
4. **Overflow at $10^9 \times 10^9$** destroys every area and product (problems 43, 45, 46, 49).
5. **$n = 1$** has no pairs and no neighbours (problems 11, 21, 31, 34).
6. **Negative modulo** in prefix-remainder problems (problem 14, by extension).
7. **Disconnected input** needs an outer loop over all vertices (problems 35, 38, 40, 42).
8. **Recursion depth** on a $1000 \times 1000$ grid overflows the stack (problems 35, 39, 41).
9. **"Construct any"** is rarely unique — do not search for the intended construction, just build a valid one (problems 20, 21, 23).
10. **Greedy that needs to undo** is not greedy (problems 31, 32, 33) — check whether a simpler invariant settles it.

---

## 11. Closing Chapter 2

You now have the complete Level 1 toolkit:

| Technique | Where it was built |
|---|---|
| Reading constraints as instructions | Topic 02 |
| Simulation and pairs/tuples | Topic 03 |
| Complete search and backtracking | Topic 04 |
| Sorting, sets, maps, prefix sums | Topic 05 |
| Casework discipline | Topic 06 |
| Greedy with proof, graphs, BFS/DFS | Topics 07–08 |
| Geometry and ad hoc reasoning | Topic 09 |
| Technique selection under pressure | Topics 10–11 |

That is the whole of what separates a beginner from a Division 2 contestant who can reliably solve A, B and C. What it does **not** include is the machinery that makes hard problems tractable: prefix sums as a first-class tool, two pointers and binary search on answers, dynamic programming, and disjoint set union.

---

## Next Up

→ Chapter 3 — CP Level 2, starting with **Prefix Sums (From Basics to Advanced)**

Chapter 3 opens with the one technique that turns almost every "sum over a subarray" question into subtraction: **prefix sums**, first in one dimension, then in two. From there it builds toward binary search on the answer, custom comparators, coordinate compression, and graph traversal proper — the material that raises a ceiling rather than filling a toolkit.

Before moving on, finish §9's target: classify 40 of 50, solve 30 of 50.
