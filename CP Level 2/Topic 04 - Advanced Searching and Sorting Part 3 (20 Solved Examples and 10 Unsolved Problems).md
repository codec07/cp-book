# Topic 04 — Advanced Searching and Sorting Part 3

Twenty problems. Every one is solvable with the techniques from Topics 01–03, and none of them says which technique to use.

That is the point of this file. In Topics 01–03 the tool was named in the section heading. Here the statement is all you get, and the work is the translation step: *this phrasing means two pointers*, *this constraint means binary search on the answer*, *this ordering requirement means a custom comparator*.

> **Prerequisites:** Topics [01](Topic%2001%20-%20Prefix%20Sums%20%28From%20Basics%20to%20Advanced%20Level%29.md), [02](Topic%2002%20-%20Advanced%20Searching%20and%20Sorting%20Part%201%20%28Two%20Pointers%2C%20Binary%20Search%20on%20Sorted%20and%20Unsorted%20Arrays%29.md) and [03](Topic%2003%20-%20Advanced%20Searching%20and%20Sorting%20Part%202%20%28Custom%20Comparators%2C%20Coordinate%20Compression%20and%20Greedy%20Sorting%29.md) of this chapter.
> **You will be able to:**
> - Map a statement to a technique without being told which one applies.
> - Combine two techniques in one solution — the real Level 2 skill.
> - Handle duplicates, ties, and equality boundaries in each pattern.
> - Recognise the ten statements that recur across every contest.

---

## 1. How to use this set

Each problem is presented the way a contest would present it: a **statement** written as prose, with the constraints a judge would publish and no hint about the method. Everything that could give the answer away is folded behind a disclosure triangle:

- **Statement** - always visible. Read it twice and decide what has to be computed before opening anything.
- **Solution** (collapsed) - the deduction from the statement to the technique, the code, a worked **input / output** pair, the complexity, and **the trap**.

The twenty are ordered by technique, not by difficulty. §2 is the index, but it is the answer key, so it is collapsed too: open it only after you have attempted a problem, or at the end, to check whether your instinct was naming the same technique.

A statement that reads oddly is usually doing its job. Contest prose carries decoys - a delivery schedule that does not matter, an office that never appears again, an "at most" that is really an equality. Finding the one sentence that constrains the answer is part of the problem.

---

## 2. The technique map

<details>
<summary>Spoiler index - open only after you have attempted the set</summary>

| # | Problem | Technique | The clue buried in the statement |
|---|---|---|---|
| 1 | Three Sum | sort + converging pointers | "kinds of three-donation groups", $n \le 3000$ |
| 2 | Squares of a Sorted Array | two ends, fill backwards | input arrives sorted, output must be sorted |
| 3 | Partition Labels | last-occurrence table + scan | "a letter in one piece only", "largest number of pieces" |
| 4 | At Most $K$ Distinct | variable sliding window | "at most $K$ different genres", longest streak |
| 5 | Product Less Than $K$ | shrinking window | "product ... strictly below $K$", count runs |
| 6 | Remove Duplicates II | slow/fast write pointer | "at most two of any one number", "no spare storage" |
| 7 | Find Peak Element | binary search, no sort | "higher than both neighbours", logarithmic probes |
| 8 | Rotated Sorted Array | binary search on a broken order | "the stack was cut and swapped", logarithmic page opens |
| 9 | Bouquets | binary search on the answer | "earliest morning", $k$ adjacent beds |
| 10 | K-th Smallest Pair Distance | answer search + two pointers | "$k$-th smallest value" among all pairs, $10^4$ lamps |
| 11 | Median of Two Sorted Arrays | partition binary search | "median", "logarithmic number of entries" |
| 12 | K-th in a Sorted Matrix | value search + staircase count | rows and columns both increase, $k$-th entry |
| 13 | Longest Increasing Subsequence | `tails` + `lower_bound` | "longest subsequence, gaps allowed", $n \le 2 \times 10^5$ |
| 14 | Sort Characters by Frequency | counting + comparator | "most frequent letter first", tie rule |
| 15 | Reorder Log Files | custom comparator + stability | two classes with contradictory orders |
| 16 | Custom Sort String | order table | the alphabet order is *given* |
| 17 | Count Inversions | merge sort, counting | "pairs in the wrong relative order" |
| 18 | Non-Overlapping Intervals | greedy by end time | "reject as few as possible", touching is fine |
| 19 | Burst Balloons | greedy by end point | "destroys every banner containing $x$", fewest darts |
| 20 | Two City Scheduling | sort by the difference | exactly half to each office, per-person costs |

**Read the table as a whole.** Eight of the twenty are binary search, which is why it earns three sections in this chapter. The right-hand column is the actual skill: every row is a sentence that a contest setter wrote to be read quickly and misread slowly. Learn to hear "$k$-th smallest" and start looking for a monotone predicate, and you will have learned more than the code below teaches.

</details>

---


## 3. Problem 1 — Three Sum

> **Statement.** The treasurer of a small club is preparing the annual audit. Every donation is recorded as a signed number - money in is positive, money out is negative - and the audit asks for the number of *kinds* of three-donation groups whose signed amounts cancel to exactly zero. Two groups are the same kind when they contain the same three amounts, no matter which members gave them or in what order the ledger lists them. A bad year puts three thousand entries in the ledger, and every amount satisfies $\lvert a \rvert \le 10^9$.


<details>
<summary>Solution</summary>

### Why this approach

Three nested loops are $O(n^3) = 2.7 \times 10^{10}$ — far too slow. Sorting enables the pair-sum trick: fix one element and find pairs summing to its negation with converging pointers. That is $O(n^2)$, which at $n = 3000$ is $9 \times 10^6$.

### Solution

Sort. For each anchor `i`, run converging pointers over the remaining suffix. The work is all in the duplicate handling: skip repeated anchors, and after a hit, skip repeated values on both pointers so the same triple is not counted twice.

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

    for (int i = 0; i + 2 < n; i++) {
        if (i > 0 && a[i] == a[i - 1]) continue;      // duplicate anchor

        int j = i + 1, k = n - 1;
        while (j < k) {
            long long s = a[i] + a[j] + a[k];
            if (s == 0) {
                ans++;
                while (j < k && a[j] == a[j + 1]) j++;   // duplicate second
                while (j < k && a[k] == a[k - 1]) k--;   // duplicate third
                j++; k--;
            } else if (s < 0) j++;
            else              k--;
        }
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
6
-1 0 1 2 -1 -4
```

**Output**
```text
2
```

Sorted: $-4, -1, -1, 0, 1, 2$. The two distinct triples are $\{-1, -1, 2\}$ and $\{-1, 0, 1\}$. The second `-1` is skipped as an anchor because it equals the first.

**Complexity.** $O(n^2)$ after an $O(n \log n)$ sort.

### The trap

The duplicate skips change the problem. Without them you count *index* triples, so $\{-1,-1,2\}$ is counted once per way of choosing which `-1` is which. Read whether the statement wants distinct values or distinct index triples — they differ by a factor equal to the number of repeats, and both appear in real problems.

</details>

---

## 4. Problem 2 — Squares of a Sorted Array

> **Statement.** A calibration rig logs signed offsets from a reference line, and it logs them already sorted - most negative first, most positive last. The physics module downstream cannot use offsets at all; it needs energies, the square of each offset, and it insists on receiving them in non-decreasing order as well. Produce the energy list. There are up to $2 \times 10^5$ offsets, and an offset may be as large as $10^9$ in absolute value.


<details>
<summary>Solution</summary>

### Why this approach

Squaring destroys the order: $-4$ becomes 16 and moves to the end. But the largest square must come from one of the two **ends** of the original array, because absolute value is unimodal. So fill the output from the back, taking the larger end square each time.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    vector<long long> res(n);
    int i = 0, j = n - 1, p = n - 1;

    while (i <= j) {
        long long l = a[i] * a[i];
        long long r = a[j] * a[j];
        if (l > r) { res[p--] = l; i++; }
        else       { res[p--] = r; j--; }
    }

    for (int t = 0; t < n; t++) cout << res[t] << " \n"[t + 1 == n];
    return 0;
}
```

**Input**
```text
5
-4 -1 0 3 10
```

**Output**
```text
0 1 9 16 100
```

**Complexity.** $O(n)$ — no sorting needed, because the order is recovered by construction.

### The trap

The array of squares must be `long long`. A value of $3 \times 10^9$ does not fit in `int`, and neither does its square times anything else you might multiply by. Here `a` is already `long long`, so `a[i] * a[i]` is computed in 64 bits.

</details>

---

## 5. Problem 3 — Partition Labels

> **Statement.** An archivist is cutting a long paper tape printed with lowercase letters into consecutive pieces. A letter is territorial: if one piece contains it, no other piece may. Subject to that rule, the archivist wants the largest possible number of pieces, and asks only for their lengths in order. The tape holds up to $2 \times 10^5$ letters.


<details>
<summary>Solution</summary>

### Why this approach

A part cannot end before the last occurrence of any letter inside it. So compute the last occurrence of every letter once, then scan: extend the current part's end to cover each letter's last occurrence; when the scan reaches that end, the part is closed.

This is a **greedy with a precomputed table**, not a window — there is no shrinking, only extending.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s;
    cin >> s;

    int last[26];
    fill(last, last + 26, -1);
    for (int i = 0; i < (int)s.size(); i++) last[s[i] - 'a'] = i;

    vector<int> parts;
    int start = 0, end = 0;

    for (int i = 0; i < (int)s.size(); i++) {
        end = max(end, last[s[i] - 'a']);
        if (i == end) {
            parts.push_back(end - start + 1);
            start = i + 1;
        }
    }

    for (int t = 0; t < (int)parts.size(); t++)
        cout << parts[t] << " \n"[t + 1 == (int)parts.size()];

    return 0;
}
```

**Input**
```text
ababcbacadefegdehijhklij
```

**Output**
```text
9 7 8
```

The first part is `ababcbaca` — it must extend to index 8 because that is where `a` occurs last. The second is `defegde`, and the third is `hijhklij`.

**Complexity.** $O(n)$ time, $O(1)$ space beyond the output.

### The trap

Computing `last` only for letters seen so far in the current part. The end of a part depends on the last occurrence of every letter *inside* it, which may be far to the right — so the table must be built in a full pass first.

</details>

---

## 6. Problem 4 — Longest Subarray With At Most $K$ Distinct

> **Statement.** A runner builds a single unbroken streak from an album of tracks, played in album order. She can push through at most $K$ different genres before the streak stops feeling like one workout, and she refuses to skip or reorder anything. What is the longest streak she can play? The album has up to $2 \times 10^5$ tracks, and $K$ never exceeds the number of distinct genres.


<details>
<summary>Solution</summary>

### Why this approach

"At most $K$ distinct" is monotone: adding an element can only increase the distinct count, removing from the left can only decrease it. That is exactly the sliding-window precondition, so the window is valid and the answer is found in one pass.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    cin >> n >> k;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    map<long long,int> cnt;
    int left = 0, best = 0;

    for (int right = 0; right < n; right++) {
        cnt[a[right]]++;

        while ((int)cnt.size() > k) {
            auto it = cnt.find(a[left]);
            if (--it->second == 0) cnt.erase(it);
            left++;
        }

        best = max(best, right - left + 1);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
5 2
1 2 1 2 3
```

**Output**
```text
4
```

The window `1 2 1 2` has two distinct values. Adding the `3` makes three, forcing `left` forward until a `1` or `2` is dropped — and the window `2 1 2 3` is still length 4.

**Complexity.** $O(n \log n)$ with `map`; $O(n)$ with `unordered_map`.

### The trap

Erasing the key when its count reaches zero. If you leave zeros in the map, `cnt.size()` counts keys that are no longer in the window and the `while` loop shrinks too far — producing a smaller answer that still looks plausible.

</details>

---

## 7. Problem 5 — Subarrays With Product Less Than $K$

> **Statement.** A fraud detector examines runs of consecutive transactions from a single account. Every amount is a positive integer, and a run is flagged as *suspicious* when the product of its amounts is strictly below a risk threshold $K$. The compliance team wants the number of non-empty runs that would be flagged. An account may show $3 \times 10^4$ transactions, and $K$ may be as large as $10^6$.


<details>
<summary>Solution</summary>

### Why this approach

Product, not sum — but all values are positive, so the product is monotone in window size exactly as a sum of positives is. The window shrinks by division. The key counting insight: when the window ending at `right` is valid, it contributes **`right - left + 1`** valid subarrays, one for each possible starting point.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    if (k <= 1) { cout << 0 << '\n'; return 0; }

    long long prod = 1, ans = 0;
    int left = 0;

    for (int right = 0; right < n; right++) {
        prod *= a[right];

        while (left <= right && prod >= k) {
            prod /= a[left];
            left++;
        }

        ans += right - left + 1;      // all suffixes ending at 'right'
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
4 100
10 5 2 6
```

**Output**
```text
8
```

The valid subarrays are `[10]`, `[10,5]`, `[5]`, `[5,2]`, `[5,2,6]`, `[2]`, `[2,6]`, `[6]` — eight of the ten. `[10,5,2]` equals 100 and is excluded by the strict inequality.

**Complexity.** $O(n)$ — each index enters and leaves the window once.

### The trap

`k <= 1` must be handled before the loop. With $k = 1$ and all values $\ge 1$, no product is ever below 1, so the window empties completely and `right - left + 1` becomes negative — silently corrupting the count.

</details>

---

## 8. Problem 6 — Remove Duplicates II

> **Statement.** A museum shelf holds artefacts sorted by inventory number. Curators tolerate at most two artefacts of any one number on the shelf, and the storage room is full, so the shelf must be compacted in place: every number keeps its first two appearances, everything after them is removed, and the surviving order is unchanged. Report how many artefacts remain and what the shelf looks like. The shelf may hold $2 \times 10^5$ artefacts, and there is no spare storage.


<details>
<summary>Solution</summary>

### Why this approach

Slow/fast pointers. The `slow` pointer is the write head, and the test for "may write this element" compares it against the element **two positions back in the output** — because that is the one which would make it a third copy.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> a(n);
    for (auto &x : a) cin >> x;

    int slow = 0;
    for (int fast = 0; fast < n; fast++) {
        if (slow < 2 || a[slow - 2] != a[fast]) {
            a[slow++] = a[fast];
        }
    }

    cout << slow << '\n';
    for (int t = 0; t < slow; t++) cout << a[t] << " \n"[t + 1 == slow];
    return 0;
}
```

**Input**
```text
6
1 1 1 2 2 3
```

**Output**
```text
5
1 1 2 2 3
```

The third `1` is rejected because `a[slow-2]` is also `1`. The two `2`s are both accepted; the `3` extends the array.

**Complexity.** $O(n)$ time, $O(1)$ extra space.

### The trap

Comparing against `a[fast-2]` (the input) instead of `a[slow-2]` (the output). In this problem the two coincide only when nothing has been removed yet — so for the first few elements it looks correct, and the error appears later in the input.

Note also `slow < 2 || ...`: the first two elements are always written, which is what avoids reading `a[-1]`.

</details>

---

## 9. Problem 7 — Find Peak Element

> **Statement.** A weather station sampled a mountain ridge at equally spaced points. No two consecutive readings are equal, and the profile may rise and fall as often as it likes. The survey needs the location of *some* point that is strictly higher than both of its immediate neighbours, with the two ends compared against an imaginary reading of minus infinity. The archive is enormous and the retrieval budget permits only a logarithmic number of probes, so walking the readings one by one is not an option. At least one such point exists.


<details>
<summary>Solution</summary>

### Why this approach

The array is **not sorted**, so ordinary binary search does not apply. But the predicate "a peak lies at or to the right of `mid`" is monotone: if `a[mid] < a[mid+1]`, the sequence is still climbing at `mid`, so a peak must exist to the right. Binary search works on that predicate.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    int lo = 0, hi = n - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < a[mid + 1]) lo = mid + 1;   // climbing: peak is right
        else                     hi = mid;       // descending: peak is here or left
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
4
1 2 3 1
```

**Output**
```text
2
```

`a[2] = 3` is greater than both `a[1] = 2` and `a[3] = 1`.

**Complexity.** $O(\log n)$.

### The trap

Adding a validity check like `if (mid > 0 && a[mid] > a[mid-1] && ...) return mid;` inside the loop. It is not wrong, but it invites the belief that the predicate is "is this a peak" — which is not monotone. The predicate that makes binary search work is "is the slope positive here", and that is a comparison of two adjacent elements only.

</details>

---

## 10. Problem 8 — Search in a Rotated Sorted Array

> **Statement.** During a move, the pages of a warehouse manifest were rotated: they had been in increasing order of item code, then the stack was cut somewhere and the two parts were swapped, each part keeping its internal order. A customer asks whether a particular code is on the manifest, and the clerk wants its page number, counting pages from zero. The manifest holds up to $2 \times 10^5$ distinct codes, and the clerk may not walk the stack - only a logarithmic number of page opens is permitted. Report the page number, or $-1$ if the code is absent.


<details>
<summary>Solution</summary>

### Why this approach

Standard binary search needs sorted order, and the array is split into two sorted runs. But at any midpoint, **at least one of the two halves is fully sorted** — you can tell which by comparing `a[lo]` with `a[mid]`. Search inside the sorted half if the target lies in its range, otherwise search the other.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, target;
    cin >> n >> target;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    int lo = 0, hi = n - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;

        if (a[mid] == target) { cout << mid << '\n'; return 0; }

        if (a[lo] <= a[mid]) {                       // left half sorted
            if (a[lo] <= target && target < a[mid]) hi = mid - 1;
            else                                    lo = mid + 1;
        } else {                                     // right half sorted
            if (a[mid] < target && target <= a[hi]) lo = mid + 1;
            else                                    hi = mid - 1;
        }
    }

    cout << -1 << '\n';
    return 0;
}
```

**Input**
```text
7 0
4 5 6 7 0 1 2
```

**Output**
```text
4
```

The `0` sits at index 4, immediately after the rotation point.

**Complexity.** $O(\log n)$.

### The trap

Using `<` instead of `<=` in `a[lo] <= a[mid]`. When `lo == mid` — which happens once the range has shrunk to two elements — the left "half" is a single element and must be treated as sorted. With `<`, the code takes the wrong branch and the search can terminate on the wrong element.

</details>

---

## 11. Problem 9 — Minimum Days to Make Bouquets

> **Statement.** A florist rents $n$ flower beds in a row; bed $i$ produces its flowers on day $b_i$ and nothing before that. A wedding order asks for $m$ bouquets, each requiring exactly $k$ *adjacent* beds in bloom on the same day, and no bed may be used twice. The florist wants the earliest morning on which the order can be assembled, or a report that it never can be. There are up to $2 \times 10^5$ beds, bloom days up to $10^9$, and an order may be impossible even when the beds are numerous.


<details>
<summary>Solution</summary>

### Why this approach

The answer is a day, and feasibility is monotone: if day $d$ works, every later day works. So binary search on the day, with a greedy check that counts groups of $k$ consecutive bloomed flowers.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, k;
    long long m;
    cin >> n >> m >> k;

    vector<long long> bloom(n);
    for (auto &x : bloom) cin >> x;

    auto ok = [&](long long day) {
        long long groups = 0, run = 0;
        for (long long x : bloom) {
            if (x <= day) {
                run++;
                if (run == k) { groups++; run = 0; }
            } else {
                run = 0;                    // a non-bloomed flower breaks the run
            }
        }
        return groups >= m;
    };

    long long lo = 1, hi = *max_element(bloom.begin(), bloom.end());
    if (!ok(hi)) { cout << -1 << '\n'; return 0; }

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
5 3 1
1 10 3 10 2
```

**Output**
```text
3
```

With `k = 1`, each bloomed flower is its own bouquet. On day 3 three flowers have bloomed — positions 0, 2 and 4 — so three bouquets exist.

**Complexity.** $O(n \log(\max \text{bloom}))$.

### The trap

Forgetting the `if (!ok(hi))` pre-check. Without it, the binary search converges on `hi` even when the task is impossible, and prints a day that does not actually work. The `-1` case is easy to miss because it needs `m * k > n` as well as the bloom constraint.

</details>

---

## 12. Problem 10 — K-th Smallest Pair Distance

> **Statement.** An acoustic engineer measures $n$ lamps along a corridor by position. The disturbance caused by a pair of lamps is the absolute difference of their positions, and the engineer ranks all pairs by disturbance, counting equal disturbances as separate ranks. She needs the $k$-th smallest value in that ranking. There can be $10^4$ lamps, so the ranking is far too long to write down.


<details>
<summary>Solution</summary>

### Why this approach

The number of pairs is $5 \times 10^7$ — too many to materialise. But "how many pairs have difference $\le d$?" is easy after sorting (two pointers), and it is monotone in $d$. So binary search the **distance**, not the answer list.

This is the answer-search pattern applied to a counting predicate rather than a feasibility check.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;
    sort(a.begin(), a.end());

    auto countLE = [&](long long d) {
        long long c = 0;
        int j = 0;
        for (int i = 0; i < n; i++) {
            if (j < i + 1) j = i + 1;
            while (j < n && a[j] - a[i] <= d) j++;
            c += j - i - 1;
        }
        return c;
    };

    long long lo = 0, hi = a[n-1] - a[0];
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (countLE(mid) >= k) hi = mid;
        else                   lo = mid + 1;
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
3 1
1 3 1
```

**Output**
```text
0
```

Sorted: $1, 1, 3$. The three distances are $0, 2, 2$; the smallest is 0.

**Complexity.** $O(n \log n + n \log(\text{range}))$ — the counting function is $O(n)$ because `j` never moves backwards.

### The trap

Resetting `j = 0` inside the loop for each `i`. That makes the counting $O(n^2)$ and the whole solution $O(n^2 \log(\text{range}))$, which at $n = 10^4$ is slow enough to matter. The shared `j` is what makes it linear, and it works because `a[j] - a[i]` decreases as `i` increases.

</details>

---

## 13. Problem 11 — Median of Two Sorted Arrays

> **Statement.** Two libraries hold their catalogue numbers in increasing order. A historian needs the median of the combined catalogue, but the collections are on loan: the numbers may be read where they lie and must not be merged, copied, or re-shelved. The procedure may open only a logarithmic number of entries, measured in the smaller collection. Print the median with four digits after the decimal point. Each library holds up to $2 \times 10^5$ numbers.


<details>
<summary>Solution</summary>

### Why this approach

Merging is $O(n + m)$, which is acceptable on most problems and useless when $O(\log)$ is demanded. The logarithmic solution binary searches the **partition point**: how many elements of the smaller array go on the left side of the median.

At a correct partition, the largest left element is $\le$ the smallest right element in both arrays. That single condition is what the search tests, and it is monotone in the partition index.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;

    vector<long long> A(n), B(m);
    for (auto &x : A) cin >> x;
    for (auto &x : B) cin >> x;

    if (n > m) { swap(n, m); swap(A, B); }   // binary search the smaller array

    int lo = 0, hi = n;
    int half = (n + m + 1) / 2;
    double median = 0;

    while (lo <= hi) {
        int i = (lo + hi) / 2;               // elements of A on the left
        int j = half - i;                    // elements of B on the left

        long long aL = (i == 0) ? LLONG_MIN : A[i-1];
        long long aR = (i == n) ? LLONG_MAX : A[i];
        long long bL = (j == 0) ? LLONG_MIN : B[j-1];
        long long bR = (j == m) ? LLONG_MAX : B[j];

        if (aL <= bR && bL <= aR) {
            if ((n + m) % 2) median = max(aL, bL);
            else             median = (max(aL, bL) + min(aR, bR)) / 2.0;
            break;
        } else if (aL > bR) hi = i - 1;      // too many from A
        else                lo = i + 1;      // too few from A
    }

    cout << fixed << setprecision(4) << median << '\n';
    return 0;
}
```

**Input**
```text
2 1
1 3
2
```

**Output**
```text
2.0000
```

Combined: $1, 2, 3$ — the median is 2.

**Complexity.** $O(\log(\min(n, m)))$ time after reading the input.

### The trap

The sentinels. Using $0$ instead of `LLONG_MIN` / `LLONG_MAX` breaks whenever the arrays contain negatives or large values, and the failure is data-dependent. Also note `half = (n+m+1)/2` — the `+1` makes the left side the larger one for odd totals, so the `max(aL, bL)` branch is correct for both parities.

</details>

---

## 14. Problem 12 — K-th Smallest in a Sorted Matrix

> **Statement.** An auditor is given an $n \times n$ ledger in which every row increases from left to right and every column increases from top to bottom. She wants the $k$-th smallest entry of the whole ledger, counting repetitions as separate entries. The ledger is at most $1000 \times 1000$, and $k$ counts entries, not distinct values.


<details>
<summary>Solution</summary>

### Why this approach

The matrix is not globally sorted, so indexing by position is out. But "how many elements are $\le x$?" is computable in $O(n)$ with a staircase walk from the bottom-left corner, and it is monotone in $x$. Binary search the **value**.

The staircase works because starting at the bottom-left, moving up decreases the value and moving right increases it — so each step eliminates a row or a column.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<vector<long long>> a(n, vector<long long>(n));
    for (auto &row : a) for (auto &x : row) cin >> x;

    auto countLE = [&](long long x) {
        long long cnt = 0;
        int r = n - 1, c = 0;                 // bottom-left
        while (r >= 0 && c < n) {
            if (a[r][c] <= x) { cnt += r + 1; c++; }   // whole column above, plus this
            else               r--;                    // too big: move up
        }
        return cnt;
    };

    long long lo = a[0][0], hi = a[n-1][n-1];
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (countLE(mid) >= k) hi = mid;
        else                   lo = mid + 1;
    }

    cout << lo << '\n';
    return 0;
}
```

**Input**
```text
3 8
1 5 9
10 11 13
12 13 15
```

**Output**
```text
13
```

The sorted order is $1, 5, 9, 10, 11, 12, 13, 13, 15$; the eighth is 13.

**Complexity.** $O(n \log(\text{range}))$ time, $O(n^2)$ to store the matrix.

### The trap

Binary searching the *index* from `1` to `n*n` and mapping to a cell. There is no monotone mapping from index to position in a row-and-column sorted matrix — only from value to count. The count is monotone; the position is not.

</details>

---

## 15. Problem 13 — Longest Increasing Subsequence in $O(n \log n)$

> **Statement.** An analyst follows a sequence of measurements and wants the longest *subsequence* that is strictly increasing - elements taken in their original order, gaps allowed. The sequence can contain $2 \times 10^5$ measurements, with values up to $10^9$.


<details>
<summary>Solution</summary>

### Why this approach

The $O(n^2)$ DP is the natural solution and it is too slow. The faster version maintains `tails[len]` = the **smallest possible tail** of an increasing subsequence of length `len`. That array is sorted, so each element can be placed with `lower_bound`.

`tails` is not a subsequence of the input — it is a table of best-case endings, which is why only its length is the answer.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    vector<long long> tails;             // tails[len-1] = smallest tail of length len
    for (long long x : a) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x);
        else                   *it = x;
    }

    cout << tails.size() << '\n';
    return 0;
}
```

**Input**
```text
8
10 9 2 5 3 7 101 18
```

**Output**
```text
4
```

One longest increasing subsequence is $2, 3, 7, 101$ — four elements.

**Complexity.** $O(n \log n)$ time, $O(n)$ space.

### The trap

Using `upper_bound` instead of `lower_bound` for the **strictly** increasing version. `lower_bound` replaces the first element $\ge x$, which is right for strict increase; `upper_bound` would allow equal elements to extend the sequence, silently answering the *non-decreasing* version instead. The two variants differ by exactly that one call.

</details>

---

## 16. Problem 14 — Sort Characters by Frequency

> **Statement.** A typesetter receives a manuscript as a single word and must rearrange its letters: the most frequent letter comes first, and letters with equal frequency appear in alphabetical order. Print the reordered word. The word may be $2 \times 10^5$ letters long and contains only lowercase letters.


<details>
<summary>Solution</summary>

### Why this approach

Counting is $O(n)$; the ordering is a two-key comparator. Since the alphabet is fixed, this is also a case where a counting sort over 26 buckets beats a comparison sort — but the comparator version is the one that generalises.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s;
    cin >> s;

    int cnt[26] = {0};
    for (char c : s) cnt[c - 'a']++;

    vector<int> order;
    for (int i = 0; i < 26; i++) if (cnt[i]) order.push_back(i);

    sort(order.begin(), order.end(), [&](int x, int y) {
        if (cnt[x] != cnt[y]) return cnt[x] > cnt[y];   // frequency descending
        return x < y;                                    // character ascending
    });

    for (int c : order)
        cout << string(cnt[c], char('a' + c));

    cout << '\n';
    return 0;
}
```

**Input**
```text
tree
```

**Output**
```text
eert
```

`e` occurs twice and comes first; `r` and `t` each occur once, so they are ordered alphabetically.

**Complexity.** $O(n + 26 \log 26)$ — effectively $O(n)$.

### The trap

Forgetting the tie-break, which makes the output depend on `sort`'s internal order for equal counts — it will pass the sample by luck and fail on hidden tests. The statement almost always specifies the tie rule; if it says "any order", any tie-break is safe, and if it is silent, alphabetic is the safe default.

</details>

---

## 17. Problem 15 — Reorder Log Files

> **Statement.** A server writes $n$ log lines. Each line begins with an identifier, then either a sequence of lowercase words or a sequence of numbers, never both. Operators want the word lines printed first, ordered by everything after the identifier (ties broken by the identifier itself), followed by the number lines in the exact order the server wrote them. Reproduce the reordered log. There may be up to $2 \times 10^5$ lines.


<details>
<summary>Solution</summary>

### Why this approach

Two classes with two different rules, in one output — a job for one comparator that handles both, plus `stable_sort` (or an explicit rule) to preserve digit-log order.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    cin.ignore();

    vector<string> logs(n);
    for (auto &l : logs) getline(cin, l);

    auto content = [](const string &s) {         // text after the identifier
        size_t p = s.find(' ');
        return p == string::npos ? string() : s.substr(p + 1);
    };
    auto id = [](const string &s) {
        size_t p = s.find(' ');
        return p == string::npos ? s : s.substr(0, p);
    };
    auto isLetter = [&](const string &s) {
        size_t p = s.find(' ');
        return p != string::npos && p + 1 < s.size() && isalpha((unsigned char)s[p + 1]);
    };

    stable_sort(logs.begin(), logs.end(), [&](const string &x, const string &y) {
        bool lx = isLetter(x), ly = isLetter(y);
        if (lx != ly) return lx;                 // letter-logs first
        if (!lx) return false;                   // both digit-logs: keep input order
        if (content(x) != content(y)) return content(x) < content(y);
        return id(x) < id(y);                    // tie: identifier ascending
    });

    for (auto &l : logs) cout << l << '\n';
    return 0;
}
```

**Input**
```text
4
a2 act car
b1 act car
c3 7 8 9
d4 own kit
```

**Output**
```text
a2 act car
b1 act car
d4 own kit
c3 7 8 9
```

`a2` and `b1` have identical content, so the identifier decides. `d4`'s content `own kit` sorts after `act car`. The digit-log `c3` goes last.

**Complexity.** $O(n \log n)$ comparisons, each $O(L)$ in the line length.

### The trap

Returning `false` for both digit logs is only correct with `stable_sort`. With plain `sort` the digit-log order is unspecified, and the statement explicitly requires input order. This is one of the rare cases where `stable_sort` is the right tool rather than an explicit tie-break, because the required tie-break is "original position" and that key is not available to the comparator.

</details>

---

## 18. Problem 16 — Custom Sort String

> **Statement.** A deciphering team has recovered the order of an alien alphabet as a string `order`, and intercepted a message `s` written in that alphabet. To reconstruct the message they retypeset it so that its letters follow the alien order; any letter that does not occur in the recovered alphabet is appended at the end in alphabetical order. Both strings contain only lowercase letters and can be $2 \times 10^5$ long.


<details>
<summary>Solution</summary>

### Why this approach

The ordering is **given as data**, not computable — so build a rank table: `rank[c]` is the position of `c` in `order`, or a large value if absent. Then a comparator on that table sorts everything in one pass.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string order, s;
    cin >> order >> s;

    int rankOf[26];
    fill(rankOf, rankOf + 26, 100);          // not in order: sort last
    for (int i = 0; i < (int)order.size(); i++) rankOf[order[i] - 'a'] = i;

    int cnt[26] = {0};
    for (char c : s) cnt[c - 'a']++;

    string out;
    for (char c : order)
        out += string(cnt[c - 'a'], c);

    for (int i = 0; i < 26; i++)             // the leftovers, alphabetically
        if (rankOf[i] == 100) out += string(cnt[i], char('a' + i));

    cout << out << '\n';
    return 0;
}
```

**Input**
```text
cba abcd
```

**Output**
```text
cbad
```

`c`, `b`, `a` follow `order`; `d` is not in `order` and is appended.

**Complexity.** $O(|order| + |s| + 26)$.

### The trap

Building the output with `order` only, and forgetting letters of `s` that do not appear in `order` — they vanish entirely, and the output is shorter than the input, which some judges detect as a wrong length rather than a wrong string.

</details>

---

## 19. Problem 17 — Count Inversions

> **Statement.** A jury produced a ranking of $n$ contestants. The official order is increasing, and the organisers measure how scrambled the jury's list is by counting the pairs of contestants that appear in the wrong relative order - pairs $i < j$ with `a[i] > a[j]`. Report that number. There may be $2 \times 10^5$ contestants.


<details>
<summary>Solution</summary>

### Why this approach

$O(n^2)$ pairs is too many. Merge sort can count them for free: whenever the right half's element is placed before the remaining left half, every remaining element of the left half forms an inversion with it. So the inversion count is a by-product of the merge, at no extra asymptotic cost.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long invCount = 0;

void mergeSort(vector<long long> &a, vector<long long> &buf, int l, int r) {
    if (r - l <= 1) return;
    int mid = (l + r) / 2;

    mergeSort(a, buf, l, mid);
    mergeSort(a, buf, mid, r);

    int i = l, j = mid, k = l;
    while (i < mid && j < r) {
        if (a[i] <= a[j]) {
            buf[k++] = a[i++];
        } else {
            invCount += mid - i;     // all remaining left elements are greater
            buf[k++] = a[j++];
        }
    }
    while (i < mid) buf[k++] = a[i++];
    while (j < r)   buf[k++] = a[j++];

    for (int t = l; t < r; t++) a[t] = buf[t];
}

int main() {
    int n;
    cin >> n;

    vector<long long> a(n), buf(n);
    for (auto &x : a) cin >> x;

    mergeSort(a, buf, 0, n);

    cout << invCount << '\n';
    return 0;
}
```

**Input**
```text
3
3 1 2
```

**Output**
```text
2
```

The inversions are $(3,1)$ and $(3,2)$.

**Complexity.** $O(n \log n)$ time, $O(n)$ extra space.

### The trap

`invCount += mid - i` counts the elements **still unread** in the left half — not `mid - i - 1`, and not `j - mid`. When `a[j]` is taken before `a[i]`, every left element from `i` to `mid-1` is greater than `a[j]` (the left half is sorted), so exactly `mid - i` new inversions are created.

Using `int` for the counter also overflows: the maximum is $n(n-1)/2 \approx 2 \times 10^{10}$ at $n = 2 \times 10^5$.

</details>

---

## 20. Problem 18 — Non-Overlapping Intervals

> **Statement.** A hall receives proposals as time intervals. The committee must reject as few proposals as possible so that the accepted ones never overlap in time; proposals that merely touch at an endpoint are compatible. Report how many proposals must be rejected. There may be up to $2 \times 10^5$ proposals.


<details>
<summary>Solution</summary>

### Why this approach

Equivalently: maximise the number kept. That is activity selection, whose correct key is **end time ascending** — keeping the interval that finishes earliest leaves the most room for the rest. The answer is `n - kept`.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<long long,long long>> iv(n);
    for (auto &p : iv) cin >> p.first >> p.second;

    sort(iv.begin(), iv.end(), [](const auto &a, const auto &b) {
        return a.second < b.second;              // by end time
    });

    long long kept = 0;
    long long lastEnd = LLONG_MIN;
    for (auto &p : iv) {
        if (p.first >= lastEnd) {                // starts after the previous ends
            kept++;
            lastEnd = p.second;
        }
    }

    cout << n - kept << '\n';
    return 0;
}
```

**Input**
```text
4
1 2
2 3
3 4
1 3
```

**Output**
```text
1
```

Keeping `[1,2], [2,3], [3,4]` gives three mutually compatible intervals; only `[1,3]` must go.

**Complexity.** $O(n \log n)$.

### The trap

Sorting by start time. It is the natural choice and it is wrong: an interval that starts early and ends late blocks everything after it. The exchange argument from Chapter 2 applies unchanged — earlier finishing is never worse.

Note `p.first >= lastEnd`: intervals that merely touch are compatible here. If the statement says otherwise, change the comparison.

</details>

---

## 21. Problem 19 — Minimum Arrows to Burst Balloons

> **Statement.** Banners hang along a street, each spanning an interval $[l, r]$ of positions. A single dart thrown at position $x$ destroys every banner whose interval contains $x$. Find the smallest number of darts needed to destroy all banners. There may be up to $2 \times 10^5$ banners.


<details>
<summary>Solution</summary>

### Why this approach

A greedy in the same family as Problem 18: sort by **right endpoint**, shoot at the first uncovered balloon's right end, and skip every balloon that contains that point. Shooting at the right end is optimal because it is the last possible moment to cover the current balloon, so it covers the maximum possible set.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<long long,long long>> b(n);
    for (auto &p : b) cin >> p.first >> p.second;

    sort(b.begin(), b.end(), [](const auto &a, const auto &c) {
        return a.second < c.second;
    });

    long long arrows = 0, shot = LLONG_MIN;
    for (auto &p : b) {
        if (p.first > shot) {          // not covered by the last arrow
            arrows++;
            shot = p.second;           // shoot at its right end
        }
    }

    cout << arrows << '\n';
    return 0;
}
```

**Input**
```text
4
10 16
2 8
1 6
7 12
```

**Output**
```text
2
```

An arrow at 6 bursts `[1,6]` and `[2,8]`. An arrow at 12 bursts `[7,12]` and `[10,16]`.

**Complexity.** $O(n \log n)$.

### The trap

`p.first > shot` versus `p.first >= shot`. Arrows burst closed intervals, so a balloon starting exactly at the arrow position **is** burst, and the test must be strict `>`. Using `>=` adds a spurious arrow for every balloon that touches the previous one.

</details>

---

## 22. Problem 20 — Two City Scheduling

> **Statement.** A company must staff two offices equally: exactly half of its $n$ employees, where $n$ is even, go to the northern office and half to the southern one. Sending employee $i$ north costs $a_i$ and sending them south costs $b_i$, and the amounts are not comparable between offices. Minimise the total cost. There may be $2 \times 10^5$ employees.

<details>
<summary>Solution</summary>

### Why this approach

Everything hinges on a single derived quantity: the **penalty** for sending someone to A instead of B, $a_i - b_i$. Sorting by that difference and giving the $n/2$ most-negative penalties to A is optimal.

**Exchange argument.** Suppose in an optimal assignment some person $x$ goes to B while $y$ goes to A, with $a_x - b_x < a_y - b_y$. Swapping them changes the cost by
$$(a_y + b_x) - (a_x + b_y) = (a_y - b_y) - (a_x - b_x) > 0.$$
So the swap strictly improves, contradiction. Therefore the $n/2$ smallest differences must go to A.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<long long,long long>> p(n);   // (costA, costB)
    for (auto &q : p) cin >> q.first >> q.second;

    sort(p.begin(), p.end(), [](const auto &x, const auto &y) {
        return (x.first - x.second) < (y.first - y.second);
    });

    long long total = 0;
    for (int i = 0; i < n; i++) total += (i < n / 2) ? p[i].first : p[i].second;

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
4
10 20
30 200
400 50
30 20
```

**Output**
```text
110
```

Differences: $-10, -170, 350, 10$. The two smallest go to A — person 1 (30) and person 0 (10) — for 40. The rest go to B: person 3 (20) and person 2 (50) — for 70. Total 110.

**Complexity.** $O(n \log n)$.

### The trap

Sorting by `a_i - b_i` as `int`. The individual costs can be up to $10^9$, and while the difference itself fits, the **sum** does not — the total is a `long long`. In this code the pairs are `long long`, so the difference is computed in 64 bits; sorting by a difference computed in `int` would also overflow at the extremes.

</details>

---

## 23. The ten unsolved problems

| # | Problem | Constraints |
|---|---|---|
| 1 | **Four Sum.** Count the **distinct value quadruples** summing to a target. | $n \le 200$ |
| 2 | **Minimum Window Substring.** Find the shortest substring of `s` containing every character of `t` (with multiplicity). | $\lvert s \rvert, \lvert t \rvert \le 10^5$ |
| 3 | **Trapping Rain Water.** Given column heights, compute the total water trapped. | $n \le 2 \times 10^5$ |
| 4 | **K-th Element of Two Sorted Arrays.** Find the $k$-th smallest of the combined multiset in $O(\log(\min(n,m)))$. | $n, m \le 2 \times 10^5$ |
| 5 | **K Closest Elements.** From a sorted array, output the $k$ values closest to a target, in ascending order. Ties favour the smaller value. | $n \le 2 \times 10^5$ |
| 6 | **Sort Array by Parity II.** Rearrange in place so even indices hold even values and odd indices hold odd values. | $n \le 2 \times 10^4$ |
| 7 | **Maximum Gap.** Find the largest difference between consecutive elements of the sorted array, in linear time. | $n \le 5 \times 10^4$ |
| 8 | **Wiggle Sort.** Rearrange so that $a_1 < a_2 > a_3 < a_4 \dots$ | $n \le 10^5$ |
| 9 | **Minimum in a Rotated Array II.** Same as the distinct version, but the array may contain duplicates. | $n \le 5000$ |
| 10 | **Contains Duplicate III.** Decide whether there are $i \ne j$ with $\lvert i - j \rvert \le k$ and $\lvert a_i - a_j \rvert \le t$. | $n \le 2 \times 10^5$ |

**Ordered hints, weakest first:**

- Problem 6 is a slow/fast pointer where `fast` searches for the *next element of the right parity*.
- Problem 8 is solved by sorting and then interleaving the two halves — no comparator needed.
- Problem 7 is a bucket argument: with $n$ numbers in a range of $R$, some bucket must be empty, so the answer exceeds $R/(n-1)$.
- Problem 9 needs a third branch: when `a[mid] == a[hi]`, there is no way to tell which side is sorted, so shrink `hi` by one and retry.
- Problem 10 uses an ordered `set` of values in the current window and `lower_bound` against $a_i - t$.

---

## 24. What the twenty have in common

| Observation | Problems |
|---|---|
| The technique is chosen by the **constraint**, not the story | 1, 7, 10, 11, 12, 13 |
| Two techniques are combined | 3, 9, 10, 12, 15, 19, 20 |
| The bug is in the **boundary**, not the algorithm | 2, 6, 8, 16, 18, 19 |
| Duplicates or ties decide correctness | 1, 6, 14, 15, 17 |
| A derived quantity is sorted instead of the input | 20, 13, 7 |
| The greedy needs a proof, not a hunch | 18, 19, 20 |

Three conclusions follow from that table.

**First:** six of the twenty are difficult for a reason unrelated to the algorithm. They are difficult because of an equality, a boundary, or a duplicate. Reading the statement for those details is worth more than another hour of coding.

**Second:** the combinations are always the same shape — *sort, then search*, or *precompute, then scan*. Problems 3, 9, 10, 12 and 15 are all "build a table, then walk it". If you find yourself unsure how to combine two techniques, that is almost always the combination.

**Third:** three of the twenty need an exchange argument, and none of those three is a "sorting problem" in the statement. Greedy hides inside ordinary-sounding sentences.

---

## Key Takeaways

- The constraint chooses the technique. $n \le 3000$ with triples is $O(n^2)$; $n \le 2 \times 10^5$ is $O(n \log n)$; "in $O(\log)$" rules out anything that reads the array.
- Sorting is the universal enabler: it converts pair-sum, closest-pair, merge, and counting problems into linear scans.
- Fixed anchor plus two pointers handles every $k$-sum variant; the anchor loop makes it $O(n^{k-1})$.
- Distinct-value counting and index-triple counting differ, and duplicate skips must match the statement.
- Sliding windows require a monotone property; when one exists, incremental state beats rescanning by a factor of $n$.
- The counting identity `ans += right - left + 1` counts subarrays **ending at `right`**, not subarrays in general.
- Binary search needs a monotone predicate, not a sorted array. Peak finding is the cleanest proof.
- In a rotated array, at least one half is always sorted — detect which, then decide.
- Answer-search = monotone check + a bracket that contains the answer. Minimise uses `hi = mid`, maximise uses `lo = mid`.
- The median of two arrays is a **partition** problem, not a merge problem, and the sentinels must be true infinities.
- When the answer must be found among values, binary search the value and count with a staircase or a two-pointer sweep.
- `tails` + `lower_bound` gives LIS in $O(n \log n)$; `upper_bound` silently changes the problem to non-decreasing.
- Merge sort counts inversions for free — one extra addition in the merge.
- Greedy interval problems sort by **end**, never by start. Touching intervals need the strictness decided explicitly.
- Sorting a **derived** quantity (a difference, a penalty, a ratio) is often the whole greedy argument.
- `stable_sort` is required when the tie-break is "original position"; otherwise an explicit index tie-break is cheaper.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Counting index triples when distinct values are wanted | Answers multiplied by the repeat count | Skip duplicates on all three positions |
| 2 | Duplicate skips applied to a problem that counts indices | Undercount | Read the statement |
| 3 | Rescanning the window each step | $O(n^2)$ | Maintain incremental state |
| 4 | Leaving zero counts in a `map` during a window | `size()` is inflated; window over-shrinks | Erase at zero |
| 5 | `k <= 1` unhandled in a product window | Window empties; count goes negative | Early return |
| 6 | Comparing against the input instead of the output in slow/fast | Correct early, wrong later | `a[slow-2]` |
| 7 | Peak search with an explicit "is peak" test | Predicate is not monotone | Compare with `a[mid+1]` |
| 8 | `<` instead of `<=` in rotated search | Wrong branch when `lo == mid` | `a[lo] <= a[mid]` |
| 9 | Missing the impossible case in answer search | Prints a feasible-looking day | Check `ok(hi)` first |
| 10 | Resetting the pointer in a two-pointer count | $O(n^2)$ | Share `j` across `i` |
| 11 | `0` sentinels in the median partition | Breaks on negative or large values | `LLONG_MIN` / `LLONG_MAX` |
| 12 | Binary searching a position in a sorted matrix | No monotone position-to-value map | Search the value |
| 13 | `upper_bound` for strictly increasing LIS | Answers the wrong variant | `lower_bound` |
| 14 | No tie-break in a frequency sort | Order depends on `sort` internals | Explicit secondary key |
| 15 | `sort` when input order must survive | Digit-log order unspecified | `stable_sort` |
| 16 | Omitting letters absent from `order` | Output silently shorter | Append leftovers |
| 17 | `mid - i - 1` in inversion counting | Off by the number of merges | `mid - i` |
| 18 | `int` inversion counter | Overflows at $2 \times 10^{10}$ | `long long` |
| 19 | Sorting intervals by start for max-kept | Blocks later intervals | Sort by end |
| 20 | `>=` for closed-interval contact | Spurious extra arrow | `>` |

---

## Practice Problems

The ten unsolved problems in §23 are the practice set. Work them in technique order rather than listed order:

| Order | Problems | Why |
|---|---|---|
| 1 | 6, 8 | Pure two-pointer rearrangement; no predicate to prove |
| 2 | 5, 7 | Sort-based counting and bucketing |
| 3 | 3 | Two pointers with a running maximum on each side |
| 4 | 9 | Binary search with a third branch — the subtle case |
| 5 | 4 | Partition search, the same idea as Problem 11 |
| 6 | 2 | Sliding window with a satisfiability counter |
| 7 | 10 | Ordered set plus `lower_bound` |
| 8 | 1 | Two anchors plus pointers — the $O(n^3)$ → $O(n^2)$ generalisation |

For a wider pool at this level, the CSES *Sorting and Searching* section covers every technique in this topic except the comparator material.

---

## Next Up

→ [Topic 05 — Graph Theory Beginner Part 1](Topic%2005%20-%20Graph%20Theory%20Beginner%20Part%201%20%28Graph%20Traversal%20and%20Flood%20Fill%29.md)

Everything so far has been about arrays. The next two topics move to graphs — first traversal and flood fill, where the whole difficulty is seeing the graph inside a grid or a set of states, and then trees, where the recursion has structure you can exploit.
