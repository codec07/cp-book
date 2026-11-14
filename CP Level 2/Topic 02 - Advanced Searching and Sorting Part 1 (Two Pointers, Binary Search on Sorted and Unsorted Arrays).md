# Topic 02 — Advanced Searching and Sorting Part 1

Two techniques, one shared idea: **do not examine what you can rule out.**

A two-pointer scan rules out a whole set of pairs at once, without ever looking at them. A binary search rules out half the remaining candidates per comparison, without ever looking at them. Both replace an $O(n^2)$ enumeration with something linear or logarithmic, and both are the difference between an $O(n \log n)$ solution and a TLE at $n = 2 \times 10^5$.

> **Prerequisites:** [Topic 01 — Prefix Sums](Topic%2001%20-%20Prefix%20Sums%20%28From%20Basics%20to%20Advanced%20Level%29.md), [Chapter 2, Topic 05](../CP%20Level%201/Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Recognise the three two-pointer patterns and pick the right one in seconds.
> - Write `lower_bound` and `upper_bound` by hand without an off-by-one.
> - Choose the correct binary-search template for "first true" versus "last true".
> - Binary search on an answer rather than an array, given a monotone feasibility check.
> - Avoid the four infinite-loop and wrong-answer bugs that account for almost every failed binary search.

---

## 1. What both techniques have in common

| | Two pointers | Binary search |
|---|---|---|
| Evidence used per step | A local comparison | One comparison with the midpoint |
| Work discarded per step | One pair, or one endpoint | Half the candidates |
| Prerequisite | Sortedness, or a monotone window property | Monotone predicate |
| Time | $O(n)$ | $O(\log n)$ |
| Fails silently when | The window property is not monotone | The predicate is not monotone |

The word **monotone** appears in both rows of the last line. That is the whole topic: every one of these techniques is valid only because some quantity increases (or decreases) without turning back. If you cannot name that quantity and say why it is monotone, you do not yet have a correct solution.

---

## 2. Two pointers: the invariant

The naive pair scan is $\Theta(n^2)$. The two-pointer scan is $\Theta(n)$ — not because it computes faster, but because it **never considers most pairs at all**.

The invariant is a claim of the form:

> *Every pair that could still improve the answer has both endpoints inside the current window.*

When you move a pointer, you discard pairs that are provably not better. If that claim holds, the discarded pairs were never needed. If it does not hold, the algorithm is wrong — and it will still produce plausible answers.

There are exactly three patterns worth memorising.

---

## 3. Pattern A — Both ends converging

Used when the array is **sorted** and the answer depends on `a[i] + a[j]` or similar.

```cpp
int i = 0, j = n - 1;
while (i < j) {
    long long s = a[i] + a[j];
    if (s == target) { /* found */ break; }
    if (s < target) i++;     // need more: increase the small side
    else            j--;     // need less: decrease the large side
}
```

**Why it is correct.** If `a[i] + a[j] < target`, then since the array is sorted, `a[i] + a[k] <= a[i] + a[j] < target` for every `k <= j`. So `i` can never be part of a valid pair with any remaining `j`, and it is safe to drop. Symmetrically for the other branch.

That paragraph is the entire proof. Write it once and it holds for every two-sum variant.

---

## 4. Pattern B — Slow and fast write pointers

Used for **in-place compaction**: removing duplicates, deleting an element type, moving zeros.

```cpp
int slow = 0;
for (int fast = 0; fast < n; fast++) {
    if (keep(a[fast])) {
        a[slow++] = a[fast];
    }
}
// new logical length is slow
```

**The invariant:** everything in `a[0..slow-1]` is kept, in original order, and `slow <= fast`.

`fast` is the reader and is always ahead; `slow` is the writer and only advances when something is kept. No element is ever overwritten before it is read, because `slow <= fast`.

---

## 5. Pattern C — Sliding window

Used when the answer is a **contiguous window** and the window property changes monotonically as you add or remove elements.

```cpp
int left = 0;
for (int right = 0; right < n; right++) {
    // add a[right] to the window state
    while (windowInvalid()) {
        // remove a[left] from the window state
        left++;
    }
    // window [left, right] is now valid
}
```

**The monotonicity requirement.** Adding elements must only make the window "worse" in a way that removing elements from the left can fix. This holds for:

| Window property | Monotone? |
|---|---|
| Sum, with all elements $\ge 0$ | yes |
| Sum, with negative elements | **no** — cannot use a window |
| Number of distinct elements | yes (adding can only increase) |
| Count of a repeated element | yes |
| Maximum element (with a deque) | yes — see Chapter 6 |
| "Contains a pair summing to $K$" | **no** |

The negative-sum row is the reason [Topic 01](Topic%2001%20-%20Prefix%20Sums%20%28From%20Basics%20to%20Advanced%20Level%29.md) used a prefix-sum map for "sum equals $K$" instead of a window.

---

## 6. Solved Problem 1 — Pair Sum in a Sorted Array

> **Statement.** Given a sorted array of $n$ integers and a target $T$, find two indices $i < j$ with $a_i + a_j = T$. Output them (1-based), or `-1`. $n \le 2 \times 10^5$.

### Why this approach

Sorted input, pair condition. Pattern A applies directly: $O(n)$ total, and no extra memory.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long T;
    cin >> n >> T;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    int i = 0, j = n - 1;
    while (i < j) {
        long long s = a[i] + a[j];
        if (s == T) {
            cout << i + 1 << ' ' << j + 1 << '\n';
            return 0;
        }
        if (s < T) i++;
        else       j--;
    }

    cout << -1 << '\n';
    return 0;
}
```

**Input**
```text
6 11
1 2 4 5 6 8
```

**Output**
```text
4 5
```

The scan runs: $1+8=9<11 \Rightarrow i{+}{+}$; $2+8=10<11 \Rightarrow i{+}{+}$; $4+8=12>11 \Rightarrow j{-}{-}$; $4+6=10<11 \Rightarrow i{+}{+}$; $5+6=11$ — found at positions 4 and 5.

**Complexity.** $O(n)$ after the input; $O(n \log n)$ if you must sort first.

### The trap

Using `while (i <= j)` instead of `while (i < j)`. With `i == j`, the pair is an element added to itself, which the statement forbids. The `i < j` bound is part of the problem, not a stylistic choice.

---

## 7. Solved Problem 2 — Longest Substring Without Repeating Characters

> **Statement.** Given a string $s$, find the length of the longest substring containing no repeated character. $|s| \le 2 \times 10^5$.

### Why this approach

Contiguous window, and the property "no duplicates" is monotone: adding a character can break it, removing from the left restores it. Pattern C applies — and because the state is "last seen position of each character", the window can be repaired with one assignment instead of a `while` loop.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s;
    cin >> s;

    vector<int> last(256, -1);      // last position of each ASCII character
    int left = 0, best = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        unsigned char c = s[right];
        if (last[c] >= left) {
            left = last[c] + 1;     // jump past the previous occurrence
        }
        last[c] = right;
        best = max(best, right - left + 1);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
abcabcbb
```

**Output**
```text
3
```

After the first three characters the window is `abc`; each later character forces `left` forward, and the window never exceeds length 3.

**Complexity.** $O(n)$ time, $O(1)$ space (a fixed 256-entry table).

### The trap

Writing `while (window has a duplicate) left++` and re-scanning the window each time to check. That is $O(n^2)$ in disguise. The `last[c] >= left` test makes the repair a single step, because the only possible duplicate is the character just added.

Note the `unsigned char` cast: indexing with a negative `char` is undefined behaviour for non-ASCII input.

---

## 8. Binary search: the three templates

Almost every binary-search bug is a template mismatch. There are only three shapes, and each has a fixed answer.

### Template 1 — first index where the predicate is true

Predicate $P$ is monotone: `false ... false true ... true`. Search the half-open range $[0, n)$.

```cpp
int lo = 0, hi = n;                     // answer in [0, n]
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (P(mid)) hi = mid;               // mid is a candidate; look left
    else        lo = mid + 1;           // mid is out; look right
}
// lo == first index with P(lo) == true, or n if none
```

### Template 2 — last index where the predicate is true

Predicate is `true ... true false ... false`. Search the closed range $[0, n-1]$.

```cpp
int lo = 0, hi = n - 1;                 // answer in [0, n-1]
while (lo < hi) {
    int mid = lo + (hi - lo + 1) / 2;   // upper midpoint
    if (P(mid)) lo = mid;               // mid is a candidate; look right
    else        hi = mid - 1;           // mid is out; look left
}
// lo == last index with P(lo) == true (requires P(0) == true)
```

### Template 3 — exact match

No monotone predicate; the array is sorted and you want equality.

```cpp
int lo = 0, hi = n - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (a[mid] == x) return mid;
    if (a[mid] < x)  lo = mid + 1;
    else             hi = mid - 1;
}
return -1;
```

### Why the midpoint differs

In Template 1, `mid` is the lower midpoint. When `lo + 1 == hi`, `mid == lo`, and the `else` branch sets `lo = mid + 1 = hi`, so the loop terminates.

In Template 2, `mid` must be the **upper** midpoint. If you used `(lo + hi) / 2`, then when `lo + 1 == hi` you get `mid == lo`, and the `if` branch sets `lo = mid` — no progress, infinite loop.

| Template | Range | Midpoint | Answer | Fails if |
|---|---|---|---|---|
| 1: first true | `[0, n)` | `(lo + hi) / 2` | `lo` | predicate not monotone |
| 2: last true | `[0, n-1]` | `(lo + hi + 1) / 2` | `lo` | `P(0)` is false |
| 3: exact | `[0, n-1]` | `(lo + hi) / 2` | index or $-1$ | array not sorted |

**Memorise the pairing:** lower midpoint goes with "shrink from the right", upper midpoint goes with "shrink from the left". Get that backwards and the loop hangs.

---

## 9. `lower_bound` and `upper_bound` exactly

The two standard calls differ by one character of code and one comparison, and mixing them up is the most common source of wrong answers on ordering problems.

| Call | Returns | If all elements are smaller |
|---|---|---|
| `lower_bound(b, e, x)` | first position where `*it >= x` | `e` |
| `upper_bound(b, e, x)` | first position where `*it > x` | `e` |
| `lower_bound` on missing $x$ | insertion point that keeps the array sorted | `e` |
| count of $x$ | `upper_bound - lower_bound` | 0 |

For a sorted array `1 3 3 5 7`:

| Query | `lower_bound` index | `upper_bound` index | Count |
|---|---|---|---|
| 3 | 1 | 3 | 2 |
| 4 | 3 | 3 | 0 |
| 1 | 0 | 1 | 1 |
| 7 | 4 | 5 | 1 |
| 8 | 5 | 5 | 0 |

Note that for a missing value the two indices are equal — that is the "insertion point", and it is how `lower_bound` answers "how many elements are strictly less than $x$?" in one call.

---

## 10. Solved Problem 3 — `lower_bound` and `upper_bound` by Hand

> **Statement.** Given a sorted array and $q$ queries, for each query value $x$ output two indices: the first position with value $\ge x$ and the first position with value $> x$, using 0-based indexing and $n$ for "not found".

### Why this approach

Writing these by hand once makes the standard functions trustworthy — and some problems require the bound with a custom predicate anyway. Both are Template 1 with different comparison operators.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int lowerBound(const vector<int> &a, int x) {
    int lo = 0, hi = (int)a.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < x) lo = mid + 1;      // too small: move right
        else            hi = mid;          // >= x: this could be the answer
    }
    return lo;
}

int upperBound(const vector<int> &a, int x) {
    int lo = 0, hi = (int)a.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] <= x) lo = mid + 1;     // <= x: move right
        else             hi = mid;         // > x: candidate
    }
    return lo;
}

int main() {
    int n, q;
    cin >> n >> q;

    vector<int> a(n);
    for (auto &x : a) cin >> x;

    while (q--) {
        int x;
        cin >> x;
        cout << lowerBound(a, x) << ' ' << upperBound(a, x) << '\n';
    }

    return 0;
}
```

**Input**
```text
5 3
1 3 3 5 7
3
4
8
```

**Output**
```text
1 3
3 3
5 5
```

Matching the table in §9 exactly: for $x = 3$ the bounds are 1 and 3; for the missing 4 both are 3; for 8 both are 5, which is $n$.

**Complexity.** $O(\log n)$ per query, $O(n \log n)$ total against the $O(n)$ input.

### The trap

`hi = n - 1` in Template 1. Then the answer `n` ("not found, goes at the end") is unreachable, and every query above the maximum returns the wrong index. The half-open range `[0, n)` is what makes the "not found" case fall out naturally.

---

## 11. Binary search on unsorted data

The array does not need to be sorted. **The predicate needs to be monotone over the index.**

A classic: an array that is increasing, then decreasing (a "mountain"), and you must find the peak.

```cpp
int lo = 0, hi = n - 1;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (a[mid] < a[mid + 1]) lo = mid + 1;   // still climbing
    else                     hi = mid;       // at or past the peak
}
// lo == peak index
```

The array is not sorted. But the predicate "is the peak at or to the right of `mid`?" is monotone in `mid`, which is all binary search requires.

**The generalisation:** define `P(i)` = "the answer is at position $\ge i$". If `P` is monotone, binary search works, sorted or not.

Other unsorted-array applications:

| Problem | Monotone predicate |
|---|---|
| Find a peak in a mountain array | `a[mid] < a[mid+1]` |
| Find the rotation point in a rotated sorted array | `a[mid] > a[hi]` |
| Find the minimum in a bitonic array | same as peak, one index over |
| Find a local minimum in a valley array | `a[mid] > a[mid+1]` |

---

## 12. Binary search on the answer

This is the version that stops beginners, and it is the most valuable idea in the topic.

**The setup:** the answer is a number $x$ in a known range, and you cannot compute it directly — but you *can* answer "is $x$ feasible?" quickly.

**The recipe:**

1. Prove the feasibility predicate is monotone: if $x$ works, does $x+1$ also work? (or the reverse).
2. Find a range $[\text{lo}, \text{hi}]$ that certainly contains the answer, with `lo` infeasible or equal to the answer.
3. Binary search with Template 1 (minimise) or Template 2 (maximise).
4. Each feasibility check costs $O(f)$; total $O(f \log(\text{range}))$.

**The two orientations:**

| Goal | Predicate shape | Template | Loop body |
|---|---|---|---|
| **Minimise** $x$ such that feasible | `false ... false true ... true` | 1 | `if (ok(mid)) hi = mid; else lo = mid + 1;` |
| **Maximise** $x$ such that feasible | `true ... true false ... false` | 2 | `if (ok(mid)) lo = mid; else hi = mid - 1;` |

**The usual mistake is not the binary search; it is the feasibility function.** Write it as a separate function, test it on hand-made inputs at small, medium and boundary values, and only then wire it into the search.

---

## 13. Solved Problem 4 — Split Array, Minimise the Largest Part

> **Statement.** Split an array into at most $k$ contiguous parts. Minimise the largest part sum. $n \le 2 \times 10^5$, $a_i \ge 0$.

### Why this approach

The answer is a number (a sum), and you cannot compute it directly. But "can the array be split into at most $k$ parts each with sum $\le X$?" is easy and greedy, and it is **monotone**: if limit $X$ works, any larger limit also works. That is exactly the binary-search-on-answer profile.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<long long> a(n);
    long long sum = 0, mx = 0;
    for (auto &x : a) { cin >> x; sum += x; mx = max(mx, x); }

    // Can we cover the array with at most k parts, each summing to <= limit?
    auto ok = [&](long long limit) {
        long long parts = 1, cur = 0;
        for (long long x : a) {
            if (x > limit) return false;          // impossible regardless of parts
            if (cur + x > limit) { parts++; cur = x; }
            else cur += x;
        }
        return parts <= k;
    };

    long long lo = mx, hi = sum;                  // answer is in [mx, sum]
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid;                    // feasible: try smaller
        else         lo = mid + 1;                // infeasible: must be bigger
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
5 2
7 2 5 10 8
```

**Output**
```text
18
```

With limit 18: `[7, 2, 5]` sums to 14, `[10, 8]` sums to 18 — two parts, feasible. With limit 17, the 18 needed by the second group cannot fit, and no other split does better, so 17 is infeasible. The boundary sits at 18.

**Complexity.** $O(n \log(\sum a_i))$ — each check is $O(n)$, and the range shrinks by half each time.

### The trap

Setting `lo = 0` instead of `lo = max(a_i)`. Binary search still terminates and often still finds the right answer, but the `x > limit` guard inside `ok` becomes load-bearing, and forgetting that guard then silently returns a wrong value. Starting at `mx` makes the lower bound *true by construction*.

Note also `k` as `long long`: the comparison `parts <= k` is fine with `int` here, but the lambda captures `k` and mixed-type comparisons in lambdas are a common source of sign bugs.

---

## 14. Solved Problem 5 — Maximise the Minimum Distance

> **Statement.** Place $k$ items in $n$ positions on a line so that the smallest distance between any two chosen positions is as large as possible. $n \le 10^5$.

### Why this approach

The answer is a distance, and "can we place $k$ items with every gap $\ge d$?" is a greedy left-to-right scan. That predicate is **monotone the other way**: if gap $d$ works, any smaller gap also works. So this is the maximise orientation — Template 2, with the upper midpoint.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    cin >> n >> k;

    vector<long long> pos(n);
    for (auto &x : pos) cin >> x;
    sort(pos.begin(), pos.end());

    // Greedy: place as early as possible; if k fit, gap d is achievable.
    auto ok = [&](long long d) {
        int placed = 1;
        long long last = pos[0];
        for (int i = 1; i < n; i++) {
            if (pos[i] - last >= d) {
                placed++;
                last = pos[i];
            }
        }
        return placed >= k;
    };

    long long lo = 0, hi = pos[n-1] - pos[0];     // d = 0 always works
    while (lo < hi) {
        long long mid = lo + (hi - lo + 1) / 2;   // upper midpoint
        if (ok(mid)) lo = mid;                    // feasible: try larger
        else         hi = mid - 1;                // infeasible: smaller
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
5 3
1 2 4 8 9
```

**Output**
```text
3
```

With gap 3, choose 1, 4, 8 — all gaps are at least 3. With gap 4, after placing at 1 the next must be at least 5, so only 8 qualifies, and a third position would need to be at 12 — only two items fit.

**Complexity.** $O(n \log(\text{span}))$ after an $O(n \log n)$ sort.

### The trap

Using the lower midpoint `(lo + hi) / 2` in the maximise orientation. When the range narrows to `lo = 2, hi = 3` and `ok(2)` is true, `mid = 2` sets `lo = 2` again — no progress, infinite loop. The `+ 1` in `(lo + hi + 1) / 2` is the whole fix, and it is the single most common reason a binary search hangs.

---

## 15. Real-valued binary search

When the answer is a real number, do not test for convergence with an epsilon — run a **fixed number of iterations**. Sixty iterations shrink the range by $2^{60}$, which is far beyond `double` precision, so the loop always terminates on its own terms.

```cpp
double lo = 0, hi = 1e9;
for (int it = 0; it < 100; it++) {
    double mid = (lo + hi) / 2;
    if (ok(mid)) hi = mid;
    else         lo = mid;
}
// lo and hi are both within 1e-30 of the answer; print lo
```

| Approach | Risk |
|---|---|
| `while (hi - lo > 1e-9)` | Can loop forever if `hi - lo` underflows or oscillates |
| Fixed 100 iterations | None — provably terminates |
| Fixed 60 iterations | Enough for `double` for ranges up to $10^{18}$ |

**Cost note:** 100 iterations of an $O(n)$ check is $100n$, roughly the same as $O(n \log n)$ at typical sizes. Only reduce the iteration count if the checks are expensive.

---

## 16. Complexity summary

| Technique | Time | Space | Requires |
|---|---|---|---|
| Two ends converging | $O(n)$ | $O(1)$ | sorted array |
| Slow/fast write pointer | $O(n)$ | $O(1)$ | in-place compaction |
| Sliding window | $O(n)$ | $O(1)$ or $O(\text{alphabet})$ | monotone window property |
| Binary search (one query) | $O(\log n)$ | $O(1)$ | sorted, or monotone predicate |
| Binary search, $q$ queries | $O((n + q) \log n)$ | $O(n)$ | sort once |
| Binary search on answer | $O(f \log(\text{range}))$ | $O(1)$ extra | monotone feasibility check |
| Real binary search | $O(f \cdot 100)$ | $O(1)$ extra | monotone check |

---

## 17. Unsolved problems

Five problems, no solutions. Identify the pattern, then the invariant, then code.

| # | Problem | Constraints |
|---|---|---|
| 1 | **Container With Most Water.** Heights $h_i$ at positions $i$. Choose $i < j$ maximising $\min(h_i, h_j)\cdot(j-i)$. | $n \le 2 \times 10^5$ |
| 2 | **Remove Duplicates In Place.** From a sorted array, delete duplicates in place and output the new length followed by the resulting array. | $n \le 2 \times 10^5$, $O(1)$ extra space |
| 3 | **Minimum Eating Speed.** $n$ piles; in one hour you eat up to $v$ bananas from a single pile. Find the minimum $v$ that clears every pile within $h$ hours. | $n \le 10^4$, $a_i \le 10^9$, $h \le 10^9$ |
| 4 | **Count Pairs With Difference at Most $K$.** In a sorted array, count pairs $i < j$ with $a_j - a_i \le K$. | $n \le 2 \times 10^5$ |
| 5 | **Minimum Length Subarray With Sum $\ge S$.** All elements are positive. Find the shortest contiguous subarray with sum at least $S$, or `0` if none exists. | $n \le 2 \times 10^5$ |

**Pointers, in order of how much they give away:** problem 2 is Pattern B with `keep()` testing against `a[slow-1]`. Problems 1 and 5 are Pattern A and Pattern C respectively. Problem 3 is binary-search-on-answer with the *minimise* orientation; the feasibility check is a sum of ceilings, not a greedy scan.

---

## Key Takeaways

- Two pointers and binary search both work by **ruling out candidates without examining them**; the proof obligation is that the ruled-out set really cannot contain the answer.
- Three two-pointer patterns cover everything: converging ends, slow/fast write, sliding window.
- Converging ends needs sortedness; the proof is "if the sum is too small, `i` cannot pair with anything remaining".
- The sliding window is valid only when the window property is monotone. Negative numbers usually break it.
- There are exactly three binary-search templates, and the midpoint choice is coupled to the branch: lower midpoint with `hi = mid`, upper midpoint with `lo = mid`.
- `lower_bound` is "first $\ge x$", `upper_bound` is "first $> x$", their difference is the count, and both return `n` when not found — which requires a half-open `hi = n`.
- Binary search does **not** require a sorted array; it requires a monotone predicate over the index. Peak-finding is the proof.
- Binary search on the answer = monotone feasibility check + a range that contains the answer.
- Minimise uses Template 1; maximise uses Template 2. Mixing them causes an infinite loop, not a wrong answer — which is at least a loud failure.
- `lo` must be a value that is genuinely feasible (or the answer itself): `max(a_i)` for the split problem, `0` for a distance problem.
- Real-valued binary search uses a fixed iteration count, never an epsilon comparison.
- Test the feasibility function separately before wiring it into the search. That is where the bugs are.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Two ends converging on an unsorted array | The discard argument fails | Sort first |
| 2 | Sliding window with negative values | Window property not monotone | Prefix + map |
| 3 | `while (i <= j)` on a pair problem | Element paired with itself | `i < j` |
| 4 | Re-scanning the window each step | $O(n^2)$ | Maintain incremental state |
| 5 | Template 1 with `hi = n - 1` | Cannot return "not found" = `n` | `hi = n` |
| 6 | Lower midpoint in Template 2 | Infinite loop at `lo + 1 == hi` | `(lo + hi + 1) / 2` |
| 7 | Upper midpoint in Template 1 | Skips the first true index | `(lo + hi) / 2` |
| 8 | `lower_bound` confused with `upper_bound` | Counts are off by the frequency | `upper - lower` is the count |
| 9 | Binary search on a non-monotone predicate | Nondeterministic wrong answers | Prove monotonicity first |
| 10 | `lo = 0` for a split problem | Feasibility check becomes load-bearing | `lo = max(a_i)` |
| 11 | `hi = 0` in the maximise orientation | Answer range excludes the answer | `hi` must be feasible |
| 12 | `while (hi - lo > eps)` on doubles | May never terminate | Fixed 100 iterations |
| 13 | `(lo + hi) / 2` with large `lo`, `hi` | `int` overflow gives a negative mid | `lo + (hi - lo) / 2` |
| 14 | Recursive binary search | Depth is small but the call overhead is avoidable | Iterative loop |
| 15 | `n == 0` with `hi = n - 1` | `hi = -1` underflows to a huge `size_t` | Handle empty input first |
| 16 | Off-by-one in window length | `right - left` instead of `right - left + 1` | Count inclusive |

---

## Practice Problems

The five unsolved problems in §17 are the practice set.

| Order | Problem | Pattern | Why this order |
|---|---|---|---|
| 1 | §17.2 Remove Duplicates | B | Simplest invariant, no predicate to prove |
| 2 | §17.1 Container With Most Water | A | Convergence proof is a one-liner |
| 3 | §17.5 Minimum Length Subarray | C | Window with a "shrink while valid" loop |
| 4 | §17.4 Count Pairs With Difference $\le K$ | A | Convergence plus counting all pairs at once |
| 5 | §17.3 Minimum Eating Speed | Answer search | The check is arithmetic, not greedy |

For more, use the CSES *Sorting and Searching* section — it contains a direct application of every pattern in this topic.

---

## Next Up

→ [Topic 03 — Advanced Searching and Sorting Part 2](Topic%2003%20-%20Advanced%20Searching%20and%20Sorting%20Part%202%20%28Custom%20Comparators%2C%20Coordinate%20Compression%20and%20Greedy%20Sorting%29.md)

Both techniques so far assumed you could compare elements the obvious way. The next topic makes comparison a parameter: custom orderings that encode problem-specific rules, coordinate compression that turns sparse values into dense indices, and the greedy sorts whose correctness depends entirely on which comparator you choose.
