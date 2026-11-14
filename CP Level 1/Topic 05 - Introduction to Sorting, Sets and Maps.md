# Topic 05 — Introduction to Sorting, Sets and Maps

Sorting is not an operation you perform at the end of a solution. It is a **transformation that makes hard problems easy**: it turns "compare every pair" into "scan once", "find duplicates" into "look at neighbours", and "answer a query" into a binary search. Sets and maps are the same idea kept live — order maintained as data changes.

Chapter 1 taught you the syntax. This topic teaches you the four moves that turn an $\mathcal{O}(n^2)$ idea into an accepted $\mathcal{O}(n \log n)$, and the three structures you reach for when the data keeps changing.

> **Prerequisites:** [Topic 04 — Complete Searching](Topic%2004%20-%20Complete%20Searching.md), [Chapter 1, Topic 15 — Algorithms](../CPP%20for%20CP/Topic%2015%20-%20STL%20Part%204%20(Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators).md), [Chapter 1, Topic 13 — Set and Map](../CPP%20for%20CP/Topic%2013%20-%20STL%20Part%202%20(Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps).md)
> **You will be able to:**
> - Recognise the five situations where sorting is the whole solution.
> - Replace an $\mathcal{O}(n^2)$ pairwise scan with sort plus two pointers.
> - Use the predecessor/successor pattern for "next greater" and "closest value" queries.
> - Apply prefix sums plus a map to count subarrays with a given sum.
> - Choose between sorting once, a live `set`, and a `map`, with a stated reason.

---

## 1. Sorting as a technique

A sort costs $\mathcal{O}(n \log n)$. You accept that cost because it buys you a property that makes everything afterwards cheaper. There are exactly five things it buys:

| What sorting gives you | Problem it unlocks |
|---|---|
| **Monotonicity** | Two pointers, binary search, greedy exchange arguments |
| **Adjacency of equals** | Duplicate detection, deduplication, mode |
| **Adjacency of near-values** | Minimum difference, closest pair |
| **A canonical order** | Grouping by key, comparing multisets, matching |
| **A greedy order** | Interval scheduling, deadline problems, exchange arguments |

Every "sorting problem" in a contest is one of those five. Identify which, and the rest is bookkeeping.

**The cost-benefit test:** if your current approach examines all $\binom{n}{2}$ pairs, sorting plus a linear scan is almost always $\mathcal{O}(n \log n)$ and almost always correct. Ask: *what would I know if the array were sorted?*

---

## 2. Pattern 1 — Sort plus two pointers

The workhorse. After sorting, a single pass with two indices replaces a double loop.

### 2.1 Pair sum

Does any pair sum to a target?

```cpp
sort(a.begin(), a.end());
int i = 0, j = n - 1;
bool found = false;
while (i < j) {
    long long s = (long long)a[i] + a[j];
    if (s == target) { found = true; break; }
    if (s < target) i++;      // need a larger sum
    else            j--;      // need a smaller sum
}
```

**Why it is correct.** If `a[i] + a[j] < target`, then `a[i]` paired with anything at or below `a[j]` is also too small — so `i` can never be part of a solution, and we discard it. Symmetrically for `j`. Each step permanently discards one index, so the loop runs at most $n$ times.

**Complexity:** $\mathcal{O}(n \log n)$ for the sort plus $\mathcal{O}(n)$ for the scan. The naive version is $\mathcal{O}(n^2)$.

| Approach | Time | Space | When |
|---|---|---|---|
| Double loop | $\mathcal{O}(n^2)$ | $\mathcal{O}(1)$ | $n \le 5000$ |
| Sort + two pointers | $\mathcal{O}(n \log n)$ | $\mathcal{O}(1)$ | Cheapest, but destroys order |
| Hash map of seen values | $\mathcal{O}(n)$ expected | $\mathcal{O}(n)$ | When you need original indices |

### 2.2 Three-sum

```cpp
sort(a.begin(), a.end());
for (int i = 0; i + 2 < n; i++) {
    if (i > 0 && a[i] == a[i-1]) continue;      // skip duplicate first elements
    int j = i + 1, k = n - 1;
    while (j < k) {
        long long s = (long long)a[i] + a[j] + a[k];
        if (s == target) {
            // record (a[i], a[j], a[k])
            while (j < k && a[j] == a[j+1]) j++;   // skip duplicates
            while (j < k && a[k] == a[k-1]) k--;
            j++; k--;
        } else if (s < target) j++;
        else                    k--;
    }
}
```

$\mathcal{O}(n^2)$ total: one outer loop times a linear two-pointer scan. For $n = 3000$ that is $9 \times 10^6$ — fine. The duplicate-skipping lines are what make "count distinct triples" correct.

### 2.3 Closest pair sum

Same structure, but track the best instead of matching exactly:

```cpp
int i = 0, j = n - 1;
long long best = LLONG_MAX;
while (i < j) {
    long long s = a[i] + a[j];
    best = min(best, llabs(s - target));
    if (s < target) i++;
    else            j--;
}
```

That single change — compare to `best` rather than to equality — is the "closest" variant of every two-pointer problem.

---

## 3. Pattern 2 — Sort plus greedy

Some problems have a **greedy order**: sort by the right key, then take items in that order without ever reconsidering.

### 3.1 Interval scheduling

Choose the maximum number of non-overlapping intervals.

```cpp
sort(iv.begin(), iv.end(), [](const auto &x, const auto &y) {
    return x.second < y.second;         // by END, not start
});
int count = 0;
long long lastEnd = LLONG_MIN / 4;
for (auto [l, r] : iv) {
    if (l >= lastEnd) { count++; lastEnd = r; }
}
```

**Why by end?** The interval that finishes earliest leaves the most room for the rest. Sorting by start is the intuitive choice and it is wrong — a single long interval at the front destroys the answer.

*This is the clearest example in competitive programming of "the obvious sort key is the wrong one".*

### 3.2 Exchange-argument sorting

When the greedy order is not obvious, prove it by exchange: assume two adjacent items are in the "wrong" order, swap them, and show the objective does not get worse. Whatever property makes the swap safe is the sort key.

A standard example: schedule jobs with processing times $p_i$ and deadlines $d_i$ to minimise total lateness. The exchange argument yields "sort by deadline" — and the proof is two lines, but only if you write it.

| Problem | Sort key | Why |
|---|---|---|
| Max non-overlapping intervals | end time | Earliest finish leaves most room |
| Min total lateness | deadline | Exchange argument |
| Fractional knapsack | value/weight descending | Densest first |
| Min waiting time on one machine | processing time ascending | Shortest first (SPT) |
| Pair items to minimise max sum | one ascending, one descending | Pair extremes |

---

## 4. Pattern 3 — Sort plus deduplication

Sort, then walk. Equal values are adjacent, so all duplicate work becomes "compare with the previous element".

### 4.1 Count distinct

```cpp
sort(a.begin(), a.end());
int distinct = (n > 0);                    // guard the empty array
for (int i = 1; i < n; i++)
    if (a[i] != a[i-1]) distinct++;
```

$\mathcal{O}(n \log n)$, $\mathcal{O}(1)$ extra space. The alternative — a `set` or `unordered_set` — is $\mathcal{O}(n \log n)$ or $\mathcal{O}(n)$ expected but uses $\mathcal{O}(n)$ memory.

### 4.2 Deduplicate in place

```cpp
sort(a.begin(), a.end());
a.erase(unique(a.begin(), a.end()), a.end());
```

### 4.3 Coordinate compression

Turn arbitrary values into dense indices, which every array-indexed data structure needs:

```cpp
vector<int> c = a;
sort(c.begin(), c.end());
c.erase(unique(c.begin(), c.end()), c.end());

auto id = [&](int x) {
    return int(lower_bound(c.begin(), c.end(), x) - c.begin());
};
```

If `a` was `{100, 5, 100, 42}`, then `c` is `{5, 42, 100}` and the compressed array is `{2, 0, 2, 1}`. This is essential before a Fenwick tree or segment tree, and it is the standard first step of many problems with values up to $10^9$.

---

## 5. Pattern 4 — Sort plus adjacent comparison

After sorting, the answer is often the gap between neighbours.

### 5.1 Minimum absolute difference

```cpp
sort(a.begin(), a.end());
long long best = LLONG_MAX;
for (int i = 1; i < n; i++)
    best = min(best, (long long)a[i] - a[i-1]);
```

$\mathcal{O}(n \log n)$ instead of the $\mathcal{O}(n^2)$ pairwise minimum. The proof is one line: in sorted order, the closest pair must be adjacent — if `a[i] < a[j] < a[k]`, then `a[k] - a[j] < a[k] - a[i]`.

### 5.2 Duplicates

```cpp
sort(a.begin(), a.end());
bool hasDup = false;
for (int i = 1; i < n; i++)
    if (a[i] == a[i-1]) { hasDup = true; break; }
```

### 5.3 Mode (most frequent value)

```cpp
sort(a.begin(), a.end());
int bestVal = a[0], bestRun = 1;
for (int i = 0, j; i < n; i = j) {
    j = i;
    while (j < n && a[j] == a[i]) j++;      // [i, j) is one run
    if (j - i > bestRun) { bestRun = j - i; bestVal = a[i]; }
}
```

The run-length loop — `i = j` in the increment, `j` scanning forward — is the standard way to process groups after sorting.

---

## 6. Pattern 5 — Sort by a computed key

Sometimes the right order is not the natural one. Sort by a derived key, then process linearly.

```cpp
// Group by first element, keep insertion order within a group
stable_sort(v.begin(), v.end(), [](const auto &x, const auto &y) {
    return x.group < y.group;
});

// Sort by distance from a point
sort(pts.begin(), pts.end(), [&](const auto &p, const auto &q) {
    return hypot(p.x - cx, p.y - cy) < hypot(q.x - cx, q.y - cy);
});

// Sort by ratio, using integers to avoid floating point
sort(items.begin(), items.end(), [](const auto &x, const auto &y) {
    return (long long)x.value * y.weight > (long long)y.value * x.weight;
});
```

That last one deserves attention: comparing `value/weight` as a `double` introduces rounding errors that can flip the order of two nearly equal ratios, changing the greedy answer. Cross-multiplying with `long long` is exact. **Any comparator involving division should be rewritten as a multiplication.**

| Sorting by | Safe form | Unsafe form |
|---|---|---|
| Ratio $v/w$ | `x.v * y.w > y.v * x.w` | `x.v/x.w > y.v/y.w` |
| Slope | `dy1 * dx2 > dy2 * dx1` | `dy1/dx1 > dy2/dx2` |
| Distance | squared distance | `sqrt` distance |

---

## 7. Sets: order maintained live

A `set` is a sorted container with $\mathcal{O}(\log n)$ insert, erase and lookup. You reach for it when the data **changes** and you still need order.

### 7.1 The predecessor / successor pattern

This is the reason `set` exists in a contest.

```cpp
set<int> s = {1, 3, 5, 7, 9};

auto it = s.lower_bound(6);        // first element >= 6  -> 7
if (it != s.end()) cout << *it;    // successor

auto jt = s.lower_bound(6);        // find >= 6, then step back one
if (jt != s.begin()) { --jt; cout << *jt; }   // predecessor -> 5
```

The guard `it != s.begin()` before `--it` is mandatory — decrementing `begin()` is undefined behaviour.

| Query | Code |
|---|---|
| Smallest $\ge x$ | `s.lower_bound(x)` |
| Smallest $> x$ | `s.upper_bound(x)` |
| Largest $\le x$ | `auto it = s.upper_bound(x); if (it != s.begin()) --it;` |
| Largest $< x$ | `auto it = s.lower_bound(x); if (it != s.begin()) --it;` |
| Min / max | `*s.begin()` / `*s.rbegin()` |
| Exists | `s.count(x)` |

> **Chapter 1 reminder, because it is the most common TLE:** always use the **member** `s.lower_bound(x)`. `std::lower_bound(s.begin(), s.end(), x)` is $\mathcal{O}(n)$ because set iterators are bidirectional, not random access.

### 7.2 Next greater element, processed right to left

For each position, find the next position to its right with a larger value.

```cpp
vector<int> nge(n, -1);
set<int> seen;                       // values to the right, ordered

for (int i = n - 1; i >= 0; i--) {
    auto it = seen.upper_bound(a[i]);     // smallest value greater than a[i]
    if (it != seen.end()) nge[i] = *it;
    seen.insert(a[i]);
}
```

$\mathcal{O}(n \log n)$. The monotonic stack (Chapter 1, Topic 14) does the same job in $\mathcal{O}(n)$ when you only need "the nearest greater", but the `set` version generalises to "the smallest greater value anywhere to the right", which the stack cannot answer.

### 7.3 Maintaining a dynamic collection

```cpp
set<int> alive;
for (int i = 0; i < n; i++) alive.insert(i);

// remove an element and find its successor in one step
auto it = alive.find(x);
if (it != alive.end()) {
    auto nxt = next(it);              // valid before the erase
    alive.erase(it);
    if (nxt != alive.end()) { /* use nxt */ }
}
```

Capture the successor **before** erasing: after `erase(it)`, `it` is dangling.

---

## 8. Maps: keys with payloads

A `map` is a set of keys with attached values, kept in key order.

### 8.1 Frequency counting

```cpp
map<int,int> freq;
for (int x : a) freq[x]++;

for (auto &[value, count] : freq)
    cout << value << " appears " << count << " times\n";
```

Output is in ascending key order for free — often exactly what the problem asks for.

### 8.2 The `operator[]` trap

```cpp
map<string,int> mp;
if (mp["missing"] == 0) { }        // INSERTS a default entry
if (mp.count("missing") == 0) { }  // does not insert — correct for a presence test
```

`operator[]` default-constructs the value if the key is absent. In a loop over $10^6$ queries that all miss, this silently grows the map to $10^6$ entries — slower, more memory, and it breaks any subsequent iteration.

| Intent | Use |
|---|---|
| Read, inserting a default if absent | `mp[k]` |
| Read, throwing if absent | `mp.at(k)` |
| Test presence | `mp.count(k)` or `mp.find(k)` |
| Insert only if absent | `mp.emplace(k, v)` or `mp.insert({k, v})` |

### 8.3 Grouping

```cpp
map<string, vector<string>> groups;
for (const string &w : words) {
    string key = w;
    sort(key.begin(), key.end());     // canonical form: sorted letters
    groups[key].push_back(w);
}
// each entry of `groups` is one anagram class
```

Sorting the characters produces a canonical key, so strings that are anagrams map to the same entry. This "canonicalise then group" pattern solves every grouping problem.

---

## 9. The prefix-sum plus map pattern

The single most valuable pattern in this topic, and one of the most tested in Div. 2 C problems.

**Problem.** Count contiguous subarrays whose sum equals $k$.

**The idea.** Let $P[i]$ be the sum of the first $i$ elements. The sum of `a[l..r]` is $P[r+1] - P[l]$. We want $P[r+1] - P[l] = k$, i.e.

$$P[l] = P[r+1] - k$$

So walk forward maintaining the running prefix, and for each position ask: *how many earlier prefixes equalled `pref - k`?* A map of prefix counts answers that in $\mathcal{O}(1)$.

```cpp
long long countSubarrays(vector<long long> &a, long long k) {
    map<long long,int> seen;
    seen[0] = 1;                      // the empty prefix: critical
    long long pref = 0, ans = 0;

    for (long long x : a) {
        pref += x;
        ans += seen[pref - k];        // 0 if absent, and it does not insert
        seen[pref]++;
    }
    return ans;
}
```

**`seen[0] = 1` is not optional.** It accounts for subarrays that start at index 0: those have $P[l] = 0$, which is "the prefix before the array". Forget it and every answer starting at the first element is lost.

**Complexity:** $\mathcal{O}(n \log n)$ with a `map`, $\mathcal{O}(n)$ expected with `unordered_map`.

| Variant | Change |
|---|---|
| Sum equals $k$ | $P[l] = P[r] - k$ |
| Sum divisible by $k$ | count prefixes with the same remainder mod $k$ |
| Sum in $[lo, hi]$ | `count(≤ hi) − count(≤ lo−1)` |
| Longest subarray with sum $k$ | store the **first** index of each prefix |
| Equal 0s and 1s | map 0 → −1, then sum equals 0 |

*Contest story: the "count subarrays with sum $k$" idea reappears disguised as "number of subarrays with equal numbers of 0s and 1s" (replace 0 with −1), "subarrays divisible by k" (store remainders), and "subarrays with sum in a range" (two calls, subtract). People who recognise it solve all four in ten minutes; people who do not, TLE four times.*

---

## 10. Two-sum: three solutions and when each wins

| Version | Time | Space | Gives indices? |
|---|---|---|---|
| Double loop | $\mathcal{O}(n^2)$ | $\mathcal{O}(1)$ | yes |
| Sort + two pointers | $\mathcal{O}(n \log n)$ | $\mathcal{O}(1)$ | no — order destroyed |
| Map of value → index | $\mathcal{O}(n)$ expected | $\mathcal{O}(n)$ | yes |

```cpp
// Map version: indices, single pass, O(n)
unordered_map<long long,int> seen;
for (int i = 0; i < n; i++) {
    long long need = target - a[i];
    auto it = seen.find(need);
    if (it != seen.end()) { cout << it->second << ' ' << i << '\n'; break; }
    seen[a[i]] = i;
}
```

The decision rule: **if the problem asks for indices, the map wins. If it asks only for existence or values, sort plus two pointers is cheaper in memory and immune to hash collisions.**

For "pair with sum closest to target", only the sorted version works.

---

## 11. Choosing the right structure

The most common wasted time in a contest is reaching for the wrong container. Decide with this table.

| Requirement | Structure | Why |
|---|---|---|
| Data is static, many queries | Sort once + `binary_search` / `lower_bound` | $\mathcal{O}(\log n)$ with zero overhead |
| Insertions and deletions, need order | `set` / `map` | $\mathcal{O}(\log n)$ live |
| Only membership, no order | `unordered_set` | $\mathcal{O}(1)$ expected |
| Only frequency, then a full scan | Sort, then run-length | $\mathcal{O}(1)$ extra space |
| Key range is small ($\le 10^7$) | Plain array | Fastest possible |
| Need predecessor/successor | `set` | This is what it is for |
| Need min/max with removals | `set`, or `multiset` | `priority_queue` cannot delete arbitrary |
| Need the k-th smallest, dynamic | `set` + order statistics (or a Fenwick tree) | `set` alone cannot index |

### The cost comparison

| Operation | Sorted array | `set` | `unordered_set` | Array (small range) |
|---|---|---|---|---|
| Build | $\mathcal{O}(n \log n)$ | $\mathcal{O}(n \log n)$ | $\mathcal{O}(n)$ | $\mathcal{O}(V)$ |
| Query | $\mathcal{O}(\log n)$ | $\mathcal{O}(\log n)$ | $\mathcal{O}(1)$ exp. | $\mathcal{O}(1)$ |
| Insert | $\mathcal{O}(n)$ | $\mathcal{O}(\log n)$ | $\mathcal{O}(1)$ exp. | $\mathcal{O}(1)$ |
| Erase | $\mathcal{O}(n)$ | $\mathcal{O}(\log n)$ | $\mathcal{O}(1)$ exp. | $\mathcal{O}(1)$ |
| Memory/element | 4 B | ~40 B | ~32 B | 4 B |
| Ordered iteration | free | free | no | free |
| Cache behaviour | excellent | poor | moderate | excellent |

The `set`'s ~40 bytes per element is real: a `set<int>` of $10^7$ elements needs 400 MB and will MLE. A sorted `vector<int>` of the same data needs 40 MB.

> **The rule of thumb:** if the data never changes, sort a vector. If it changes, use a set. The gap is 10x in both memory and constant factor.

---

## 12. Worked examples

### 12.1 Distinct elements, three ways

```cpp
// 1. sort + unique: O(n log n), O(1) extra
vector<int> c = a;
sort(c.begin(), c.end());
c.erase(unique(c.begin(), c.end()), c.end());
int d1 = c.size();

// 2. set: O(n log n) time, O(n) memory
set<int> s(a.begin(), a.end());
int d2 = s.size();

// 3. unordered_set: O(n) expected
unordered_set<int> us(a.begin(), a.end());
int d3 = us.size();
```

All three are correct. Pick (1) when memory matters, (3) when speed matters and the values are not attacker-controlled.

### 12.2 Interval scheduling

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<pair<int,int>> iv(n);
    for (auto &[l, r] : iv) cin >> l >> r;

    sort(iv.begin(), iv.end(), [](const auto &x, const auto &y) {
        return x.second < y.second;
    });

    int count = 0;
    long long lastEnd = LLONG_MIN / 4;
    for (auto [l, r] : iv)
        if (l >= lastEnd) { count++; lastEnd = r; }

    cout << count << '\n';
    return 0;
}
```

**Input**
```text
4
1 3
2 5
4 7
6 9
```

**Output**
```text
2
```

Sorted by end: `(1,3) (2,5) (4,7) (6,9)`. Take `(1,3)`; `(2,5)` starts before 3, skip; `(4,7)` starts at 4 ≥ 3, take; `(6,9)` starts at 6 < 7, skip. Answer 2.

### 12.3 Count subarrays with sum $k$

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    map<long long,int> seen;
    seen[0] = 1;
    long long pref = 0, ans = 0;

    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        pref += x;
        ans += seen[pref - k];
        seen[pref]++;
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5 3
1 2 3 0 3
```

**Output**
```text
5
```

Prefixes: 1, 3, 6, 6, 9. Subarrays summing to 3: `[1,2]`, `[3]` at index 2, `[3,0]`, `[0,3]`, and `[3]` at index 4 — five in total. Note the two separate single-element `3`s: they are different subarrays even though they have the same sum, and a count (as opposed to a "does one exist" answer) must see both.

### 12.4 Next greater element with a set

```cpp
vector<int> nge(n, -1);
set<int> seen;
for (int i = n - 1; i >= 0; i--) {
    auto it = seen.upper_bound(a[i]);
    if (it != seen.end()) nge[i] = *it;
    seen.insert(a[i]);
}
```

Note this finds the smallest greater value to the right, not the nearest one. If the problem wants the nearest, use a monotonic stack instead.

---

## Key Takeaways

- Sorting is a **transformation**, not an operation. It buys monotonicity, adjacency of equals, adjacency of near-values, canonical order, or a greedy order.
- Sort + two pointers turns $\mathcal{O}(n^2)$ pairwise scans into $\mathcal{O}(n \log n)$ for pair sum, 3-sum and closest pair. Each step discards one index permanently.
- Greedy needs the **right** sort key, and it is often not the intuitive one: intervals sort by **end**, not start.
- Any comparator involving division should be cross-multiplied into integer arithmetic. Floating-point ratios flip order on ties.
- After sorting, the closest pair is adjacent — the $\mathcal{O}(n^2)$ pairwise minimum becomes one linear scan.
- Coordinate compression is `sort` + `unique` + `lower_bound`, and it is the mandatory first step before any array-indexed structure over large values.
- A `set` is for **live** order: insertions, deletions, and predecessor/successor queries in $\mathcal{O}(\log n)$.
- Always use the member `s.lower_bound(x)`. The free-function `std::lower_bound` on a set is $\mathcal{O}(n)$.
- Capture the successor **before** erasing an iterator; afterwards it dangles.
- `map::operator[]` inserts on a miss. Use `count`/`find` for presence tests.
- **Prefix sums + map** counts subarrays with a given sum in $\mathcal{O}(n \log n)$, and `seen[0] = 1` is mandatory. The same shape solves "divisible by $k$", "sum in a range", and "equal 0s and 1s".
- If the data never changes, sort a vector. If it changes, use a set. The difference is 10x in memory and constant factor.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Sorting by start for interval scheduling | One long early interval kills the answer | Sort by end time |
| 2 | Comparator divides (`x.v/x.w`) | Rounding flips order near ties | Cross-multiply in `long long` |
| 3 | `std::lower_bound` on a `set` | $\mathcal{O}(n)$ scan → TLE | `s.lower_bound(x)` |
| 4 | `--it` without checking `it != begin()` | UB on the first element | Guard the decrement |
| 5 | Erasing then reading `next(it)` | Iterator invalidated | Compute the successor before erasing |
| 6 | `mp[k]` used as a presence test | Inserts a default entry every miss | `mp.count(k)` |
| 7 | Forgetting `seen[0] = 1` in subarray sums | Misses every subarray starting at index 0 | Initialise the empty prefix |
| 8 | Two pointers on an unsorted array | The discard argument requires order | Sort first |
| 9 | Sorting loses the original indices | "Print the positions" answers become wrong | Store `{value, index}` |
| 10 | Using `unordered_map` with attacker-chosen keys | Anti-hash test → $\mathcal{O}(n^2)$ | Custom hash with a random seed |
| 11 | `set<int>` for $10^7$ elements | ~400 MB → MLE | Sorted vector, or compress the range |
| 12 | `multiset::erase(value)` expecting one removal | Erases **all** copies | `erase(s.find(value))` |
| 13 | Comparing pairs with `<=` in a comparator | Breaks strict weak ordering → possible crash | Strict `<` |
| 14 | Sorting inside a loop | $\mathcal{O}(n^2 \log n)$ | Sort once, reuse |
| 15 | Assuming `map` iteration is insertion order | It is key order | Use a vector of pairs if order matters |
| 16 | Integer sum overflow in prefix sums | `int` prefix of $10^5$ values near $10^9$ | `long long` |
| 17 | Deduplicating an unsorted array with `unique` | Only adjacent duplicates collapse | Sort first |
| 18 | Using `set` when a `bool` array fits | 40 bytes vs 1 byte, 10x slower | Array when the value range is small |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Pair sum, existence only | CSES "Sum of Two Values" | Easy |
| 2 | Three-sum with distinct triples | Codeforces Div. 2 C | Medium |
| 3 | Closest pair sum to a target | AtCoder ABC | Medium |
| 4 | Maximum non-overlapping intervals | CSES "Movie Festival" | Easy–Medium |
| 5 | Distinct elements, $n \le 2 \times 10^5$ | CSES "Distinct Numbers" | Easy |
| 6 | Coordinate compression then index queries | Codeforces Div. 2 C | Medium |
| 7 | Minimum absolute difference in an array | Codeforces Div. 3 B | Easy |
| 8 | Mode of an array with $10^6$ elements | AtCoder ABC | Easy–Medium |
| 9 | Count subarrays with sum $k$ | Codeforces Div. 2 C / CSES | Medium |
| 10 | Count subarrays divisible by $k$ | Codeforces Div. 2 D | Medium |
| 11 | Longest subarray with sum $k$ | Codeforces Div. 2 C | Medium |
| 12 | Subarrays with equal 0s and 1s | CSES / Codeforces | Medium |
| 13 | Next greater element, all variants | AtCoder ABC / CSES | Easy–Medium |
| 14 | Group anagrams together | Codeforces Div. 3 D | Medium |
| 15 | Dynamic ranking with insert and query | CSES "Josephus" variant / Codeforces | Hard |

---

## Next Up

→ [Topic 06 — Solving Casework Problems](Topic%2006%20-%20Casework%20Problems.md)

Sorting and mapping give you structure. Casework gives you **coverage**: splitting a problem into disjoint cases so that each one is simple enough to solve directly. It is the most common technique on Div. 2 C and the one that most rewards discipline — the cases must be exhaustive, and the boundary between them is where contests are won and lost.