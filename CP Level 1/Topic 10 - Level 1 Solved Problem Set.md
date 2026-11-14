# Topic 10 — Level 1 Solved Problem Set

Twenty problems, drawn from every technique in Chapter 2. Each one is solved in full, and each starts with the same question a contest forces on you: **the constraints are on the screen — what do they permit?**

That question is the whole skill. A contestant who can look at $n \le 2 \times 10^5$ and immediately know "$\mathcal{O}(n \log n)$, no nested loops" solves four problems in the time it takes another to write a brute force, watch it TLE, and start over.

> **Prerequisites:** Topics [04](Topic%2004%20-%20Complete%20Searching.md) through [09](Topic%2009%20-%20Rectangle%20Geometry%20and%20Ad%20Hoc%20Problems.md) of this chapter.
> **You will be able to:**
> - Read a constraint and name the technique before writing a line.
> - Recognise the eight recurring Level 1 patterns on sight.
> - Explain, for each solution, *why* the approach was chosen rather than only how it works.

---

## 1. How to use this set

Each problem is presented with its reasoning first:

- **Statement** — the problem, with constraints.
- **Why this approach** — the constraint-to-technique deduction. This is the part worth reading twice.
- **Solution** — the idea in prose, then the code.
- **Complexity** — time and space.

Do not skim the "why" sections. The data structures and algorithms are already in Chapter 1 and in Topics 04–09; what this set adds is the *selection* step.

---

## 2. The decision table

| Constraint | Technique | Typical time |
|---|---|---|
| $n \le 20$ | Complete search, bitmask | $\mathcal{O}(2^n)$ |
| $n \le 500$ | $\mathcal{O}(n^2)$ pairs, or $\mathcal{O}(n^3)$ triples | |
| $n \le 2 \times 10^5$ | Sort, map, two pointers, greedy, BFS/DFS | $\mathcal{O}(n \log n)$ |
| $n \le 10^9$ | Closed form, or $\mathcal{O}(\sqrt{n})$ / $\mathcal{O}(\log n)$ | |
| Answer is a number with a formula | Do the arithmetic; do not simulate | $\mathcal{O}(1)$ |
| "Count pairs / subarrays" | Sort + two pointers, or prefix + map | $\mathcal{O}(n \log n)$ |
| "Find a path / reachability" | BFS or DFS | $\mathcal{O}(n + m)$ |
| "Minimum moves on a grid / board" | BFS on an implicit graph | $\mathcal{O}(\text{states})$ |

---

## 3. Problem 1 — K-th Largest Element

> **Statement.** Given $n$ integers and $k$, output the $k$-th largest (1-indexed). $n \le 2 \times 10^5$, values fit in `int`.

**Why this approach.** There is no structure to exploit — no sortedness, no pattern. $n = 2 \times 10^5$ rules out $\mathcal{O}(n^2)$. Sorting gives $\mathcal{O}(n \log n)$, which is comfortably fast, and the answer is then a single index.

**Solution.** Sort ascending; the $k$-th largest is at index $n - k$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    cin >> n >> k;
    vector<int> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());
    cout << a[n - k] << '\n';      // k = 1 -> largest
    return 0;
}
```

**Input**
```text
5 2
3 1 4 1 5
```

**Output**
```text
4
```

Sorted: 1, 1, 3, 4, 5. The 2nd largest is 4.

**Complexity.** $\mathcal{O}(n \log n)$ time, $\mathcal{O}(1)$ extra.

> **Faster, rarely needed:** `nth_element(a.begin(), a.begin() + (n - k), a.end())` is $\mathcal{O}(n)$ average because it only partitions. Use it when $n$ is $10^7$ or the time limit is tight. For $n = 2 \times 10^5$ the sort is faster in practice.

---

## 4. Problem 2 — Two Sum Indices

> **Statement.** Given $n$ integers and a target $T$, find two distinct indices $i < j$ with $a_i + a_j = T$. Output any such pair, or `-1`. $n \le 2 \times 10^5$.

**Why this approach.** The answer depends on the original positions, so sorting is off the table — it destroys the indices. A hash map from value to index answers "have I already seen $T - a_i$?" in $\mathcal{O}(1)$, giving one pass.

**Solution.** Sweep left to right, checking each value against the map before inserting it. Checking *before* inserting is what guarantees $i \ne j$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long T;
    cin >> n >> T;

    unordered_map<long long, int> seen;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;

        auto it = seen.find(T - x);
        if (it != seen.end()) {
            cout << it->second + 1 << ' ' << i + 1 << '\n';
            return 0;
        }
        seen[x] = i;
    }

    cout << -1 << '\n';
    return 0;
}
```

**Input**
```text
4 9
2 7 11 15
```

**Output**
```text
1 2
```

**Complexity.** $\mathcal{O}(n)$ expected time, $\mathcal{O}(n)$ space.

> **The duplicate trap:** if the array is `2 2` and $T = 4$, checking before inserting handles it correctly — the first `2` is inserted, the second finds it. Inserting first and then searching would match an index with itself.

---

## 5. Problem 3 — Longest Consecutive Run

> **Statement.** Given $n$ integers, find the length of the longest set of consecutive integers present in the array, ignoring order and duplicates. $n \le 2 \times 10^5$.

**Why this approach.** Sorting brings equal values together and orders them, so "consecutive" becomes "differs by exactly 1 from the previous element". One pass over the sorted array is enough.

**Solution.** Sort, then walk. Skip duplicates; extend the run when `a[i] == a[i-1] + 1`; reset otherwise.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<int> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());

    int best = 1, run = 1;
    for (int i = 1; i < n; i++) {
        if (a[i] == a[i - 1]) continue;          // duplicate: run unchanged
        if (a[i] == a[i - 1] + 1) run++;         // extends the run
        else run = 1;                            // broken
        best = max(best, run);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
6
100 4 200 1 3 2
```

**Output**
```text
4
```

Sorted: 1, 2, 3, 4, 100, 200. The run 1–4 has length 4.

**Complexity.** $\mathcal{O}(n \log n)$ time, $\mathcal{O}(1)$ extra.

---

## 6. Problem 4 — Pairs With Difference $K$

> **Statement.** Count pairs $i < j$ with $|a_i - a_j| = K$. $n \le 2 \times 10^5$, $K \ge 0$.

**Why this approach.** Counting all pairs is $\mathcal{O}(n^2)$, too slow. After sorting, for each $i$ the valid partners form a contiguous block (or nothing), so two pointers find them in one pass.

**Solution.** Sort. For each $i$, advance $j$ until $a_j - a_i \ge K$. If equality holds, count one pair. $K = 0$ is a separate case because the pointer never advances.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long K;
    cin >> n >> K;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());

    long long ans = 0;

    if (K == 0) {
        // equal values: count unordered pairs within each run
        for (int i = 0, j; i < n; i = j) {
            j = i;
            while (j < n && a[j] == a[i]) j++;
            long long len = j - i;
            ans += len * (len - 1) / 2;
        }
    } else {
        int j = 1;
        for (int i = 0; i < n; i++) {
            if (j <= i) j = i + 1;
            while (j < n && a[j] - a[i] < K) j++;
            if (j < n && a[j] - a[i] == K) ans++;
        }
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5 2
1 5 3 4 2
```

**Output**
```text
3
```

Sorted: 1, 2, 3, 4, 5. Pairs at distance 2: (1,3), (2,4), (3,5) — three.

**Complexity.** $\mathcal{O}(n \log n)$.

> **Why $K = 0$ needs its own branch:** with $K = 0$, `while (a[j] - a[i] < 0)` is never true, so `j` stays at `i + 1` and only adjacent duplicates are counted. Runs must be counted combinatorially instead.

---

## 7. Problem 5 — Subarrays With Sum Divisible by $K$

> **Statement.** Count contiguous subarrays whose sum is divisible by $K$. $n \le 2 \times 10^5$, $1 \le K \le 10^9$.

**Why this approach.** A subarray sum is $P[r] - P[l]$. It is divisible by $K$ exactly when $P[r] \equiv P[l] \pmod K$. So the problem is: count pairs of prefixes with the same remainder — a map of remainder counts.

**Solution.** Walk forward keeping the running prefix modulo $K$. Before adding the current prefix to the count, add the number of earlier prefixes with the same remainder.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long K;
    cin >> n >> K;

    map<long long, long long> cnt;
    cnt[0] = 1;                       // the empty prefix

    long long pref = 0, ans = 0;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        pref = ((pref + x) % K + K) % K;   // normalise for negatives
        ans += cnt[pref];
        cnt[pref]++;
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5 2
4 5 0 2 3
```

**Output**
```text
6
```

Prefix sums: 0 (empty), 4, 9, 9, 11, 14. Parities: 0, 0, 1, 1, 1, 0. Three prefixes are even and three are odd, so the number of same-parity pairs is $\binom{3}{2} + \binom{3}{2} = 6$.

**Complexity.** $\mathcal{O}(n \log n)$ with `map`; $\mathcal{O}(n)$ expected with `unordered_map` and a safe hash.

> **The negative-modulo trap:** in C++, `-3 % 2 == -1`, not `1`. Always normalise with `((x % K) + K) % K` when values can be negative.

---

## 8. Problem 6 — Top $k$ Frequent Elements

> **Statement.** Given $n$ integers and $k$, output the $k$ most frequent values, in decreasing order of frequency. $n \le 2 \times 10^5$.

**Why this approach.** "How many times" is a frequency count, which is a map. The ordering requirement then needs a sort by count, which is a second pass over the distinct values only — at most $n$ of them.

**Solution.** Count with a map, copy the pairs into a vector, sort by count descending, take the first $k$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    cin >> n >> k;

    map<int,int> freq;
    for (int i = 0; i < n; i++) {
        int x;
        cin >> x;
        freq[x]++;
    }

    vector<pair<int,int>> v;            // (count, value)
    for (auto &p : freq) v.push_back({p.second, p.first});

    sort(v.begin(), v.end(), greater<>());

    for (int i = 0; i < k && i < (int)v.size(); i++)
        cout << v[i].second << " \n"[i + 1 == k];

    return 0;
}
```

**Input**
```text
6 2
1 1 1 2 2 3
```

**Output**
```text
1 2
```

Counts: 1 appears 3 times, 2 twice, 3 once. The top two are 1 and 2.

**Complexity.** $\mathcal{O}(n + d \log d)$ where $d$ is the number of distinct values.

> **Tie-breaking matters:** with `greater<>` on `(count, value)`, ties are broken by the larger value. If the judge requires a specific order among equal counts, read the statement — many problems ask for "any order" and then any tie-break is fine.

---

## 9. Problem 7 — Majority Element

> **Statement.** Given $n$ integers, one value occurs more than $\lfloor n/2 \rfloor$ times. Find it in $\mathcal{O}(n)$ time and $\mathcal{O}(1)$ space.

**Why this approach.** The $\mathcal{O}(1)$ space requirement rules out a frequency map. The guaranteed majority is a strong structural promise, and Boyer–Moore exploits exactly that promise.

**Solution.** Keep a candidate and a counter. Matching values increment, different values decrement, and a zero counter adopts a new candidate. Because the majority element appears more than half the time, it survives every cancellation.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    long long cand = 0;
    int cnt = 0;

    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        if (cnt == 0) { cand = x; cnt = 1; }
        else if (x == cand) cnt++;
        else cnt--;
    }

    cout << cand << '\n';
    return 0;
}
```

**Input**
```text
7
2 2 1 1 1 2 2
```

**Output**
```text
2
```

**Complexity.** $\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space.

> **When the promise is absent,** Boyer–Moore returns *a* candidate, not necessarily a majority. Add a verification pass counting the candidate, and output `-1` if the count does not exceed $n/2$.

---

## 10. Problem 8 — Maximum Subarray Sum

> **Statement.** Find the maximum sum of a non-empty contiguous subarray. $n \le 2 \times 10^5$, values up to $10^9$ in magnitude.

**Why this approach.** A brute force over all subarrays is $\mathcal{O}(n^2)$. Kadane's observation is that the best subarray ending at $i$ either extends the best one ending at $i-1$, or starts fresh at $i$ — so only one value needs carrying forward.

**Solution.** `cur = max(x, cur + x)`, `best = max(best, cur)`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    long long best = LLONG_MIN, cur = 0;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        cur = max(x, cur + x);     // extend, or restart here
        best = max(best, cur);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
8
-2 1 -3 4 -1 2 1 -5
```

**Output**
```text
6
```

The subarray `4 -1 2 1` sums to 6.

**Complexity.** $\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space.

> **The all-negative trap:** initialising `best = 0` forces the answer to 0 and is wrong when every element is negative. Initialise to `LLONG_MIN`, or to `a[0]` after reading it. This is the single most common Kadane bug.

---

## 11. Problem 9 — Rotate Array Right by $k$

> **Statement.** Rotate an array right by $k$ positions, in place. $n \le 2 \times 10^5$.

**Why this approach.** The obvious approach uses a temporary array but needs $\mathcal{O}(n)$ extra space. The reversal trick achieves the same rotation with three in-place reversals and no allocation.

**Solution.** Reverse the whole array, then reverse the first $k$ and the last $n-k$ separately.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<int> a(n);
    for (auto &x : a) cin >> x;

    k %= n;                                   // rotations wrap
    reverse(a.begin(), a.end());
    reverse(a.begin(), a.begin() + k);
    reverse(a.begin() + k, a.end());

    for (int i = 0; i < n; i++) cout << a[i] << " \n"[i + 1 == n];
    return 0;
}
```

**Input**
```text
5 2
1 2 3 4 5
```

**Output**
```text
4 5 1 2 3
```

**Complexity.** $\mathcal{O}(n)$ time, $\mathcal{O}(1)$ extra space.

> **Why it works:** reversing the whole array moves the last $k$ elements to the front but in reverse order; the two partial reversals restore their internal order. Draw the three steps once and the trick is permanent.

---

## 12. Problem 10 — Spiral Order of a Matrix

> **Statement.** Output the elements of an $R \times C$ matrix in clockwise spiral order starting from the top-left.

**Why this approach.** Pure simulation. There is no formula; the only way is to follow the boundary. Doing it with four shrinking boundaries avoids the index-arithmetic errors of the naive "walk until visited" approach.

**Solution.** Maintain `top`, `bottom`, `left`, `right`. Walk one edge, shrink that boundary, repeat. The two `if` guards prevent re-walking a single remaining row or column.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int R, C;
    cin >> R >> C;
    vector<vector<int>> m(R, vector<int>(C));
    for (auto &row : m) for (auto &x : row) cin >> x;

    int top = 0, bottom = R - 1, left = 0, right = C - 1;
    vector<int> out;

    while (top <= bottom && left <= right) {
        for (int c = left; c <= right; c++) out.push_back(m[top][c]);
        top++;
        for (int r = top; r <= bottom; r++) out.push_back(m[r][right]);
        right--;
        if (top <= bottom) {
            for (int c = right; c >= left; c--) out.push_back(m[bottom][c]);
            bottom--;
        }
        if (left <= right) {
            for (int r = bottom; r >= top; r--) out.push_back(m[r][left]);
            left++;
        }
    }

    for (int i = 0; i < (int)out.size(); i++)
        cout << out[i] << " \n"[i + 1 == (int)out.size()];
    return 0;
}
```

**Input**
```text
3 3
1 2 3
4 5 6
7 8 9
```

**Output**
```text
1 2 3 6 9 8 7 4 5
```

**Complexity.** $\mathcal{O}(RC)$ time, $\mathcal{O}(RC)$ output.

> **Why the guards are required:** for a $1 \times 3$ matrix, after the top row is consumed `top > bottom`, but the naive code would still walk the bottom row — printing the middle row twice. The `if (top <= bottom)` guard prevents exactly that.

---

## 13. Problem 11 — Balanced Parentheses

> **Statement.** Given a string of `(` and `)`, decide whether it is a valid balanced sequence.

**Why this approach.** Only the running depth matters; the structure is a single counter. A stack is unnecessary because there is only one bracket type — a fact worth noticing, since with two types the stack becomes mandatory.

**Solution.** Increment on `(`, decrement on `)`. Fail immediately if the depth goes negative; at the end require depth 0.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s;
    cin >> s;

    int depth = 0;
    bool ok = true;

    for (char c : s) {
        if (c == '(') depth++;
        else {
            depth--;
            if (depth < 0) { ok = false; break; }
        }
    }
    if (depth != 0) ok = false;

    cout << (ok ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
(()())
```

**Output**
```text
YES
```

**Complexity.** $\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space.

> **Why one counter is not enough for `()[]{}`:** `([)]` has a non-negative depth throughout but is invalid. The *order* of closing matters, which is exactly what a stack records and a counter cannot.

---

## 14. Problem 12 — Strictly Increasing After One Deletion

> **Statement.** Decide whether the array can be made strictly increasing by deleting at most one element. $n \le 2 \times 10^5$.

**Why this approach.** Trying every deletion is $\mathcal{O}(n^2)$. The key observation is that a single violation splits the array into a prefix and suffix that are each already increasing — so only three conditions need checking, all in $\mathcal{O}(1)$ per index.

**Solution.** Precompute `pre[i]` = "`a[0..i]` is strictly increasing" and `suf[i]` = "`a[i..n-1]` is strictly increasing". Deleting index $i$ works iff the prefix is fine, the suffix is fine, and the join `a[i-1] < a[i+1]` holds.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    if (n <= 2) { cout << "YES\n"; return 0; }

    vector<char> pre(n, 1), suf(n, 1);
    for (int i = 1; i < n; i++) pre[i] = pre[i-1] && (a[i-1] < a[i]);
    for (int i = n - 2; i >= 0; i--) suf[i] = suf[i+1] && (a[i] < a[i+1]);

    for (int i = 0; i < n; i++) {
        bool left  = (i == 0) || pre[i-1];
        bool right = (i == n - 1) || suf[i+1];
        bool join  = (i == 0 || i == n - 1) || (a[i-1] < a[i+1]);
        if (left && right && join) { cout << "YES\n"; return 0; }
    }

    cout << "NO\n";
    return 0;
}
```

**Input**
```text
4
1 3 2 4
```

**Output**
```text
YES
```

Deleting the 3 leaves `1 2 4`, strictly increasing.

**Complexity.** $\mathcal{O}(n)$ time, $\mathcal{O}(n)$ space (two `char` arrays).

> **The join condition is the subtle one:** deleting $a_i$ makes $a_{i-1}$ and $a_{i+1}$ adjacent, so their relative order must be checked separately. Without it, the test only verifies that the two remaining pieces are each internally sorted, not that they join correctly. For $a = [1, 2, 0, 1]$, deleting index 1 leaves the prefix $[1]$ and the suffix $[0,1]$ both increasing, but joins $1$ to $0$ — the join check is what rejects it, and the answer is correctly `NO`.

---

## 15. Problem 13 — Counting Triangle Triples

> **Statement.** Given $n$ side lengths, count triples of indices that form a non-degenerate triangle. $n \le 5000$.

**Why this approach.** A triangle needs $a + b > c$ for the two smaller sides. Sorting makes the largest side identifiable, and for each largest side a two-pointer scan counts valid pairs — $\mathcal{O}(n^2)$ instead of $\mathcal{O}(n^3)$.

**Solution.** Sort. For each $k$ as the largest side, place $i$ at 0 and $j$ at $k-1$. If $a_i + a_j > a_k$, then every index from $i$ to $j-1$ paired with $j$ also works, so add $j - i$ and decrement $j$; otherwise increment $i$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());

    long long ans = 0;
    for (int k = 2; k < n; k++) {
        int i = 0, j = k - 1;
        while (i < j) {
            if (a[i] + a[j] > a[k]) {
                ans += j - i;      // all pairs (i..j-1, j)
                j--;
            } else {
                i++;
            }
        }
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
4
2 3 4 5
```

**Output**
```text
3
```

Triples: (2,3,4), (2,4,5), (3,4,5). The triple (2,3,5) is degenerate ($2+3=5$) and excluded.

**Complexity.** $\mathcal{O}(n^2)$ after an $\mathcal{O}(n \log n)$ sort.

> **Why `>` and not `>=`:** the problem asks for non-degenerate triangles, so equality is excluded. Degenerate triples ($a + b = c$) collapse to a line segment and do not count.

---

## 16. Problem 14 — Lemonade Change

> **Statement.** Customers pay with 5, 10 or 20 in order; each lemonade costs 5. Starting with no change, decide whether every customer can be served.

**Why this approach.** Greedy is forced: you must give change immediately, and the only choice is which bills to hand back. The correct policy is to spend the largest bills first, because they are the least flexible.

**Solution.** Track counts of 5s and 10s. For a 20, prefer `10 + 5` over `5 + 5 + 5`, since a 10 is useless for making 5 change.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    int five = 0, ten = 0;
    bool ok = true;

    for (int i = 0; i < n; i++) {
        int bill;
        cin >> bill;

        if (bill == 5) five++;
        else if (bill == 10) {
            if (five == 0) { ok = false; break; }
            five--; ten++;
        } else {                                  // 20
            if (ten > 0 && five > 0) { ten--; five--; }
            else if (five >= 3)      { five -= 3; }
            else { ok = false; break; }
        }
    }

    cout << (ok ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
5
5 5 5 10 20
```

**Output**
```text
YES
```

**Complexity.** $\mathcal{O}(n)$.

> **Why spending the 10 first is optimal:** a 10 can only ever make change for a 20 (as `10 + 5`). A 5 is needed for 10s as well. So the more specialised bill should be spent first — the standard "spend the least flexible resource" argument.

---

## 17. Problem 15 — Maximum Units on a Truck

> **Statement.** There are $n$ box types; type $i$ has `count[i]` boxes each holding `units[i]` units. The truck holds `capacity` boxes. Maximise the total units loaded.

**Why this approach.** The boxes are independent and each has a fixed value per unit of capacity. That is exactly the density-ordering condition, so greedy by units descending is optimal — and the exchange argument is one line.

**Solution.** Sort by units descending, take as many of each type as fit.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long capacity;
    cin >> n >> capacity;

    vector<pair<int,int>> box(n);     // (units, count)
    for (auto &p : box) cin >> p.first >> p.second;

    sort(box.begin(), box.end(), greater<>());

    long long total = 0;
    for (auto &p : box) {
        long long take = min(capacity, (long long)p.second);
        total += take * p.first;
        capacity -= take;
        if (capacity == 0) break;
    }

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
3 10
5 2
10 3
7 4
```

**Output**
```text
68
```

Take 3 boxes of 10 (30 units, 7 slots left), then 4 boxes of 7 (28 units, 3 slots left), then 2 boxes of 5 (10 units, 1 slot left — and no boxes remain). Total $30 + 28 + 10 = 68$.

**Complexity.** $\mathcal{O}(n \log n)$.

> **The exchange argument:** if a solution takes a box of lower units while a higher-units box is available, swapping them strictly increases the total and does not change the capacity used. So the sorted order is optimal.

---

## 18. Problem 16 — Minimum Moves to Equalise II

> **Statement.** In one move, increment or decrement any element by 1. Find the minimum number of moves to make all elements equal. $n \le 2 \times 10^5$.

**Why this approach.** The cost for a target value $t$ is $\sum |a_i - t|$, and the sum of absolute deviations is minimised at a **median**. No search, no DP — just a sort and one pass.

**Solution.** Sort, take the median, sum the distances.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());
    long long med = a[n / 2];

    long long moves = 0;
    for (long long x : a) moves += llabs(x - med);

    cout << moves << '\n';
    return 0;
}
```

**Input**
```text
5
1 2 3 4 5
```

**Output**
```text
6
```

Median 3. Distances: 2, 1, 0, 1, 2 — total 6. Target 1 would cost $0+1+2+3+4 = 10$.

**Complexity.** $\mathcal{O}(n \log n)$.

> **Why the median and not the mean:** the mean minimises the sum of *squared* deviations; the median minimises the sum of *absolute* deviations. Using the mean here gives 3 for this example (lucky) but fails on skewed data such as `1 1 1 100`.

---

## 19. Problem 17 — Minimum Throws With Teleporters

> **Statement.** A board has squares $1 \ldots N$. From square $u$ you may move to $u + d$ for any $d \in [1,6]$, then any teleporter on the landing square moves you instantly. Find the minimum number of throws to reach $N$, or `-1`.

**Why this approach.** Every edge costs one throw, so this is an unweighted shortest-path problem on an implicit graph — vertices are squares, edges are the dice moves. BFS is the right tool, and no graph needs to be built.

**Solution.** BFS from 1, expanding six neighbours per state and applying the teleporter on arrival.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int N, T;
    cin >> N >> T;

    vector<int> jump(N + 1, 0);       // 0 = no teleporter
    for (int i = 0; i < T; i++) {
        int a, b;
        cin >> a >> b;
        jump[a] = b;
    }

    vector<int> dist(N + 1, -1);
    queue<int> q;
    dist[1] = 0;
    q.push(1);

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        if (u == N) break;

        for (int d = 1; d <= 6; d++) {
            int v = u + d;
            if (v > N) break;
            if (jump[v] != 0) v = jump[v];
            if (dist[v] != -1) continue;
            dist[v] = dist[u] + 1;
            q.push(v);
        }
    }

    cout << dist[N] << '\n';
    return 0;
}
```

**Input**
```text
30 2
11 26
3 15
```

**Output**
```text
3
```

The winning route ignores the tempting early teleporter: $1 \to 5 \to 11$, which teleports to $26$, then $26 \to 30$. Three throws. The $3 \to 15$ teleporter looks better (a bigger jump, earlier) but leads to a four-throw route.

**Complexity.** $\mathcal{O}(N)$ states, $\mathcal{O}(6N)$ edges.

> **Why BFS and not DFS:** the graph is unweighted and we need the *minimum* number of throws. DFS would find a route, not the shortest one.

---

## 20. Problem 18 — Counting Provinces

> **Statement.** A graph is given as an $n \times n$ adjacency matrix. Count its connected components.

**Why this approach.** "Connected components" is a traversal question, and the input is a matrix rather than an edge list — but the technique is unchanged: DFS from every unvisited vertex, incrementing a counter.

**Solution.** Straightforward component counting over the matrix.

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> g;
vector<bool> seen;

void dfs(int u) {
    seen[u] = true;
    for (int v = 0; v < n; v++)
        if (g[u][v] && !seen[v]) dfs(v);
}

int main() {
    cin >> n;
    g.assign(n, vector<int>(n));
    seen.assign(n, false);

    for (auto &row : g) for (auto &x : row) cin >> x;

    int comps = 0;
    for (int i = 0; i < n; i++) {
        if (seen[i]) continue;
        comps++;
        dfs(i);
    }

    cout << comps << '\n';
    return 0;
}
```

**Input**
```text
4
1 1 0 0
1 1 0 0
0 0 1 0
0 0 0 1
```

**Output**
```text
3
```

Components: $\{0,1\}$, $\{2\}$, $\{3\}$.

**Complexity.** $\mathcal{O}(n^2)$ — every matrix entry is examined once.

---

## 21. Problem 19 — Shortest Path in a Binary Matrix

> **Statement.** In an $R \times C$ matrix of 0 (open) and 1 (blocked), move between open cells that share a side or a corner (8 directions). Find the shortest path from $(0,0)$ to $(R-1,C-1)$, or `-1`.

**Why this approach.** Uniform step cost, minimum path — BFS. The 8-neighbour variant changes only the direction arrays.

**Solution.** BFS from the start, recording distances; guard the two endpoints since they may be blocked.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int R, C;
    cin >> R >> C;
    vector<vector<int>> g(R, vector<int>(C));
    for (auto &row : g) for (auto &x : row) cin >> x;

    if (g[0][0] == 1 || g[R-1][C-1] == 1) { cout << -1 << '\n'; return 0; }

    const int dr[8] = {-1,-1,-1, 0, 0, 1, 1, 1};
    const int dc[8] = {-1, 0, 1,-1, 1,-1, 0, 1};

    vector<vector<int>> dist(R, vector<int>(C, -1));
    queue<pair<int,int>> q;

    dist[0][0] = 1;            // count cells, not edges
    q.push({0, 0});

    while (!q.empty()) {
        int r = q.front().first, c = q.front().second;
        q.pop();

        for (int d = 0; d < 8; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
            if (g[nr][nc] == 1 || dist[nr][nc] != -1) continue;
            dist[nr][nc] = dist[r][c] + 1;
            q.push({nr, nc});
        }
    }

    cout << dist[R-1][C-1] << '\n';
    return 0;
}
```

**Input**
```text
3 3
0 0 0
1 1 0
1 1 0
```

**Output**
```text
4
```

Four cells: $(0,0) \to (0,1) \to (1,2) \to (2,2)$. The diagonal hop from $(0,0)$ to $(1,1)$ is blocked, so the route has to go around the wall.

**Complexity.** $\mathcal{O}(RC)$.

> **Edge count versus cell count:** initialising `dist` to 1 and counting cells is a common convention; if the problem asks for *steps*, initialise to 0 and the answer is one less.

---

## 22. Problem 20 — Container With Most Water

> **Statement.** Given heights $h_i$ at positions $i$, choose $i < j$ maximising $\min(h_i, h_j) \cdot (j - i)$.

**Why this approach.** Trying all pairs is $\mathcal{O}(n^2)$. Two pointers from both ends works because the shorter wall is the bottleneck: moving the taller side can never help, since the width shrinks and the height is still capped by the shorter wall.

**Solution.** Start at both ends, compute the area, and move the pointer at the shorter wall inward.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<long long> h(n);
    for (auto &x : h) cin >> x;

    int i = 0, j = n - 1;
    long long best = 0;

    while (i < j) {
        best = max(best, min(h[i], h[j]) * (long long)(j - i));
        if (h[i] < h[j]) i++;
        else             j--;
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
9
1 8 6 2 5 4 8 3 7
```

**Output**
```text
49
```

Indices 1 and 8: height $\min(8,7) = 7$, width $8 - 1 = 7$, area 49.

**Complexity.** $\mathcal{O}(n)$ — each pointer moves inward at most $n$ times.

> **Why discarding is safe:** when $h_i \le h_j$, any pair using $i$ with a smaller width has height $\le h_i$, so it cannot beat the current area using $j$. So $i$ can never be part of a better answer, and discarding it loses nothing.

---

## 23. Where the difficulty actually is

Reviewing the twenty, the code is rarely the hard part. The difficulty was in one of four places:

| Difficulty | Where it appeared |
|---|---|
| **Choosing the technique** | Problems 2, 17, 20 — the answer is a different tool than the obvious one |
| **A boundary or degenerate case** | Problems 3, 4, 8, 10, 12, 13 — $K=0$, all-negative, $1 \times n$, equality |
| **A non-obvious invariant** | Problems 7, 14, 16, 20 — majority, least-flexible bill, median, shorter wall |
| **Implicit graph modelling** | Problems 17, 18, 19 — the graph is not given, it must be seen |

That distribution is not a coincidence. Level 1 problems are almost never about implementing a hard algorithm; they are about **seeing the problem correctly**.

---

## Key Takeaways

- Read the constraint first and name the technique before writing anything.
- $n \le 2 \times 10^5$ means $\mathcal{O}(n \log n)$ or better — never a nested loop.
- If the answer is a formula (median, range, count), compute it; do not simulate or binary search.
- Sorting converts many "compare all pairs" problems into one pass, but destroys indices — store `(value, index)` when positions matter.
- Prefix sums plus a map solve every "count subarrays with property $P$" question where $P$ is additive or modular.
- **Degenerate inputs are the grading criteria.** $K=0$, all negatives, empty runs, $1 \times n$ matrices, equal sides — each appears in at least one of these twenty.
- Boyer–Moore, Kadane, and the median rule are invariants, not tricks: each relies on a specific promise in the statement.
- Greedy ordering by density or by "least flexible resource first" is the same argument wearing different clothes.
- An implicit graph (a board, a set of states) needs only BFS — no adjacency list must be built.
- Two pointers from both ends works when one side is provably the bottleneck.
- Verify the sample by hand *and* one degenerate case before submitting.
- Initialise accumulator extremes correctly: `LLONG_MIN` for maxima, not `0`.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Nested loops at $n = 2 \times 10^5$ | TLE | Sort / map / two pointers |
| 2 | Sorting when indices are required | Positions lost | Store `(value, index)` |
| 3 | `unordered_map` with attacker-chosen keys | $\mathcal{O}(n^2)$ worst case | `map`, or a custom hash |
| 4 | `-3 % 2 == -1` in C++ | Wrong remainder key | `((x % K) + K) % K` |
| 5 | Kadane initialised to 0 | All-negative array returns 0 | `LLONG_MIN` or `a[0]` |
| 6 | Boyer–Moore without a verification pass | No majority guaranteed | Count and check |
| 7 | Two-sum inserting before searching | Index matches itself | Search, then insert |
| 8 | Difference-$K$ counting with $K=0$ | Only adjacent duplicates counted | Count runs combinatorially |
| 9 | Spiral traversal without guards | Single row/column printed twice | `if (top <= bottom)` |
| 10 | `k %= n` omitted in rotation | Out-of-bounds when $k > n$ | Reduce $k$ first |
| 11 | Triangle counting with `>=` | Degenerate triples counted | Strict `>` |
| 12 | Median replaced by mean | Wrong on skewed data | Sort and take `a[n/2]` |
| 13 | `int` for `count * units` | Overflow | `long long` |
| 14 | BFS counting edges as cells | Off by one | Decide the convention |
| 15 | DFS for shortest path | Not shortest | BFS |
| 16 | Recursive DFS depth $10^6$ | Stack overflow | Iterative, or larger stack |
| 17 | Assuming the graph is connected | Components missed | Loop over all vertices |
| 18 | Forgetting to normalise rectangle corners | Negative area | `swap` on input |
| 19 | `double` accumulators for integer answers | Precision drift | Stay in `long long` |
| 20 | Trusting the sample only | Samples never contain degeneracies | Test the edge case by hand |

---

## Practice Problems

The twenty problems in this topic are the practice set. To get more from them:

1. **Re-solve each from the statement alone** a week later, without looking at the code.
2. **Write down the constraint-to-technique deduction** before reading the "why" section — if you reach the same conclusion in under 30 seconds, the pattern is learned.
3. **Break each solution deliberately**: set $K=0$, use all negatives, use a single row. Finding the bug yourself is worth more than reading about it.

For a larger pool, see [Topic 11 — Level 1 Unsolved Problem Set](Topic%2011%20-%20Level%201%20Unsolved%20Problem%20Set.md).

---

## Next Up

→ [Topic 11 — Level 1 Unsolved Problem Set](Topic%2011%20-%20Level%201%20Unsolved%20Problem%20Set.md)

Fifty problems, no solutions, constraints given. That is the point: everything you have practised is now tested without a safety net, and the habit that matters is classifying the problem before coding it.
