# Topic 09 - Level 2 Unsolved Problem Set

Fifty problems. No solutions, no code, no hints inside the statements.

This is the test of the chapter. Every problem here is solvable with Topics 01–07 - prefix sums, two pointers, binary search, comparators, greedy, graph traversal, trees, functional graphs and bits. Several combine two of them, which is the actual Level 2 skill.

The set is grouped by family so that the technique is visible from the group heading. That is deliberate: when you are learning, seeing the family removes one unknown at a time. Read the statement anyway and ask what in it *demands* the family - the phrase that gives it away is the thing worth memorising, not the answer.

> **Prerequisites:** All of Chapter 3, especially [Topic 08 - Level 2 Solved Problem Set](Topic%2008%20-%20Level%202%20Solved%20Problem%20Set.md)
> **You will be able to:**
> - Recognise each of the eight problem shapes from a cold statement.
> - Write the matching solution without looking anything up.
> - Judge your own attempt honestly against a time budget.
> - Know exactly which Level 3 topic each unsolved problem is waiting for.

---

## 1. How to use this set

Three rules, and they matter more than the problems.

**Rule 1 - attempt before reading anything.** For each problem, spend the full time budget with no editorial, no search, no code from a previous solution. Reading a solution you did not need is worse than failing a problem you did.

**Rule 2 - write down the invariant before the code.** One sentence: what does this pointer/prefix/window/queue mean at the top of every iteration? If you cannot write that sentence, the code will not survive the boundary cases.

**Rule 3 - test on the three inputs you were not given.** Empty or single element, all equal, and the largest value the constraints allow. Most wrong answers in this chapter are boundary failures, not algorithmic ones.

Time budgets: **A** problems 20 minutes, **B** problems 45 minutes, **C** problems 90 minutes. If the budget expires with no accepted solution, read a hint - not a full editorial - and try again the next day.

---

## 2. The fifty at a glance

| Group | Topics | Problems | Count |
|---|---|---|---|
| A | Prefix sums, difference arrays | 1–7 | 7 |
| B | Two pointers, sliding windows | 8–15 | 8 |
| C | Binary search | 16–23 | 8 |
| D | Sorting, comparators, greedy | 24–32 | 9 |
| E | Graph traversal | 33–40 | 8 |
| F | Trees and functional graphs | 41–45 | 5 |
| G | Bits | 46–50 | 5 |

Difficulty key: **A** warm-up, **B** contest level, **C** stretch - expect to need more than one attempt.

---

## 3. Group A - Prefix sums and difference arrays

Every problem here is a running total in disguise. The two directions are worth separating: prefix sums precompute **answers to range queries**, difference arrays precompute **the effect of range updates**. If the statement says "after all operations, print", the difference array is almost certainly the tool.

**1. Maximum Subarray Sum** - Find the maximum sum of a non-empty contiguous subarray.
*Constraints:* $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$. *Difficulty:* A.

**2. Zero-Sum Subarrays** - Count the contiguous subarrays whose sum is exactly zero.
*Constraints:* $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$. *Difficulty:* A.

**3. Rectangle Painting** - An $n \times m$ grid starts at zero. Each of $q$ operations adds $x$ to every cell of a rectangle given by $(r_1, c_1, r_2, c_2)$. Print the final grid.
*Constraints:* $n, m \le 1000$, $q \le 2 \times 10^5$, $\lvert x \rvert \le 10^9$. *Difficulty:* B.

**4. Maximum Sum Submatrix** - Find the maximum sum over all axis-aligned submatrices.
*Constraints:* $n, m \le 200$, $\lvert a_{ij} \rvert \le 10^4$. *Difficulty:* B.

**5. Range XOR Updates** - An array of $n$ zeros receives $q$ operations, each XORing $x$ into every position of $[l, r]$. Print the final array.
*Constraints:* $n, q \le 2 \times 10^5$, $x < 2^{30}$. *Difficulty:* B.

**6. Maximum Circular Subarray Sum** - The array is circular: after the last element comes the first. Find the maximum sum of a non-empty contiguous subarray in the circular order.
*Constraints:* $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$. *Difficulty:* B.

**7. Subarray Sums in a Range** - Count the pairs $(l, r)$ with $L \le a_l + \cdots + a_r \le R$.
*Constraints:* $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$, $\lvert L \rvert, \lvert R \rvert \le 10^{14}$. *Difficulty:* C.

---

## 4. Group B - Two pointers and sliding windows

All eight problems are about a window that moves. The distinction that decides the code: a **fixed-length** window slides by adding one and removing one; a **variable-length** window grows on the right and shrinks on the left, and is only valid when the property being tracked is monotone - adding an element can only make "sum exceeds S" or "too many distinct" worse, never better.

If shrinking does not restore validity, the window is the wrong tool.

**8. Longest Subarray With Sum At Most S** - The array contains only positive integers. Find the longest subarray whose sum is at most $S$.
*Constraints:* $n \le 2 \times 10^5$, $a_i \ge 1$, $S \le 10^{14}$. *Difficulty:* A.

**9. Shortest Subarray With Sum At Least S** - Positive integers again. Find the minimum length of a subarray whose sum is at least $S$, or print $-1$.
*Constraints:* $n \le 2 \times 10^5$, $a_i \ge 1$, $S \le 10^{14}$. *Difficulty:* A.

**10. At Most K Flips** - A binary array may have at most $K$ of its zeros changed to ones. Find the longest run of ones achievable.
*Constraints:* $n \le 2 \times 10^5$, $0 \le K \le n$. *Difficulty:* B.

**11. Counting Windows With At Most K Distinct** - Count the contiguous subarrays containing at most $K$ distinct values.
*Constraints:* $n \le 2 \times 10^5$, $a_i \le 10^9$, $K \le n$. *Difficulty:* B.

**12. Smallest Covering Window** - Find the shortest substring of $s$ that contains every distinct character of $s$ at least once.
*Constraints:* $\lvert s \rvert \le 2 \times 10^5$, lowercase letters. *Difficulty:* B.

**13. Three Sum Closest** - Choose three distinct indices and report the sum closest to a target $T$.
*Constraints:* $n \le 2000$, $\lvert a_i \rvert \le 10^9$, $\lvert T \rvert \le 10^9$. *Difficulty:* B.

**14. Closest Pair Sum** - Choose two distinct indices and report the sum closest to $T$.
*Constraints:* $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$. *Difficulty:* B.

**15. Move Zeroes** - Move every zero to the end while preserving the relative order of the non-zero elements. Do it in place with $O(1)$ extra space.
*Constraints:* $n \le 2 \times 10^5$. *Difficulty:* A.

---

## 5. Group C - Binary search

Eight problems, and only three of them search an array. The rest search an **answer**: the smallest speed, the smallest day, the $k$-th smallest value. The test is always the same - find a predicate that is false for every value below the answer and true from the answer onward, then decide whether the code minimises (`hi = mid`) or maximises (`lo = mid`).

If you cannot state the predicate, you are not ready to write the loop.

**16. Insert Position** - In a sorted array of distinct integers, find the first index whose value is at least $x$, or $n$ if there is none.
*Constraints:* $n \le 2 \times 10^5$, one query per run. *Difficulty:* A.

**17. Mountain Peak** - The array strictly increases and then strictly decreases. Find the index of the maximum.
*Constraints:* $n \le 2 \times 10^5$, all adjacent values distinct. *Difficulty:* A.

**18. First and Last** - A sorted array with duplicates. For each of $q$ values, print the first and last index where it occurs, or `-1 -1`.
*Constraints:* $n, q \le 2 \times 10^5$. *Difficulty:* A.

**19. Integer Square Root** - Find the largest integer $r$ with $r^2 \le n$.
*Constraints:* $n \le 10^{18}$. *Difficulty:* A.

**20. K-th in a Multiplication Table** - Consider the $n \times n$ table where cell $(i, j)$ holds $i \cdot j$. Find the $k$-th smallest entry, counting duplicates separately.
*Constraints:* $n \le 10^5$, $k \le n^2$. *Difficulty:* C.

**21. Median of a Row-Sorted Matrix** - An $n \times m$ matrix has every row sorted ascending. Find the median of all $n m$ entries.
*Constraints:* $n, m \le 1000$, $n m$ odd, entries $\le 10^9$. *Difficulty:* C.

**22. Minimum Shipping Speed** - There are $n$ piles with `a[i]` items. Each hour you pick one pile and remove up to `speed` items from it. Find the minimum integer speed that empties every pile within $h$ hours.
*Constraints:* $n \le 2 \times 10^5$, $a_i \le 10^9$, $h \ge n$. *Difficulty:* B.

**23. Search in a Bitonic Array** - The array strictly increases then strictly decreases. Decide whether $x$ occurs, in $O(\log n)$.
*Constraints:* $n \le 2 \times 10^5$, all values distinct. *Difficulty:* B.

---

## 6. Group D - Sorting, comparators and greedy

Nine problems where the sort is the algorithm. Two questions recur: **what key do I sort by**, and **what do I prove** to show that key is right.

For greedy problems, the proof is nearly always an exchange argument: take an optimal solution that differs from the greedy choice, swap, and show the swap does not make it worse. If you cannot write that exchange in two lines, the greedy is probably wrong - and the counterexample is usually two items where the "obvious" key ties.

**24. Frequency Sort** - Sort the integers by decreasing frequency; break ties by increasing value.
*Constraints:* $n \le 2 \times 10^5$, $a_i \le 10^9$. *Difficulty:* A.

**25. Length Then Lexicographic** - Sort the strings by length ascending; break ties lexicographically.
*Constraints:* $n \le 2 \times 10^5$, total length $\le 2 \times 10^5$. *Difficulty:* A.

**26. Can Attend All** - Given meeting intervals, decide whether one person can attend every meeting without overlap. Touching endpoints are allowed.
*Constraints:* $n \le 2 \times 10^5$, times $\le 10^9$. *Difficulty:* A.

**27. Job Sequencing** - Each job takes 1 unit of time, has a deadline $d_i$ and a profit $p_i$. Choose a subset to maximise total profit with at most one job per time slot, all finished by their deadlines.
*Constraints:* $n \le 10^5$, $d_i \le n$, $p_i \le 10^9$. *Difficulty:* C.

**28. Fractional Knapsack** - Items have value $v_i$ and weight $w_i$; the bag holds weight $W$; items may be split. Maximise total value.
*Constraints:* $n \le 10^5$, $v_i, w_i \le 10^9$, $W \le 10^{14}$. Print with 3 decimals. *Difficulty:* B.

**29. Mice and Holes** - $n$ mice and $n$ holes are on a line. Assign each mouse to a distinct hole, minimising the maximum travel distance.
*Constraints:* $n \le 10^5$, positions $\le 10^9$. *Difficulty:* B.

**30. Boats to Save People** - Each boat carries at most two people and at most `limit` total weight. Find the minimum number of boats.
*Constraints:* $n \le 2 \times 10^5$, weights $\le \text{limit} \le 10^9$. *Difficulty:* B.

**31. Task Cooldown** - Tasks are labelled by letters. Between two executions of the same task there must be at least $n$ intervals occupied by other tasks or idle. Find the minimum total intervals to finish all tasks.
*Constraints:* $n \le 100$, number of tasks $\le 10^4$. *Difficulty:* C.

**32. Candy Distribution** - Children stand in a row with ratings. Each child gets at least one candy; a child with a higher rating than an adjacent child must get strictly more. Minimise the total.
*Constraints:* $n \le 2 \times 10^5$, ratings $\le 10^9$. *Difficulty:* C.

---

## 7. Group E - Graph traversal

Eight problems, and four of them never say "graph". A grid is a graph; a board with jumps is a graph; a set of prerequisites is a directed graph. The first task is always the same: **name the vertices and the edges**, then decide whether you need distances (BFS), regions (DFS), or ordering (topological sort).

Direction matters throughout. Parent-based cycle detection works only on undirected graphs; a directed cycle needs the three-state test.

**33. Island Count** - Count the 4-connected regions of `1`s in a binary grid.
*Constraints:* $R, C \le 1000$. *Difficulty:* A.

**34. Flood Fill** - Replace the colour of the whole region connected to a starting cell with a new colour. Print the resulting grid.
*Constraints:* $R, C \le 500$. *Difficulty:* A.

**35. Maze Shortest Path** - A grid contains open cells and walls. Find the minimum number of 4-directional steps from the top-left to the bottom-right, or print $-1$.
*Constraints:* $R, C \le 1000$. *Difficulty:* A.

**36. Directed Cycle** - Decide whether a directed graph contains a directed cycle.
*Constraints:* $n, m \le 2 \times 10^5$. *Difficulty:* B.

**37. Provinces** - $n$ cities and $m$ undirected roads. Count the connected groups of cities.
*Constraints:* $n, m \le 2 \times 10^5$. *Difficulty:* A.

**38. Knight Moves** - On an $n \times n$ board, find the minimum number of knight moves between two given squares, or $-1$.
*Constraints:* $n \le 100$. *Difficulty:* B.

**39. Snakes and Ladders** - A $20 \times 20$ board is numbered 1 to 400 in boustrophedon order. Some squares teleport forward (ladders) or backward (snakes). Find the minimum dice throws from square 1 to square 400, or $-1$.
*Constraints:* up to 15 ladders and 15 snakes. *Difficulty:* C.

**40. Topological Order** - Given a directed graph, output any topological order of its vertices, or $-1$ if it contains a cycle.
*Constraints:* $n, m \le 2 \times 10^5$. *Difficulty:* B.

---

## 8. Group F - Trees and functional graphs

Five problems. Both shapes come with a guaranteed structure, so the work is choosing the traversal order: pre-order for downward quantities, reverse pre-order for upward ones, and the Kahn peel for functional graphs.

The recursion-depth trap applies to all of them. At $n = 2 \times 10^5$ a path-shaped tree is a legal input.

**41. Tree Height** - A tree is rooted at vertex 1. Print its height in edges.
*Constraints:* $n \le 2 \times 10^5$. *Difficulty:* A.

**42. Tree Diameter** - Print the number of edges on the longest path between any two vertices of a tree.
*Constraints:* $n \le 2 \times 10^5$. *Difficulty:* A.

**43. Distance Queries** - Answer $q$ queries: the number of edges on the path between $u$ and $v$.
*Constraints:* $n, q \le 2 \times 10^5$. *Difficulty:* B.

**44. Cycle Lengths** - In a functional graph given by `f[1..n]`, print the cycle length of every component.
*Constraints:* $n \le 2 \times 10^5$. *Difficulty:* B.

**45. Component Count** - In a functional graph, count the connected components, treating the directed edges as undirected.
*Constraints:* $n \le 2 \times 10^5$. *Difficulty:* A.

---

## 9. Group G - Bits

Five problems, and the split is the usual one: two are direct bit manipulation, two are XOR algebra, and one is per-bit counting. Watch the widths - `int` is not enough above bit 30, and `1 << 31` is undefined behaviour.

**46. Popcount Line** - For each of $n$ integers, print the number of set bits.
*Constraints:* $n \le 2 \times 10^5$, $a_i < 2^{31}$. *Difficulty:* A.

**47. Power of Two** - For each of $q$ integers, print `YES` if it is a power of two, and then the smallest power of two greater than or equal to it.
*Constraints:* $q \le 2 \times 10^5$, $n \le 10^{18}$. *Difficulty:* A.

**48. XOR of All Subarrays** - Compute the sum, over all contiguous subarrays, of the XOR of the subarray's elements.
*Constraints:* $n \le 2 \times 10^5$, $a_i \le 10^9$. *Difficulty:* C.

**49. Reverse Bits** - Reverse the order of the 32 bits of each integer.
*Constraints:* $q \le 2 \times 10^5$, values $< 2^{32}$. *Difficulty:* B.

**50. Two Unique Numbers** - Every element appears exactly twice except two, which appear once. Print both, in increasing order, in $O(n)$ time and $O(1)$ extra space.
*Constraints:* $n \le 2 \times 10^5$, values $\le 10^9$. *Difficulty:* B.

---

## 10. Suggested schedule

Fifty problems at roughly three per day is a little over two weeks. A five-week plan with review built in:

| Week | Problems | Focus |
|---|---|---|
| 1 | 1–10 | Prefix, difference, windows; build the invariant habit |
| 2 | 11–23 | Counting windows and answer-search; write the predicate first |
| 3 | 24–32 | Greedy proofs; one exchange argument per problem |
| 4 | 33–45 | Graph and tree traversals; iterative versions only |
| 5 | 46–50, then revisit every failure | Bits, plus a second attempt at what you did not solve |

Keep a log with one line per problem: name, family, minutes spent, verdict (`solved`, `solved slowly`, `failed`), and the reason for failure in one phrase - *wrong invariant*, *boundary*, *overflow*, *wrong technique*. After two weeks the "reason" column is a diagnosis of your own habits, and it is more useful than the solved count.

---

## 11. Grading yourself

| Verdict | Meaning | What to do next |
|---|---|---|
| Solved inside budget | The technique is yours | Move on; revisit in a week |
| Solved over budget | Shape recognised, details slow | Re-derive the invariant from scratch |
| Failed, then solved from a hint | One missing idea | Rewrite the solution the next day with no reference |
| Failed, solved only from an editorial | The pattern was not recognised | Add the give-away phrase to your notes |
| Failed twice | A Level 3 technique is hiding here | Note it, move on, return after Chapter 4 |

A problem counts as solved only when the code passes the three extra tests from §1 - empty or single element, all equal, and the maximum constraint.

---

## 12. Where these lead

Several problems here are deliberately at the edge of Level 2, and each points at a specific Level 3 topic:

| Problem | Waiting for |
|---|---|
| 7 - subarray sums in a range | sorting a prefix array, then binary search |
| 20, 21 - table and matrix medians | value-space binary search with a counting function |
| 27 - job sequencing | disjoint-set "next free slot" |
| 31, 32 - cooldown and candy | greedy with a mathematical formula |
| 36, 40 - directed cycle, topological order | DFS colours and Kahn's algorithm |
| 43 - distance queries | binary lifting or an Euler tour with a segment tree |
| 48 - XOR of all subarrays | per-bit counting with prefix XOR |

If one of these refuses to yield after two honest attempts, that is not a gap in this chapter - it is a preview of the next one.

---

## Key Takeaways

- Fifty problems is a diagnostic, not a scoreboard; the failure reasons are the output.
- Every problem in Groups A–C is a loop over one variable - find the variable first.
- Prefix sums answer range queries; difference arrays apply range updates. The direction of the question chooses between them.
- A sliding window is valid only when the tracked property is monotone in window size.
- Binary search needs a predicate, not an array. State it in one sentence before coding.
- "Minimise the maximum" and "k-th smallest" are answer searches, and the counting function does the work.
- A greedy needs an exchange argument; the sort key is the conclusion of that argument, not a guess.
- On a grid, say "cell is a vertex, side is an edge" out loud before choosing BFS or DFS.
- Undirected cycle detection uses the parent test; directed cycle detection needs three states.
- Trees: pre-order down, reverse pre-order up, never recursion at $10^5$ on a path.
- Functional graphs: peel with Kahn, then reverse the peel order.
- Bits: `int` ends at bit 30, `1 << 31` is UB, and `ones * zeros` needs `long long`.
- Three extra tests after the sample - empty, all equal, maximum - catch most boundary failures.
- Re-solve failures from scratch a day later; reading your own old code teaches nothing.

---

## Common Mistakes & Pitfalls

These are practice mistakes rather than coding mistakes, and they cost more time than any bug.

| # | Mistake | Cost | Fix |
|---|---|---|---|
| 1 | Reading the editorial after ten minutes | Removes the learning, keeps the illusion | Full time budget first |
| 2 | Coding before naming the invariant | Debugging a solution with no idea what it means | One sentence first |
| 3 | Testing only on the sample | Sample usually passes boundary bugs | Three extra tests |
| 4 | Choosing the technique by the story | Wrong tool, correct-looking code | Choose by the constraint |
| 5 | Tuning constants in a wrong algorithm | Hours lost to a TLE that was an RE | Re-check the complexity first |
| 6 | Skipping a problem because it "looks easy" | The easy-looking ones hide the boundary | Solve it or log why not |
| 7 | Counting solved instead of logging failures | No diagnosis | Keep the reason column |
| 8 | Reusing old code without rereading it | Inherits old bugs | Rewrite from memory |
| 9 | Ignoring the maximum constraints | Overflow and recursion depth | Test at the limit |
| 10 | Reading solutions in the same session | The idea fades within a day | Sleep, then re-derive |

---

## Practice Problems

The fifty problems above **are** the practice set. Two ways to use it:

**By family.** Work Groups A→G in order, one group per sitting. This is the fastest way to consolidate a technique, because the family is known and the only variable is the problem.

**By difficulty.** Take all A problems first (about two days), then all B, then C. This mixes families and forces the recognition step, which is what a contest actually tests. Do this second pass at least once before moving to Level 3.

---

## Next Up

**Chapter 4 - CP Level 3**, starting with Fenwick trees and segment trees.

Level 2 gave you shapes: ranges, windows, answers, orderings, regions, trees, bits. Level 3 gives you *structures* that hold those shapes under updates - Fenwick trees and segment trees for ranges, disjoint sets for connectivity, Dijkstra for weights, and dynamic programming for the problems where the answer depends on a state rather than a value.

Before moving on, finish this chapter's target: attempt all fifty, solve thirty of them without reading an editorial, and re-solve every failure from scratch a day later.
