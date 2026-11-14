# Topic 01 — Prefix Sums (From Basics to Advanced Level)

Every technique in this chapter is a form of **precomputation for a different question**. Prefix sums are the simplest and the most reused: you pay $O(n)$ once, then answer any range-sum question in $O(1)$.

The idea is almost embarrassingly small — store the running total. What makes it a chapter-opening topic is how far the same trick reaches: from a single array, to a 2D grid, to a 3D cube, to counting subarrays with a given sum, to range updates through its mirror image, the **difference array**.

> **Prerequisites:** [Chapter 2, Topic 05 — Introduction to Sorting, Sets and Maps](../CP%20Level%201/Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Build 1D and 2D prefix sums from memory and answer range queries in $O(1)$.
> - Count subarrays with a given sum, sum divisible by $K$, or XOR equal to $K$.
> - Apply 1D and 2D difference arrays for range updates.
> - Choose between a prefix array, a difference array, and a hash map without hesitating.
> - Recognise the three situations where prefix sums quietly fail.

---

## 1. The core idea

A brute-force range-sum query costs $O(n)$. With $q$ queries that is $O(nq)$ — at $n = q = 10^5$, a guaranteed TLE.

The fix is one line of algebra. If

$$P_i = a_1 + a_2 + \cdots + a_i$$

then

$$\sum_{i=l}^{r} a_i = P_r - P_{l-1}.$$

One subtraction. The whole technique is that identity plus the discipline to index it correctly.

| Approach | Build | Query | Total for $q$ queries |
|---|---|---|---|
| Brute force | $O(1)$ | $O(n)$ | $O(nq)$ |
| Prefix sums | $O(n)$ | $O(1)$ | $O(n + q)$ |

**When the array changes between queries, prefix sums stop working.** An update at position $i$ changes every $P_j$ with $j \ge i$. That is why range-update problems fall to a difference array (this topic) or a Fenwick tree (Chapter 6).

---

## 2. 1D prefix sums

Use a sentinel at index 0 so that `$l = 1$` never needs a special case:

```cpp
// a is 1-indexed, a[1..n]
vector<long long> P(n + 1, 0);
for (int i = 1; i <= n; i++)
    P[i] = P[i - 1] + a[i];
```

Three properties worth memorising:

| Property | Consequence |
|---|---|
| `P[0] = 0` | `sum(1..r) = P[r] - P[0] = P[r]`, no special case |
| `P` is non-decreasing **iff** all $a_i \ge 0$ | Enables binary search on `P` for non-negative arrays |
| `P` has the same type as the sum, not the elements | `int` elements can need `long long` prefix |

That third row is a real bug source. An array of $10^5$ values each up to $10^9$ sums to $10^{14}$, which overflows `int` silently.

### The 0-based version

If the array is 0-indexed with `a[0..n-1]`, define

```cpp
vector<long long> P(n + 1, 0);
for (int i = 0; i < n; i++)
    P[i + 1] = P[i] + a[i];
```

Then `sum(l..r)` inclusive is `P[r + 1] - P[l]`. The offset is the whole difference: `P[i]` means "sum of the first `i` elements", not "sum up to index `i`".

**Pick one convention per problem and write it in a comment.** Mixing them is the single most common prefix-sum bug, and it produces answers that are wrong by exactly one element.

---

## 3. Solved Problem 1 — Static Range Sum Queries

> **Statement.** Given an array of $n$ integers and $q$ queries, each query gives $l \le r$. Output $\sum_{i=l}^{r} a_i$. $n, q \le 2 \times 10^5$, $|a_i| \le 10^9$.

### Why this approach

No updates, many queries. That is the exact profile prefix sums are built for: pay $O(n)$ once, then $O(1)$ per query. Total $O(n + q)$ instead of $O(nq)$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;

    vector<long long> P(n + 1, 0);
    for (int i = 1; i <= n; i++) {
        long long x;
        cin >> x;
        P[i] = P[i - 1] + x;
    }

    while (q--) {
        int l, r;
        cin >> l >> r;
        cout << P[r] - P[l - 1] << '\n';
    }

    return 0;
}
```

**Input**
```text
5 3
1 2 3 4 5
1 5
2 4
3 3
```

**Output**
```text
15
9
3
```

The whole range sums to 15; $a_2 + a_3 + a_4 = 2 + 3 + 4 = 9$; and a single element is $P_3 - P_2 = 6 - 3 = 3$.

**Complexity.** $O(n + q)$ time, $O(n)$ space.

### The trap

`P[r] - P[l - 1]` with `l = 1` reads `P[0]`, which exists only because you allocated `n + 1` entries. Allocating `n` entries and indexing `P[l]` instead of `P[l-1]` gives a wrong answer on every query that is not a prefix.

---

## 4. Solved Problem 2 — Counting Subarrays With Sum $K$

> **Statement.** Count the contiguous subarrays whose sum equals $K$. $n \le 2 \times 10^5$, values may be negative.

### Why this approach

The values can be negative, so a sliding window is invalid — shrinking the window does not reliably decrease the sum. Prefix sums turn the condition into an equality between two prefixes:

$$P_r - P_{l-1} = K \iff P_{l-1} = P_r - K.$$

Sweeping right to left, at each position you need the number of *earlier* prefix sums equal to $P_r - K$. That is a frequency map.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long K;
    cin >> n >> K;

    map<long long, long long> cnt;
    cnt[0] = 1;                 // the empty prefix

    long long pref = 0, ans = 0;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        pref += x;
        ans += cnt[pref - K];   // earlier prefixes that complete a valid subarray
        cnt[pref]++;
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5 3
1 2 3 2 1
```

**Output**
```text
3
```

Prefix sums are $0, 1, 3, 6, 8, 9$. Subarrays summing to 3: `[1,2]`, `[3]`, `[2,1]`.

**Complexity.** $O(n \log n)$ with `map`; $O(n)$ expected with `unordered_map`.

### The trap

`cnt[0] = 1` is not optional. Without it, every subarray starting at index 0 is missed, because its completing prefix is the empty prefix and nothing else. This one line is responsible for more wrong answers than the rest of the pattern combined.

---

## 5. The three counting variants

The same shape answers three different questions. Only the "key" changes.

| Question | Key stored in the map | Why |
|---|---|---|
| Count subarrays with sum $K$ | `P[i]` | $P_r - P_l = K$ |
| Count subarrays with sum divisible by $K$ | `P[i] mod K` | $P_r \equiv P_l \pmod K$ |
| Count subarrays with XOR $K$ | prefix XOR | $X_r \oplus X_l = K \iff X_l = X_r \oplus K$ |
| Count subarrays with sum 0 | `P[i]` | special case of $K = 0$ |

**XOR has no inverse under addition**, but it is its own inverse: $x \oplus x = 0$. So the same subtraction identity holds with $\oplus$ replacing $-$.

Two normalisation habits:

- **Modulo with negatives:** `((x % K) + K) % K`. In C++, `-3 % 2 == -1`.
- **XOR needs no normalisation** but does need the same `cnt[0] = 1` sentinel.

---

## 6. 2D prefix sums

For a grid, define

$$S_{i,j} = \sum_{r \le i,\ c \le j} a_{r,c}$$

with a zero row and column as sentinels. The recurrence is inclusion–exclusion on rectangles:

$$S_{i,j} = a_{i,j} + S_{i-1,j} + S_{i,j-1} - S_{i-1,j-1}$$

and a rectangle query $(r_1,c_1)$ to $(r_2,c_2)$ inclusive is

$$S_{r_2,c_2} - S_{r_1-1,c_2} - S_{r_2,c_1-1} + S_{r_1-1,c_1-1}.$$

**Read the query as four corners of a rectangle in prefix space.** The bottom-right corner is added; the two edges above and to the left are removed; the top-left corner was removed twice so it is added back. If you can draw that picture, you never need to memorise the formula.

```cpp
vector<vector<long long>> S(n + 1, vector<long long>(m + 1, 0));
for (int i = 1; i <= n; i++)
    for (int j = 1; j <= m; j++)
        S[i][j] = a[i][j] + S[i-1][j] + S[i][j-1] - S[i-1][j-1];
```

**Memory is the constraint.** An $n \times m$ `long long` prefix table costs $8nm$ bytes: $8$ MB for $1000 \times 1000$, but $800$ MB for $10^4 \times 10^4$. Always check that the table fits before assuming a 2D prefix is viable.

---

## 7. Solved Problem 3 — 2D Rectangle Sum Queries

> **Statement.** Given an $n \times m$ grid and $q$ queries, each query gives the corners of a rectangle. Output the sum inside it. $n, m \le 1000$, $q \le 2 \times 10^5$.

### Why this approach

Static grid, many rectangle queries — the 2D analogue of Problem 1. Build in $O(nm)$, answer each query in $O(1)$ from four table lookups.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, m, q;
    cin >> n >> m >> q;

    vector<vector<long long>> S(n + 1, vector<long long>(m + 1, 0));

    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            long long x;
            cin >> x;
            S[i][j] = x + S[i-1][j] + S[i][j-1] - S[i-1][j-1];
        }
    }

    while (q--) {
        int r1, c1, r2, c2;
        cin >> r1 >> c1 >> r2 >> c2;
        long long ans = S[r2][c2] - S[r1-1][c2] - S[r2][c1-1] + S[r1-1][c1-1];
        cout << ans << '\n';
    }

    return 0;
}
```

**Input**
```text
3 3 2
1 2 3
4 5 6
7 8 9
1 1 2 2
2 2 3 3
```

**Output**
```text
12
28
```

The first rectangle is $\{1,2,4,5\}$, sum 12. The second is $\{5,6,8,9\}$, sum 28.

**Complexity.** $O(nm + q)$ time, $O(nm)$ space.

### The trap

Building the table with `+ a + S[i-1][j] + S[i][j-1] - S[i-1][j-1]` but querying with only three terms (forgetting the `+ S[r1-1][c1-1]`). The result is correct whenever $r_1 = 1$ or $c_1 = 1$ — which is exactly what the sample test usually contains.

---

## 8. Difference arrays

A difference array answers the **mirror question**: many range updates, then a final read.

Instead of storing values, store *changes*. To add $v$ to every position in $[l, r]$:

```cpp
d[l]   += v;
d[r+1] -= v;      // cancel from here onward
```

Then one prefix pass reconstructs the array:

```cpp
long long run = 0;
for (int i = 1; i <= n; i++) {
    run += d[i];
    a[i] = run;
}
```

**Why it works.** A prefix sum and a difference array are inverse operations. `d` is the derivative of `a`; prefixing `d` recovers `a`. Adding $v$ on an interval is exactly "make the derivative jump up at $l$ and back down after $r$".

| | Prefix array | Difference array |
|---|---|---|
| Build | $O(n)$ | $O(n)$ zeros |
| Range query | $O(1)$ | $O(n)$ (must rebuild) |
| Range update | $O(n)$ | $O(1)$ |
| Use when | many queries, no updates | many updates, few reads |

**The two are duals.** If a problem has both range updates and range queries, neither alone is enough — that is a Fenwick or segment tree, in Chapter 6.

---

## 9. Solved Problem 4 — Range Add, Then Output the Array

> **Statement.** Start with an array of $n$ zeros. Apply $q$ operations `l r v`, each adding $v$ to every position in $[l, r]$. Output the final array.

### Why this approach

$q$ updates, one final read. Applying each update directly is $O(nq)$; the difference array makes each update $O(1)$ and the final reconstruction a single $O(n)$ pass.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, q;
    cin >> n >> q;

    vector<long long> d(n + 2, 0);      // +2: r + 1 may be n + 1

    while (q--) {
        int l, r;
        long long v;
        cin >> l >> r >> v;
        d[l]     += v;
        d[r + 1] -= v;
    }

    long long run = 0;
    for (int i = 1; i <= n; i++) {
        run += d[i];
        cout << run << " \n"[i == n];
    }

    return 0;
}
```

**Input**
```text
5 3
1 3 10
2 4 5
1 5 1
```

**Output**
```text
11 16 16 6 1
```

Position 1 receives $10 + 1 = 11$; positions 2–3 receive $10 + 5 + 1 = 16$; position 4 receives $5 + 1 = 6$; position 5 receives only the last operation, $1$.

**Complexity.** $O(n + q)$ time, $O(n)$ space.

### The trap

Allocating `d[n + 1]` and then writing `d[r + 1]` when `r = n`. That writes one past the end — and because it happens to be memory just past the vector, it often "works" locally and crashes on a judge. Allocate `n + 2`.

---

## 10. 2D difference arrays

The same duality in two dimensions. To add $v$ to the rectangle $(r_1,c_1)$–$(r_2,c_2)$:

```cpp
d[r1][c1]     += v;
d[r1][c2 + 1] -= v;
d[r2 + 1][c1] -= v;
d[r2 + 1][c2 + 1] += v;
```

Then one 2D prefix pass reconstructs the grid. The four-point pattern is the derivative of a rectangle, exactly as the two-point pattern is the derivative of an interval.

**Why the fourth sign is `+`.** The bottom-right corner is subtracted twice — once by the row cancellation, once by the column cancellation — so it must be added back. This is the same double-subtraction that forced the `+ S[r1-1][c1-1]` term in the 2D prefix query.

---

## 11. Solved Problem 5 — Rectangle Add, Then Output the Grid

> **Statement.** Start with an $n \times m$ grid of zeros. Apply $q$ operations, each adding $v$ to every cell of a rectangle. Output the final grid.

### Why this approach

Same reasoning as Problem 4, one dimension up. Each update costs four writes instead of $O(\text{area})$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m, q;
    cin >> n >> m >> q;

    // +2 on both axes so r2 + 1 == n and c2 + 1 == m are legal
    vector<vector<long long>> d(n + 2, vector<long long>(m + 2, 0));

    while (q--) {
        int r1, c1, r2, c2;
        long long v;
        cin >> r1 >> c1 >> r2 >> c2 >> v;
        d[r1][c1]         += v;
        d[r1][c2 + 1]     -= v;
        d[r2 + 1][c1]     -= v;
        d[r2 + 1][c2 + 1] += v;
    }

    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            d[i][j] += d[i-1][j] + d[i][j-1] - d[i-1][j-1];
            cout << d[i][j] << " \n"[j == m];
        }
    }

    return 0;
}
```

**Input**
```text
3 3 2
1 1 2 2 5
2 2 3 3 3
```

**Output**
```text
5 5 0
5 8 3
0 3 3
```

The first rectangle covers the top-left $2 \times 2$ with 5. The second covers rows 2–3 and columns 2–3 with 3. Cell $(2,2)$ is in both, so it holds $5 + 3 = 8$.

**Complexity.** $O(nm + q)$ time, $O(nm)$ space.

### The trap

Reconstructing with a prefix pass that skips the `- d[i-1][j-1]` term. The 2D prefix recurrence is the same one used in §6 — writing a fresh "simplified" version is how the bug gets in.

---

## 12. Prefix or difference? A decision table

| Problem shape | Tool | Build | Per operation |
|---|---|---|---|
| Many range **sums**, no updates | Prefix array | $O(n)$ | $O(1)$ |
| Many rectangle **sums**, no updates | 2D prefix | $O(nm)$ | $O(1)$ |
| Many range **adds**, one final read | Difference array | $O(1)$ | $O(1)$, then $O(n)$ |
| Many rectangle **adds**, one final read | 2D difference | $O(1)$ | $O(1)$, then $O(nm)$ |
| Count subarrays with a property | Prefix + map | — | $O(n \log n)$ |
| Range sums **and** range adds | Fenwick / segment tree | — | $O(\log n)$ — Chapter 6 |

The row to remember is the last one. Prefix and difference arrays are *offline* tools: they assume one of the two directions has no operations in it. The moment both appear, you need a tree.

---

## 13. Complexity summary

| Task | Time | Space |
|---|---|---|
| Build 1D prefix | $O(n)$ | $O(n)$ |
| 1D range sum | $O(1)$ | — |
| Build 2D prefix | $O(nm)$ | $O(nm)$ |
| 2D rectangle sum | $O(1)$ | — |
| Build 3D prefix | $O(nmk)$ | $O(nmk)$ |
| Range add (1D) | $O(1)$ | $O(n)$ |
| Rectangle add (2D) | $O(1)$ | $O(nm)$ |
| Reconstruct after updates | $O(n)$ / $O(nm)$ | — |
| Count subarrays with sum $K$ | $O(n \log n)$ | $O(n)$ |
| Count subarrays divisible by $K$ | $O(n \log K)$ with `map`, $O(n)$ with array of size $K$ | $O(K)$ |

The last row is worth noting: when $K \le 10^6$, use a plain `vector<int> cnt(K)` instead of a `map`, and the whole solution becomes $O(n)$.

### A note on 3D prefix sums

The same inclusion–exclusion extends one dimension further:

$$S_{i,j,k} = a_{i,j,k} + S_{i-1,j,k} + S_{i,j-1,k} + S_{i,j,k-1} - S_{i-1,j-1,k} - S_{i-1,j,k-1} - S_{i,j-1,k-1} + S_{i-1,j-1,k-1}.$$

Seven terms, alternating signs — add the three "one index down" slabs, subtract the three "two indices down" bars, add back the corner. It appears in 3D grid problems and almost nowhere else, and $O(nmk)$ memory usually rules it out before the formula matters. Recognise it; do not memorise it.

---

## 14. Unsolved problems

Five problems, no solutions. Each uses exactly one of the tools in §12 — decide which before you write anything.

| # | Problem | Constraints |
|---|---|---|
| 1 | **Count Subarrays Divisible by $K$.** Count the contiguous subarrays whose sum is divisible by $K$. | $n \le 2 \times 10^5$, $1 \le K \le 10^5$, $\lvert a_i \rvert \le 10^9$ |
| 2 | **Count Subarrays With XOR $K$.** Count the contiguous subarrays whose XOR equals $K$. | $n \le 2 \times 10^5$, $a_i \le 10^6$ |
| 3 | **Longest Subarray With Sum at Most $K$.** All elements are positive. Find the maximum length of a contiguous subarray with sum $\le K$. | $n \le 2 \times 10^5$, $a_i \ge 1$ |
| 4 | **Grid Range Queries.** Given an $n \times m$ grid of 0/1, answer $q$ queries, each asking how many 1s lie inside a given rectangle. | $n, m \le 1000$, $q \le 2 \times 10^5$ |
| 5 | **Range Add, Point Queries.** Apply $q$ range additions, then answer $p$ point queries. | $n, q, p \le 2 \times 10^5$ |

**Hints, in the order you should need them:** problem 3 does not need a map — positivity makes the two-pointer window valid, and prefix sums make the window check $O(1)$. Problem 4 is Problem 3 of this topic with a different query format. Problem 5 is Problem 4 with the reconstruction deferred until after all updates.

---

## Key Takeaways

- A prefix array converts $O(n)$ range sums into $O(1)$, at the cost of one $O(n)$ build.
- `P[0] = 0` is the sentinel that removes every boundary special case — allocate `n + 1`, not `n`.
- `P[i]` means "sum of the first $i$ elements". The off-by-one between that and "sum up to index $i$" causes the majority of prefix-sum bugs.
- The prefix type must hold the *sum*, not the element: `long long` for arrays of `int` up to $10^9$.
- Counting subarrays with sum $K$ is a map of prefix sums, plus the mandatory `cnt[0] = 1`.
- Sum divisible by $K$ stores remainders; XOR stores prefix XOR. XOR is its own inverse, which is why the same identity survives.
- Negative modulo must be normalised: `((x % K) + K) % K`.
- 2D prefix sums are inclusion–exclusion over four corners: add bottom-right, subtract the two edges, add back the top-left.
- $8nm$ bytes is the real cost of a 2D `long long` table — check it before building.
- Difference arrays are the inverse of prefix arrays: they make updates $O(1)$ and queries $O(n)$.
- Range add is `d[l] += v; d[r+1] -= v`. The `+1` is not a detail; allocate `n + 2`.
- Rectangle add is four writes, with a `+` on the bottom-right corner because it was subtracted twice.
- Prefix and difference arrays are both offline. Both-direction problems need a Fenwick or segment tree.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Allocating `n` instead of `n + 1` | `P[0]` is out of bounds or garbage | `n + 1`, sentinel 0 |
| 2 | `P[r] - P[l]` instead of `P[r] - P[l-1]` | Every non-prefix query is short by one element | `l - 1` |
| 3 | `int` prefix for `int` elements | Sum overflows at $10^9 \times 10^5$ | `long long` |
| 4 | Omitting `cnt[0] = 1` | Subarrays starting at index 0 are lost | Always seed it |
| 5 | Storing the raw sum instead of the remainder | Divisibility never matches | Store `pref % K` |
| 6 | `pref % K` with negative values | C++ gives a negative remainder | `((x % K) + K) % K` |
| 7 | Sliding window on an array with negatives | Window is not monotone in sum | Prefix + map instead |
| 8 | 2D query missing `+ S[r1-1][c1-1]` | Correct only when $r_1 = 1$ or $c_1 = 1$ | Four terms, always |
| 9 | `d[r + 1]` with `d` of size `n + 1` | Writes past the end when `r = n` | Size `n + 2` |
| 10 | 2D difference with `+` on the wrong corner | Rectangle bleeds into a quadrant | `+ - - +` |
| 11 | Rebuilding the prefix inside a query loop | $O(nq)$ | Build once |
| 12 | Using a 2D prefix at $10^4 \times 10^4$ | ~800 MB | Check memory first |
| 13 | `unordered_map` attacked by judged input | $O(n^2)$ worst case | `map`, or a custom hash |
| 14 | Forgetting that prefix sums break on updates | Wrong answers after the first update | Difference array or Fenwick |
| 15 | `lower_bound` on a prefix array with negatives | Prefix is not sorted | Binary search is invalid |

---

## Practice Problems

The five unsolved problems in §14 are the practice set. Work them in this order:

| Order | Problem | Why this order |
|---|---|---|
| 1 | §14.1 Divisible by $K$ | Extension of Solved Problem 2 with one changed key |
| 2 | §14.4 Grid Range Queries | Extension of Solved Problem 3 with a different query format |
| 3 | §14.5 Range Add, Point Queries | Extension of Solved Problem 4 in the offline direction |
| 4 | §14.2 XOR $K$ | Same map pattern, unfamiliar operation |
| 5 | §14.3 Longest Sum $\le K$ | Requires noticing that positivity changes the tool |

For a larger pool, use [CSES "Prefix Sum Queries" and "Subarray Sums"](https://cses.fi/problemset/) — the first is a Fenwick preview, the second a direct application of this topic.

---

## Next Up

→ [Topic 02 — Advanced Searching and Sorting Part 1](Topic%2002%20-%20Advanced%20Searching%20and%20Sorting%20Part%201%20%28Two%20Pointers%2C%20Binary%20Search%20on%20Sorted%20and%20Unsorted%20Arrays%29.md)

Prefix sums answer questions about a *fixed* range. The next topic answers questions about *where* something is: two pointers that shrink a window in linear time, and binary search that locates a threshold in logarithmic time — including the version that searches for an answer rather than an array element.
