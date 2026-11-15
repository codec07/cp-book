# Topic 08 - Level 2 Solved Problem Set

Twenty problems, one from every corner of this chapter.

Nothing in this set tells you which technique to use. Each statement is written the way a contest writes it: constraints, a story, and no hint. The work is the same translation step as in Topic 04 - read the constraint, find the phrase, choose the tool - but now the toolbox includes graphs, trees and bits as well as arrays.

Every problem below has been checked: the code compiles, and the shown output is the output of that code on the shown input.

> **Prerequisites:** Topics [01](Topic%2001%20-%20Prefix%20Sums%20%28From%20Basics%20to%20Advanced%20Level%29.md) through [07](Topic%2007%20-%20Bitwise%20Operators.md) of this chapter, plus [Chapter 2, Topic 05 - Introduction to Sorting, Sets and Maps](../CP%20Level%201/Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Choose a technique from the constraints before writing a line of code.
> - Recognise the eight recurring problem shapes disguised in new stories.
> - Write each shape's code from memory.
> - Spot the boundary condition that each shape fails on.

---

## 1. How to read this set

Each problem has the same five parts:

- **Statement** - with constraints, exactly as a judge would give them.
- **Why this approach** - the deduction. This is the part worth reading twice.
- **Solution** - the code.
- **Input / Output** - a small case that exercises the interesting part.
- **Complexity** and **The trap**.

Work them in order the first time. They are grouped by family, and each family's second problem is a variation that exposes what the first one glossed over.

---

## 2. The map

| # | Problem | Family | The phrase that gives it away |
|---|---|---|---|
| 1 | Split Points | prefix sums | "left sum equals right sum" |
| 2 | Range Add, Point Query | difference array | "$q$ range updates, print at the end" |
| 3 | Best Window of $k$ | sliding window | "consecutive", fixed length |
| 4 | Most Water | two pointers | "two lines", area between them |
| 5 | Sort Colours | three pointers | "in place, three values" |
| 6 | Merge Two Arrays | two ends | "sorted, merge without extra space" |
| 7 | Bag Splitting | binary search on answer | "minimise the maximum", $10^9$ values |
| 8 | The Duplicate | binary search on value | $n+1$ values in $[1,n]$ |
| 9 | Sorted Matrix Search | staircase | rows and columns sorted |
| 10 | Platform Count | sweep after sorting | "maximum overlap at any time" |
| 11 | Cookie Assignment | greedy two pointers | "maximise the number satisfied" |
| 12 | Queue by Height | sort then insert | two keys, one defines position |
| 13 | Truck Loading | greedy after sorting | capacity plus per-unit value |
| 14 | Diagonal Grid Path | BFS, 8 directions | "minimum moves", diagonals allowed |
| 15 | Closed Islands | flood fill from the border | "not touching the boundary" |
| 16 | Nearest Zero | multi-source BFS | "distance to the nearest" |
| 17 | K-th Ancestor | binary lifting | "$q$ queries, jump $k$ levels up" |
| 18 | Word Products | bitmask of letters | "share no common letter" |
| 19 | Divisible Subarrays | prefix sums mod $K$ | "sum divisible by" |
| 20 | Products Except Self | prefix and suffix | "without division" |

**The right way to use this table:** cover the right-hand columns and read only the middle one. If you can name the technique from the family alone, you know the chapter.

---

## 3. Problem 1 - Split Points

> **Statement.** Count the indices $i$ for which the sum of the elements strictly left of $i$ equals the sum of the elements strictly right of it. $n \le 2 \times 10^5$, values $\le 10^9$.

### Why this approach

Every index asks the same question about two ranges. Precompute the total; then walking left to right gives the left sum for free, and the right sum is `total - left - a[i]`. One pass, no second array.

This is prefix sums used without materialising a prefix array - the running variable is enough because the queries arrive in order.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n);
    long long total = 0;
    for (auto &x : a) { cin >> x; total += x; }

    long long left = 0, count = 0;

    for (int i = 0; i < n; i++) {
        long long right = total - left - a[i];
        if (left == right) count++;
        left += a[i];
    }

    cout << count << '\n';
    return 0;
}
```

**Input**
```text
5
1 3 5 2 2
```

**Output**
```text
1
```

Only index 2 works: the left sum is $1 + 3 = 4$ and the right sum is $2 + 2 = 4$.

**Complexity.** $O(n)$ time, $O(1)$ extra space.

### The trap

`right = total - left` instead of `total - left - a[i]`. The element at $i$ belongs to neither side; including it makes the equality almost never hold, and the sample may still pass if the array happens to have zeros.

---

## 4. Problem 2 - Range Add, Point Query

> **Statement.** An array of $n$ zeros receives $q$ updates, each adding $x$ to every position in $[l, r]$. After all updates print the array. $n, q \le 2 \times 10^5$, values fit in `long long`.

### Why this approach

Applying each update element by element is $O(nq) = 4 \times 10^{10}$. The difference array turns each update into two writes - `d[l] += x` and `d[r+1] -= x` - and one prefix sum at the end rebuilds the result. This is the mirror image of prefix sums from Topic 01, and it is the reason range updates cost $O(1)$ each.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, q;
    cin >> n >> q;

    vector<long long> diff(n + 2, 0);

    while (q--) {
        int l, r;
        long long x;
        cin >> l >> r >> x;
        diff[l] += x;
        diff[r + 1] -= x;
    }

    long long cur = 0;
    for (int i = 1; i <= n; i++) {
        cur += diff[i];
        cout << cur << " \n"[i == n];
    }

    return 0;
}
```

**Input**
```text
5 3
1 3 2
2 5 1
2 2 4
```

**Output**
```text
2 7 3 1 1
```

Position 2 receives all three updates, $2 + 1 + 4 = 7$. Position 3 receives the first two, $2 + 1 = 3$. Position 4 receives only the second, and position 5 the same.

**Complexity.** $O(n + q)$.

### The trap

The array must have size `n + 2`. The write `diff[r + 1] -= x` happens at index `n + 1` when $r = n$, and a vector of size `n + 1` writes out of bounds - which often survives because the next vector starts immediately after, producing a wrong answer only on some inputs.

---

## 5. Problem 3 - Best Window of k

> **Statement.** Find the maximum sum of `k` consecutive elements. $n \le 2 \times 10^5$, $k \le n$.

### Why this approach

A fixed-length window slides: subtract the element leaving, add the element entering. Each step is $O(1)$ instead of $O(k)$, so the whole scan is $O(n)$ rather than $O(nk)$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    cin >> n >> k;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    long long sum = 0;
    for (int i = 0; i < k; i++) sum += a[i];

    long long best = sum;

    for (int i = k; i < n; i++) {
        sum += a[i] - a[i - k];      // in on the right, out on the left
        best = max(best, sum);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
6 3
2 1 5 1 3 2
```

**Output**
```text
9
```

The windows are $8, 7, 9, 6$; the third is largest.

**Complexity.** $O(n)$.

### The trap

Initialising `best = 0`. With an all-negative array the true maximum is negative and every window is rejected - the answer comes out 0, which is not even a valid window sum. Initialise `best` from the first window.

---

## 6. Problem 4 - Most Water

> **Statement.** Two vertical lines at positions $i < j$ with heights $h_i$ and $h_j$ hold water of area $\min(h_i, h_j) \cdot (j - i)$. Find the maximum over all pairs. $n \le 2 \times 10^5$, heights $\le 10^9$.

### Why this approach

There are $n^2$ pairs and the area has two factors pulling in opposite directions. Start with the widest pair - both ends - and move the **shorter** line inward. Moving the taller one cannot help: the width shrinks and the height is still capped by the shorter line, so that pair cannot beat the current one, and by induction every skipped pair is dominated.

### Solution

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
        long long area = min(h[i], h[j]) * (long long)(j - i);
        best = max(best, area);

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

The best pair is at positions 1 and 8, with heights 8 and 7 and width 7: $7 \times 7 = 49$.

**Complexity.** $O(n)$.

### The trap

`min(h[i], h[j]) * (j - i)` computed in `int`. Heights up to $10^9$ times a width up to $2 \times 10^5$ is $2 \times 10^{14}$; the cast to `long long` must be on the product, not added afterwards.

---

## 7. Problem 5 - Sort Colours

> **Statement.** An array contains only 0, 1 and 2. Sort it in one pass using $O(1)$ extra space.

### Why this approach

Three regions: 0s at the front, 1s in the middle, 2s at the back. Maintain pointers `lo` (end of the 0s), `mid` (current) and `hi` (start of the 2s). Each element is inspected once and swapped into place.

This is the Dutch national flag partition, and it is the template for any three-way partition.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> a(n);
    for (auto &x : a) cin >> x;

    int lo = 0, mid = 0, hi = n - 1;

    while (mid <= hi) {
        if (a[mid] == 0)      swap(a[lo++], a[mid++]);
        else if (a[mid] == 1) mid++;
        else                  swap(a[mid], a[hi--]);
    }

    for (int i = 0; i < n; i++) cout << a[i] << " \n"[i == n - 1];
    return 0;
}
```

**Input**
```text
6
2 0 2 1 1 0
```

**Output**
```text
0 0 1 1 2 2
```

**Complexity.** $O(n)$ time, $O(1)$ space.

### The trap

Incrementing `mid` after swapping with `hi`. The element swapped in from the back has not been inspected yet, so `mid` must stay put. Incrementing it skips an element, and the array comes out unsorted in a way that depends on the input order - usually passing the sample and failing a random test.

---

## 8. Problem 6 - Merge Two Sorted Arrays

> **Statement.** Given sorted arrays $A$ (size $p$) and $B$ (size $q$), merge them into $A$ without extra space. $p + q \le 2 \times 10^5$.

### Why this approach

Merging from the front overwrites elements that have not been read yet. Merging **from the back** fills the array from its largest slot, and the largest remaining element is always at the end of $A$'s unmerged part or of $B$. Nothing unread is ever overwritten.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int p, q;
    cin >> p >> q;

    vector<long long> A(p + q), B(q);
    for (int i = 0; i < p; i++) cin >> A[i];
    for (auto &x : B) cin >> x;

    int i = p - 1, j = q - 1, k = p + q - 1;

    while (j >= 0) {
        if (i >= 0 && A[i] > B[j]) A[k--] = A[i--];
        else                       A[k--] = B[j--];
    }

    for (int t = 0; t < p + q; t++) cout << A[t] << " \n"[t == p + q - 1];
    return 0;
}
```

**Input**
```text
3 3
1 2 3
2 5 6
```

**Output**
```text
1 2 2 3 5 6
```

**Complexity.** $O(p + q)$.

### The trap

The loop condition is `j >= 0`, not `i >= 0 && j >= 0`. When $B$ is exhausted the remaining prefix of $A$ is already in place; looping further would only copy elements onto themselves. But if the loop stops while `j >= 0` - which `i >= 0 && j >= 0` does when $A$ runs out first - elements of $B$ are never copied, and they are the smallest ones.

---

## 9. Problem 7 - Bag Splitting

> **Statement.** There are $n$ bags with `a[i]` balls. You may perform at most $h$ splits; each split replaces a bag with two bags holding positive amounts. Minimise the maximum bag size. $n \le 2 \times 10^5$, $a_i \le 10^9$.

### Why this approach

"Minimise the maximum" with a monotone feasibility test - a bag of size $x$ needs $\lceil x / L \rceil - 1$ splits to reach limit $L$. If limit $L$ works, every larger limit works, so binary search the answer and count splits.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long h;
    cin >> n >> h;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    auto ok = [&](long long limit) {
        long long ops = 0;
        for (long long x : a) {
            ops += (x + limit - 1) / limit - 1;    // bags needed, minus one
            if (ops > h) return false;
        }
        return true;
    };

    long long lo = 1, hi = *max_element(a.begin(), a.end());

    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid;
        else         lo = mid + 1;
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
4 4
2 4 8 2
```

**Output**
```text
2
```

At limit 2 the bags need $1, 2, 4, 1$ pieces - four splits in total, which is exactly the budget. Limit 1 would need twelve.

**Complexity.** $O(n \log(\max a))$.

### The trap

The ceiling division. `x / limit` rounds down and undercounts pieces; `(x + limit - 1) / limit` is the correct form. With `x` up to $10^9$ and `limit` starting at 1, `x + limit - 1` stays inside `long long`.

---

## 10. Problem 8 - The Duplicate

> **Statement.** An array of length $n + 1$ contains integers in $[1, n]$ with exactly one value repeated. Find it without modifying the array and with $O(1)$ extra space. $n \le 10^5$.

### Why this approach

Binary search on the **value**, not the index. If fewer than or equal to `mid` values are counted, the duplicate is above `mid`; if more, it is at or below. The count is monotone in `mid`, which is all binary search needs.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> a(n + 1);
    for (auto &x : a) cin >> x;

    int lo = 1, hi = n;

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;

        int count = 0;
        for (int x : a)
            if (x <= mid) count++;

        if (count > mid) hi = mid;      // too many small values
        else             lo = mid + 1;
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
4
1 3 4 2 2
```

**Output**
```text
2
```

At `mid = 2` there are three values $\le 2$ (the two 2s and the 1) in a range that should hold two, so the duplicate is at or below 2. At `mid = 1` there is exactly one, so the answer is 2.

**Complexity.** $O(n \log n)$ time, $O(1)$ space.

### The trap

Indexing the search from 0. The values run $1 \ldots n$, and `mid = 0` makes the predicate meaningless - every element is greater than 0, so the count is always `n + 1` and the search commits to an answer immediately.

The second trap is the comparison direction: `count > mid` means *too many numbers in a range that should have few*, which is the signature of the duplicate being small.

---

## 11. Problem 9 - Sorted Matrix Search

> **Statement.** An $r \times c$ matrix is sorted left-to-right in every row and top-to-bottom in every column. Decide whether a target is present in $O(r + c)$. $r, c \le 1000$.

### Why this approach

Start at the **top-right corner**. That cell is the largest in its row and the smallest in its column, so one comparison eliminates a whole row or a whole column: too big means the target is not in this column (move left), too small means it is not in this row (move down). After $r + c$ steps either the target is found or the walk leaves the matrix.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int r, c;
    cin >> r >> c;

    vector<vector<long long>> m(r, vector<long long>(c));
    for (auto &row : m)
        for (auto &x : row) cin >> x;

    long long target;
    cin >> target;

    int i = 0, j = c - 1;
    bool found = false;

    while (i < r && j >= 0) {
        if (m[i][j] == target) { found = true; break; }

        if (m[i][j] > target) j--;      // column j is too large
        else                  i++;      // row i is too small
    }

    cout << (found ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
4 4
1 4 7 11
2 5 8 12
3 6 9 16
10 13 14 17
5
```

**Output**
```text
YES
```

From 11 the walk goes left to 7, down to 8, left to 5 and stops.

**Complexity.** $O(r + c)$ time, $O(rc)$ to store the matrix.

### The trap

Starting at the top-left. From there both neighbours are larger, so a comparison tells you nothing about which way to go - the staircase property needs one neighbour smaller and one larger, which only the top-right (or bottom-left) corner provides.

---

## 12. Problem 10 - Platform Count

> **Statement.** Trains arrive and depart at given times. Find the minimum number of platforms so that no train waits. $n \le 2 \times 10^5$.

### Why this approach

This is a maximum-overlap problem. Sorting arrivals and departures separately and sweeping gives the number of trains present at every moment without any interval tree. Each event is a $+1$ or a $-1$, and the answer is the largest running total.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> arrive(n), depart(n);
    for (auto &x : arrive) cin >> x;
    for (auto &x : depart) cin >> x;

    sort(arrive.begin(), arrive.end());
    sort(depart.begin(), depart.end());

    int i = 0, j = 0, current = 0, best = 0;

    while (i < n) {
        if (arrive[i] <= depart[j]) {    // a train arrives first
            current++;
            best = max(best, current);
            i++;
        } else {                         // a train leaves first
            current--;
            j++;
        }
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
6
900 940 950 1100 1500 1800
910 1200 1120 1130 1900 2000
```

**Output**
```text
3
```

At time 1100 the trains that arrived at 940, 950 and 1100 are all still present, so three platforms are needed.

**Complexity.** $O(n \log n)$ for the sorts, $O(n)$ for the sweep.

### The trap

`arrive[i] <= depart[j]` versus `<`. A platform freed at the same instant another train arrives can be reused, so arrivals must be processed first when the times tie. With `<` the sweep adds a platform that is never needed, and the answer is too large by one on every tied event.

---

## 13. Problem 11 - Cookie Assignment

> **Statement.** Each child $i$ needs a cookie of size at least `g[i]`. Each cookie $j$ has size `s[j]`. Each child gets at most one cookie. Maximise the number of satisfied children. $n, m \le 2 \times 10^5$.

### Why this approach

Sort both. The smallest cookie that satisfies the least demanding child should go to that child - giving it a bigger cookie wastes capacity, and giving it a smaller one is impossible. Two pointers then move monotonically.

The exchange argument is short: in any optimal assignment, swapping the cookie of a satisfied child for a smaller one that still satisfies it never hurts any other child.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;

    vector<long long> g(n), s(m);
    for (auto &x : g) cin >> x;
    for (auto &x : s) cin >> x;

    sort(g.begin(), g.end());
    sort(s.begin(), s.end());

    int i = 0, j = 0, satisfied = 0;

    while (i < n && j < m) {
        if (s[j] >= g[i]) {      // this cookie satisfies this child
            satisfied++;
            i++;
            j++;
        } else {
            j++;                 // too small; try the next cookie
        }
    }

    cout << satisfied << '\n';
    return 0;
}
```

**Input**
```text
2 3
1 2
1 2 3
```

**Output**
```text
2
```

Cookie 1 satisfies the child needing 1, cookie 2 satisfies the child needing 2, and cookie 3 is spare.

**Complexity.** $O(n \log n + m \log m)$.

### The trap

Advancing `i` when the cookie is too small. That marks the child as satisfied without a cookie - a silent +1 on the answer that looks right whenever the greedy happens to match. Only advancing `j` on failure keeps the count honest.

---

## 14. Problem 12 - Queue by Height

> **Statement.** $n$ people are described by $(h, k)$: height $h$, and exactly $k$ people in front of them have height $\ge h$. Reconstruct the original queue. $n \le 2000$.

### Why this approach

Sort by height **descending**, ties by $k$ ascending, then insert each person at index $k$. At the moment of insertion everyone already placed is at least as tall, so index $k$ is exactly the number of qualifying people in front. Shorter people inserted later do not affect the count, which is why the order works.

This is a comparator problem where the comparator produces an **insertion order**, not the final order - the sort is a preprocessing step for the `insert`.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<int,int>> p(n);          // (height, k)
    for (auto &q : p) cin >> q.first >> q.second;

    sort(p.begin(), p.end(), [](const auto &a, const auto &b) {
        if (a.first != b.first) return a.first > b.first;   // taller first
        return a.second < b.second;                          // smaller k first
    });

    vector<pair<int,int>> line;
    for (auto &q : p)
        line.insert(line.begin() + q.second, q);

    for (auto &q : line) cout << q.first << ' ' << q.second << '\n';
    return 0;
}
```

**Input**
```text
6
7 0
4 4
7 1
5 0
6 1
5 2
```

**Output**
```text
5 0
7 0
5 2
6 1
4 4
7 1
```

**Complexity.** $O(n^2)$ - the `insert` shifts elements. For $n \le 2000$ that is $4 \times 10^6$ operations; at $n = 2 \times 10^5$ it needs a Fenwick tree for empty slots.

### The trap

Inserting at `k` **after** sorting ascending by height. Then the people already placed are shorter, the count means nothing, and the output looks arbitrary. The direction of the height sort is the whole argument.

The second trap is ties on height: they must be ordered by $k$ ascending, or two equal-height people can be inserted in an order that violates both counts.

---

## 15. Problem 13 - Truck Loading

> **Statement.** A truck has capacity `size` boxes. There are $n$ box types; type $i$ has `count[i]` boxes and each holds `units[i]` items. Maximise the number of items loaded. $n \le 10^5$.

### Why this approach

Every box consumes exactly one unit of capacity, so the value per unit of capacity is `units[i]` - no ratio to compute, no fractional knapsack. Sort by units descending and take greedily until the truck is full.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long truckSize;
    cin >> n >> truckSize;

    vector<pair<long long,long long>> box(n);    // (units, count)
    for (auto &p : box) cin >> p.second >> p.first;

    sort(box.rbegin(), box.rend());              // units descending

    long long total = 0;

    for (auto &p : box) {
        long long take = min(truckSize, p.second);
        total += take * p.first;
        truckSize -= take;
        if (truckSize == 0) break;
    }

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
3 4
1 3
2 2
3 1
```

**Output**
```text
8
```

Take 1 box of the 3-unit type, 2 of the 2-unit type, and 1 of the 1-unit type: $3 + 4 + 1 = 8$ with capacity exactly 4.

**Complexity.** $O(n \log n)$.

### The trap

Reading the input as `units` then `count`. The statement's natural order is "count, units", and swapping them makes the greedy take `units` copies of a box that holds `count` items. The sample often still runs, just with the wrong answer.

The second trap is `int` for `total`: $10^5$ boxes times $10^3$ units times $10^3$ count can reach $10^{11}$.

---

## 16. Problem 14 - Diagonal Grid Path

> **Statement.** A grid contains open cells `.` and walls `#`. Find the minimum number of moves from the top-left to the bottom-right, moving in any of the **eight** directions, or `-1` if unreachable. $r, c \le 1000$.

### Why this approach

Unweighted grid, minimum moves: BFS. The only difference from the four-direction version is the direction arrays. Diagonal moves cost 1, not $\sqrt 2$ - the problem counts moves, not distance, which is why BFS applies.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int r, c;
    cin >> r >> c;

    vector<string> grid(r);
    for (auto &row : grid) cin >> row;

    if (grid[0][0] == '#') { cout << -1 << '\n'; return 0; }

    vector<vector<int>> dist(r, vector<int>(c, -1));
    queue<pair<int,int>> q;

    dist[0][0] = 0;
    q.push({0, 0});

    int dr[8] = {-1, -1, -1, 0, 0, 1, 1, 1};
    int dc[8] = {-1, 0, 1, -1, 1, -1, 0, 1};

    while (!q.empty()) {
        int cr = q.front().first;
        int cc = q.front().second;
        q.pop();

        for (int d = 0; d < 8; d++) {
            int nr = cr + dr[d];
            int nc = cc + dc[d];

            if (nr < 0 || nr >= r || nc < 0 || nc >= c) continue;
            if (grid[nr][nc] == '#' || dist[nr][nc] != -1) continue;

            dist[nr][nc] = dist[cr][cc] + 1;
            q.push({nr, nc});
        }
    }

    cout << dist[r - 1][c - 1] << '\n';
    return 0;
}
```

**Input**
```text
3 3
...
...
...
```

**Output**
```text
2
```

Two diagonal moves reach the opposite corner.

**Complexity.** $O(rc)$.

### The trap

Checking the destination cell only at the end. If the bottom-right cell is a wall it is never enqueued, `dist` stays `-1`, and the code prints `-1` - which is correct by luck. But if the **start** is a wall, the BFS floods the grid anyway and may reach the destination, printing a path that begins inside a wall. The early return on `grid[0][0] == '#'` is not optional.

The second trap is mixing up `dr` and `dc` - all eight offsets must be consistent, and a transposed pair still produces plausible answers on square grids.

---

## 17. Problem 15 - Closed Islands

> **Statement.** A grid contains land (`0`) and water (`1`). Count the land regions that touch no border. Land connects through sides. $r, c \le 500$.

### Why this approach

An open region is exactly one that reaches the border. So invert the question: flood fill from every border land cell, marking those regions as water, and whatever land remains is closed. This turns "is it closed" into "is it connected to the outside", which is a single traversal.

The alternative - testing each region against the border after collecting it - needs a "touched border" flag threaded through the recursion and is easy to get wrong.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int r, c;
vector<string> grid;

void flood(int i, int j) {
    if (i < 0 || i >= r || j < 0 || j >= c) return;
    if (grid[i][j] != '0') return;
    grid[i][j] = '1';

    flood(i + 1, j);
    flood(i - 1, j);
    flood(i, j + 1);
    flood(i, j - 1);
}

int main() {
    cin >> r >> c;

    grid.resize(r);
    for (auto &row : grid) cin >> row;

    for (int i = 0; i < r; i++) {          // left and right borders
        if (grid[i][0] == '0') flood(i, 0);
        if (grid[i][c - 1] == '0') flood(i, c - 1);
    }
    for (int j = 0; j < c; j++) {          // top and bottom borders
        if (grid[0][j] == '0') flood(0, j);
        if (grid[r - 1][j] == '0') flood(r - 1, j);
    }

    int closed = 0;

    for (int i = 0; i < r; i++)
        for (int j = 0; j < c; j++)
            if (grid[i][j] == '0') {
                closed++;
                flood(i, j);
            }

    cout << closed << '\n';
    return 0;
}
```

**Input**
```text
5 5
11111
10011
11111
10011
11111
```

**Output**
```text
2
```

The two $1 \times 2$ land strips are both surrounded by water and neither touches the border.

**Complexity.** $O(rc)$ time; the recursion depth is $O(rc)$ in the worst case.

### The trap

Scanning the four border loops as `i < r` for rows and `c < c` for columns - a typo that is invisible until the grid is not square. Keep the two loops distinct and check the indices separately: `grid[i][0]`, `grid[i][c-1]`, `grid[0][j]`, `grid[r-1][j]`.

The second trap is forgetting that the border scan must **also** handle a grid of one row or one column, where the two loops visit the same cells. Flooding an already-converted cell is harmless because the first check rejects it.

---

## 18. Problem 16 - Nearest Zero

> **Statement.** For every cell of a binary matrix, print the distance (in side-adjacent steps) to the nearest `0`. $r, c \le 1000$.

### Why this approach

One BFS per zero is $O(\#\text{zeros} \cdot rc)$. Instead, push **all** zeros into the queue at distance 0 and run one BFS - the multi-source trick from Topic 05. Each cell is reached first by its nearest zero, because BFS explores in rings around all sources simultaneously.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int r, c;
    cin >> r >> c;

    vector<vector<int>> a(r, vector<int>(c));
    vector<vector<int>> dist(r, vector<int>(c, -1));
    queue<pair<int,int>> q;

    for (int i = 0; i < r; i++)
        for (int j = 0; j < c; j++) {
            cin >> a[i][j];
            if (a[i][j] == 0) {
                dist[i][j] = 0;
                q.push({i, j});
            }
        }

    int dr[4] = {-1, 1, 0, 0};
    int dc[4] = {0, 0, -1, 1};

    while (!q.empty()) {
        int cr = q.front().first;
        int cc = q.front().second;
        q.pop();

        for (int d = 0; d < 4; d++) {
            int nr = cr + dr[d];
            int nc = cc + dc[d];

            if (nr < 0 || nr >= r || nc < 0 || nc >= c) continue;
            if (dist[nr][nc] != -1) continue;

            dist[nr][nc] = dist[cr][cc] + 1;
            q.push({nr, nc});
        }
    }

    for (int i = 0; i < r; i++)
        for (int j = 0; j < c; j++)
            cout << dist[i][j] << " \n"[j == c - 1];

    return 0;
}
```

**Input**
```text
3 3
0 0 0
0 1 0
1 1 1
```

**Output**
```text
0 0 0
0 1 0
1 2 1
```

The bottom-middle cell is two steps from the nearest zero by either side.

**Complexity.** $O(rc)$ - every cell is enqueued at most once.

### The trap

Running a BFS from the first zero and then patching the rest. The distances from one zero are simply wrong elsewhere, and the patch is another $O(rc)$ per zero.

The second trap is stamping `dist` with a large number and minimising instead of using `-1` as "unvisited". Minimising requires a relaxation step and can enqueue a cell many times; the `-1` check enqueues exactly once.

---

## 19. Problem 17 - K-th Ancestor

> **Statement.** A tree is rooted at 1. Answer $q$ queries: given $v$ and $k$, print the $k$-th ancestor of $v$, or `0` if it does not exist. $n, q \le 2 \times 10^5$.

### Why this approach

The naive climb is $O(k)$ per query - $O(n)$ worst case. Binary lifting stores `up[b][v]`, the $2^b$-th ancestor, so any $k$ is a sum of powers of two and the query is $O(\log n)$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, LOGN;
vector<vector<int>> adj, up;
vector<int> depth;

int main() {
    int q;
    cin >> n >> q;

    adj.assign(n + 1, {});
    for (int i = 0; i < n - 1; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    depth.assign(n + 1, 0);
    vector<int> parent(n + 1, 0), st = {1};
    parent[1] = 0;

    while (!st.empty()) {
        int u = st.back();
        st.pop_back();

        for (int v : adj[u]) {
            if (v == parent[u]) continue;
            parent[v] = u;
            depth[v] = depth[u] + 1;
            st.push_back(v);
        }
    }

    LOGN = 1;
    while ((1 << LOGN) <= n) LOGN++;

    up.assign(LOGN, vector<int>(n + 1, 0));
    up[0] = parent;

    for (int b = 1; b < LOGN; b++)
        for (int v = 1; v <= n; v++)
            up[b][v] = up[b - 1][up[b - 1][v]];

    while (q--) {
        int v;
        long long k;
        cin >> v >> k;

        if (k > depth[v]) { cout << 0 << '\n'; continue; }

        long long step = k;
        for (int b = 0; b < LOGN; b++)
            if ((step >> b) & 1) v = up[b][v];

        cout << v << '\n';
    }

    return 0;
}
```

**Input**
```text
5 5
1 2
1 3
2 4
2 5
4 1
4 2
5 1
3 0
1 1
```

**Output**
```text
2
1
2
3
0
```

Vertex 4's 1st ancestor is 2 and its 2nd is 1. Vertex 5's parent is 2. Zero steps stay put. Vertex 1 has no ancestor, so the answer is 0.

**Complexity.** $O(n \log n)$ preprocessing, $O(\log n)$ per query.

### The trap

The `k > depth[v]` test must come **before** the lift, and it must use the original depth. Once the loop starts modifying `v`, the depth no longer describes the current vertex, and the existence check becomes meaningless.

The second trap is `k` as `int`: a query may ask for up to $10^9$ steps, which is legal input even though no ancestor exists. Reading it as `long long` makes the comparison safe.

---

## 20. Problem 18 - Word Products

> **Statement.** Given $n$ words, find $\max |a| \cdot |b|$ over pairs of words that share no common letter. Print 0 if no pair qualifies. $n \le 1000$, $|w| \le 1000$.

### Why this approach

Comparing two words letter by letter is $O(L)$; doing it for all pairs is $O(n^2L)$, which at $10^3 \times 10^3 \times 10^3$ is too slow for comfortable limits. Instead encode each word's letter set as a 26-bit mask, and the disjointness test becomes `(mask[i] & mask[j]) == 0` - one instruction.

This is the "set as integer" use of bits, and it is the single most common bitmask application outside DP.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<string> w(n);
    for (auto &s : w) cin >> s;

    vector<int> mask(n, 0), len(n);

    for (int i = 0; i < n; i++) {
        len[i] = (int)w[i].size();
        for (char c : w[i]) mask[i] |= 1 << (c - 'a');
    }

    int best = 0;

    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if ((mask[i] & mask[j]) == 0)
                best = max(best, len[i] * len[j]);

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
6
abcw
baz
foo
bar
xtfn
abcdef
```

**Output**
```text
16
```

`abcw` and `xtfn` share no letter: $4 \times 4 = 16$.

**Complexity.** $O(n \cdot L + n^2)$.

### The trap

Comparing characters in the pair loop. It is $O(n^2 L)$ and passes small tests, then times out - and the fix is exactly the mask above.

The second trap is `1 << (c - 'a')` with `c` as a signed `char` on systems where `char` is signed and the input is non-ASCII. Input here is lowercase letters, so it is safe; still, casting to `int` before shifting is the habit.

---

## 21. Problem 19 - Divisible Subarrays

> **Statement.** Count the contiguous subarrays whose sum is divisible by $K$. $n \le 2 \times 10^5$, $K \le 10^5$, values may be negative.

### Why this approach

A subarray sum is a difference of prefixes, and divisibility by $K$ depends only on the **remainder**: $P_r \equiv P_{l-1} \pmod K$. So the condition becomes "two prefixes with the same remainder", and the count is a pair count per remainder class.

This is the prefix-sum counting pattern with a modulus, and negatives are the only subtlety.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    map<long long, long long> count;
    count[0] = 1;                    // the empty prefix

    long long prefix = 0, answer = 0;

    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        prefix += x;

        long long r = ((prefix % k) + k) % k;

        answer += count[r];
        count[r]++;
    }

    cout << answer << '\n';
    return 0;
}
```

**Input**
```text
6 5
4 5 0 -2 -3 1
```

**Output**
```text
7
```

The seven qualifying subarrays are $[4,5,0,-2,-3,1]$, $[5]$, $[5,0]$, $[5,0,-2,-3]$, $[0]$, $[0,-2,-3]$ and $[-2,-3]$, with sums $5, 5, 5, 0, 0, -5, -5$. Each one is the difference of two prefixes with the same remainder modulo 5.

**Complexity.** $O(n \log n)$ with `map`; $O(n)$ expected with `unordered_map`.

### The trap

`prefix % k` with negative `prefix`. In C++, `-3 % 5` is `-3`, not `2`, so negative remainders fall into separate classes and the pair counts break. `((prefix % k) + k) % k` normalises to $[0, k-1]$.

The second trap is `count[0] = 1`. Without it every subarray starting at index 0 disappears from the count.

---

## 22. Problem 20 - Products Except Self

> **Statement.** For each index, print the product of all elements except that one, without using division. $n \le 2 \times 10^5$, products fit in `long long`.

### Why this approach

The answer at $i$ is (product of everything left of $i$) times (product of everything right of $i$). Two sweeps: fill `res[i]` with the prefix product, then multiply in the suffix product while walking backwards. $O(n)$ time, and the output array doubles as the prefix array.

Division would fail on zeros, which is exactly why the problem forbids it.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n), res(n);
    for (auto &x : a) cin >> x;

    long long pre = 1;
    for (int i = 0; i < n; i++) {
        res[i] = pre;                // product of everything to the left
        pre *= a[i];
    }

    long long suf = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= suf;               // times everything to the right
        suf *= a[i];
    }

    for (int i = 0; i < n; i++) cout << res[i] << " \n"[i == n - 1];
    return 0;
}
```

**Input**
```text
4
1 2 3 4
```

**Output**
```text
24 12 8 6
```

**Complexity.** $O(n)$ time, $O(n)$ output space (the `a` array is not modified).

### The trap

Multiplying in the suffix **before** using `res[i]`. The backward loop must read the prefix value, multiply by the suffix, and only then extend the suffix with `a[i]`. Swapping the last two lines multiplies each answer by its own element.

The second trap is zero. With one zero, every position except the zero's own gets 0 and the zero's position gets the product of the others - which the two-sweep code handles correctly, and division-based code does not.

---

## 23. What the twenty cover

| Family | Problems | Count |
|---|---|---|
| Prefix sums and difference arrays | 1, 2, 19, 20 | 4 |
| Two pointers and sliding windows | 3, 4, 5, 6 | 4 |
| Binary search (answer or value) | 7, 8 | 2 |
| Sorting and greedy | 10, 11, 12, 13 | 4 |
| Staircase search | 9 | 1 |
| Graph traversal | 14, 15, 16 | 3 |
| Trees and binary lifting | 17 | 1 |
| Bits | 18 | 1 |

Three observations from the table.

**Four of the twenty are prefix sums in some form**, and two of those (19, 20) do not look like prefix sums at all. That ratio is not an accident: the running product and the running remainder are the same idea wearing different clothes.

**Only two are binary search**, but both are answer-search rather than lookup. In contest sets at this level, lookup binary search is a tool inside a larger solution, not a problem on its own.

**Three are graph traversals, and none of them gives you a graph.** A grid is the graph. Recognising that is the entire difficulty.

---

## Key Takeaways

- Read the constraint first: it selects the technique more often than the story does.
- "Left equals right" is a running sum, not two arrays.
- $q$ range updates plus one final print is a difference array.
- Fixed-length windows slide in $O(1)$ per step; variable windows need monotonicity.
- Two pointers from both ends require knowing which end to move and why.
- Three-way partition uses `lo`, `mid`, `hi`, and never advances `mid` after a swap from the right.
- Merging from the back is the only way to merge in place.
- "Minimise the maximum" with a monotone check is binary search on the answer.
- Binary search on the **value** works when the count of values below a threshold is monotone.
- In a matrix sorted both ways, start at a corner where one neighbour is smaller and one larger.
- Maximum overlap is a sorted sweep of $+1$ and $-1$ events.
- Greedy needs an exchange argument, and the argument usually decides the sort key.
- Insert-at-index problems sort by the key that fixes the final order, then insert.
- Value per unit of capacity is the greedy key when every item costs the same capacity.
- Unweighted minimum moves means BFS, and eight directions is only a bigger direction array.
- "Closed" is "not connected to the border" - flood from the border and invert.
- Multi-source BFS initialises the queue with every source at distance 0.
- Binary lifting answers any jump in $O(\log n)$; the existence check comes before the lift.
- A letter set is a 26-bit mask, and disjointness is one `&`.
- Prefix sums modulo $K$ count divisible subarrays; normalise negative remainders.
- Prefix and suffix products solve "except self" without division, and survive zeros.

---

## Common Mistakes & Pitfalls

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Right sum includes `a[i]` | Equality almost never holds | Subtract the element |
| 2 | Difference array of size `n + 1` | Out-of-bounds at `r = n` | Size `n + 2` |
| 3 | `best = 0` for a window sum | All-negative arrays answer 0 | Initialise from the first window |
| 4 | `int` area in the water problem | Overflow at $2 \times 10^{14}$ | `long long` |
| 5 | `mid++` after swapping with `hi` | One element skipped | Leave `mid` in place |
| 6 | Merging from the front | Overwrites unread data | Merge from the back |
| 7 | `x / limit` in a ceiling count | Undercounts pieces | `(x + limit - 1) / limit` |
| 8 | Binary search starting at 0 | Predicate meaningless | Search $1 \ldots n$ |
| 9 | Starting the matrix walk at the top-left | No direction is eliminated | Use the top-right corner |
| 10 | `<` for arrivals on a tie | Extra platform | `<=` |
| 11 | Advancing the child on a small cookie | Inflated answer | Advance only the cookie |
| 12 | Sorting heights ascending before insert | Arbitrary queue | Descending, ties by $k$ |
| 13 | Swapping count and units in the input | Wrong greedy | Match the statement order |
| 14 | Forgetting the start-is-wall check | Path from inside a wall | Early return |
| 15 | Border loops with `c < c` | Columns only for square grids | Distinct index checks |
| 16 | One BFS per source | Time limit exceeded | Multi-source queue |
| 17 | Existence check after lifting | Depths no longer valid | Check `k > depth[v]` first |
| 18 | Character-by-character word comparison | $O(n^2L)$ timeout | Bitmask |
| 19 | Negative `%` in C++ | Remainder classes split | `((p % k) + k) % k` |
| 20 | Multiplying suffix before using the prefix | Each answer inflated by its element | Order the two lines |

---

## Practice Problems

Topic 09 is the unsolved set: fifty problems, all professional style with constraints, spanning the same eight families. Work them in the order given there.

Two habits to carry over from this set:

- Before coding, write one sentence naming the technique and one sentence justifying it. If the justification is "it looks right", you have not found the argument yet.
- After the code passes the sample, re-read the trap paragraph for that family. In this chapter, most wrong answers are boundary conditions, not algorithm choices.

---

## Next Up

→ [Topic 09 - Level 2 Unsolved Problem Set](Topic%2009%20-%20Level%202%20Unsolved%20Problem%20Set.md)

Fifty problems, grouped by family, with constraints and a graded hint for each. That is the last topic of Level 2, and the point where the chapter stops teaching and starts testing.

