# Topic 03 — Advanced Searching and Sorting Part 2

`sort(a.begin(), a.end())` sorts by `<`. That default is correct only when "smaller" is what the problem means — and in contest problems, it usually is not.

This topic makes comparison a **parameter**. Three skills follow from it:

- **Custom comparators** — sort by a rule the problem defines, including rules that are not numeric order at all.
- **Coordinate compression** — turn arbitrary values into dense indices, so a value can index an array.
- **Greedy sorting** — where the comparator *is* the algorithm, and getting it wrong means getting the answer wrong.

> **Prerequisites:** [Topic 02 — Advanced Searching and Sorting Part 1](Topic%2002%20-%20Advanced%20Searching%20and%20Sorting%20Part%201%20%28Two%20Pointers%2C%20Binary%20Search%20on%20Sorted%20and%20Unsorted%20Arrays%29.md), [Chapter 2, Topic 05](../CP%20Level%201/Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Write any comparator the statement implies, and prove it is a strict weak ordering.
> - Avoid the undefined behaviour that comes from an invalid comparator.
> - Sort by multiple keys with explicit tie-breaks.
> - Compress coordinates and explain why the compressed values are safe to use as array indices.
> - Recognise when the comparator is the whole greedy argument.

---

## 1. Sorting is a parameter

A comparator is a function answering one question: **does $a$ come before $b$?**

```cpp
sort(v.begin(), v.end(), cmp);   // cmp(a, b) == true  means a is before b
```

Everything else — the algorithm, the recursion, the swaps — is `std::sort`'s business. Which means the only thing you can get wrong is `cmp`, and you can get it wrong in two very different ways:

| Failure | Symptom |
|---|---|
| Comparator not a strict weak ordering | Undefined behaviour: crashes, or a silently wrong order |
| Comparator is valid but encodes the wrong rule | Undefined behaviour is gone; the answer is still wrong |

The first produces `Segmentation fault` or `std::sort` reading past the end. The second produces a green "Wrong answer". Both are common; only the second is dangerous.

---

## 2. The comparator contract

A comparator must be a **strict weak ordering**. In practice, four rules:

1. **Irreflexive:** `cmp(a, a)` must be `false`.
2. **Asymmetric:** if `cmp(a, b)` is true, `cmp(b, a)` must be false.
3. **Transitive:** if `cmp(a, b)` and `cmp(b, c)`, then `cmp(a, c)`.
4. **Transitive on equivalence:** if `a` and `b` are "equivalent" (neither comes first), and `b` and `c` are equivalent, then `a` and `c` are equivalent.

Rule 1 is where nearly every bug lives:

```cpp
sort(v.begin(), v.end(), [](int a, int b) { return a <= b; });   // WRONG
sort(v.begin(), v.end(), [](int a, int b) { return a < b; });    // right
```

`a <= b` makes `cmp(a, a)` true, so the sort's internal "is this element already in place?" test never terminates cleanly. In GCC's `introsort`, it typically ends with a read out of bounds. On some inputs it appears to work — which is worse.

**The rule to memorise:** a comparator answers *strictly before*, never *before or equal*. Equality is expressed by returning `false` in both directions.

### The other three rules

```cpp
// Transitivity breaker: compares distance from a fixed point inconsistently
sort(v.begin(), v.end(), [&](int a, int b) { return abs(a - pivot) < abs(b - pivot); });
```

This one is actually fine — absolute distance is a real ordering. Transitivity breaks with things like "put evens before odds, but odd values descending" mixed carelessly with a second rule. The practical test: if you cannot describe the comparator as "sort by key $K$, ties broken by key $K_2$", it probably violates something.

**If in doubt, express the comparator as a lexicographic comparison of explicit keys.** That construction is always a strict weak ordering.

---

## 3. Three ways to supply a comparator

```cpp
// 1. Lambda — the default choice for one-off sorts
sort(v.begin(), v.end(), [](const auto &a, const auto &b) {
    return a.second < b.second;
});

// 2. Function object — when the comparator needs state
struct ByDistance {
    int px, py;
    bool operator()(const pair<int,int> &a, const pair<int,int> &b) const {
        long long da = 1LL*(a.first-px)*(a.first-px) + 1LL*(a.second-py)*(a.second-py);
        long long db = 1LL*(b.first-px)*(b.first-px) + 1LL*(b.second-py)*(b.second-py);
        return da < db;
    }
};
sort(points.begin(), points.end(), ByDistance{cx, cy});

// 3. operator< — when the type has one natural order
struct Edge {
    int u, v, w;
    bool operator<(const Edge &o) const {
        if (w != o.w) return w < o.w;
        return u < o.u;
    }
};
sort(edges.begin(), edges.end());
```

| Form | Use when |
|---|---|
| Lambda | One sort, no state, obvious rule |
| Function object | The comparator is reused, or needs captured state |
| `operator<` | The type is sorted in many places the same way |

**Rule 3 must be `const`.** A non-const `operator<` fails to compile when sorting a `const` container or a container of const objects, and the error message points at `std::sort`, not at your type.

---

## 4. Sorting by multiple keys

The statement "sort by score descending, then name ascending" is a lexicographic order on the tuple `(-score, name)`. Two ways to express it:

```cpp
// Explicit: readable, obviously correct
sort(v.begin(), v.end(), [](const Student &a, const Student &b) {
    if (a.score != b.score) return a.score > b.score;   // primary: descending
    return a.name < b.name;                              // secondary: ascending
});
```

```cpp
// Packed: shorter, easy to get the signs wrong
sort(v.begin(), v.end(), [](const Student &a, const Student &b) {
    return make_tuple(-a.score, a.name) < make_tuple(-b.score, b.name);
});
```

The packed form works because `tuple`'s `<` is lexicographic. It is concise and worth knowing, but negation only works for signed integers — for `unsigned` or `long long` at the limits, negating is a bug. **Use the explicit form when descending is involved.**

**`pair` and `tuple` already have the order you want.** `pair<int,int>` sorts by first, then second. `tuple` sorts left to right. So `vector<pair<int,int>>` needs no comparator at all — and if the input order of the fields matches the priority order, that is the shortest correct solution.

---

## 5. Solved Problem 1 — Multi-Key Record Sorting

> **Statement.** Given $n$ students with a name and a score, sort them by score descending; among equal scores, by name ascending. $n \le 2 \times 10^5$.

### Why this approach

Two keys with opposite directions is the exact case where the packed `make_tuple(-score, name)` trick is tempting and where the explicit form is safer. The explicit form also documents the priority order to the next reader.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Student {
    string name;
    int score;
};

int main() {
    int n;
    cin >> n;

    vector<Student> v(n);
    for (auto &s : v) cin >> s.name >> s.score;

    sort(v.begin(), v.end(), [](const Student &a, const Student &b) {
        if (a.score != b.score) return a.score > b.score;   // primary: high score first
        return a.name < b.name;                             // secondary: name ascending
    });

    for (auto &s : v) cout << s.name << ' ' << s.score << '\n';
    return 0;
}
```

**Input**
```text
5
alice 90
bob 85
carol 90
dave 85
eve 95
```

**Output**
```text
eve 95
alice 90
carol 90
bob 85
dave 85
```

**Complexity.** $O(n \log n)$ comparisons, each $O(1)$.

### The trap

Returning `true` when both keys are equal:

```cpp
if (a.score != b.score) return a.score > b.score;
return a.name <= b.name;      // WRONG: not irreflexive
```

The output often looks correct on small tests because the equality case is rare in the data. It is undefined behaviour regardless, and it will eventually crash or reorder unpredictably.

---

## 6. The `a + b > b + a` comparator

The canonical "comparator is the problem" example.

> Arrange integers so their concatenation is the largest possible number.

"Largest first" fails: `9` and `91` → `991` beats `919`, so 9 first. But `1` and `10` → `110` beats `101`, so 1 first — and `10` and `9` → `910` beats `109`, so 9 first. The rule is not about the numbers' magnitudes; it is about their concatenations:

$$\text{place } a \text{ before } b \iff a{+}b > b{+}a$$

where `+` is string concatenation.

**Why this is a valid ordering.** Comparing $a{+}b$ with $b{+}a$ is the same as comparing the infinite repetitions $aaa\ldots$ with $bbb\ldots$. Infinite repetition is a fixed string per element, and comparing fixed strings is a total order — so transitivity holds. That is the whole proof, and it is why this comparator is safe when a naive "sort by first digit" comparator is not.

---

## 7. Solved Problem 2 — Largest Number

> **Statement.** Given $n$ non-negative integers, concatenate them in some order to form the largest possible number. Output it as a string. $n \le 10^5$, $a_i \le 10^9$.

### Why this approach

The comparator from §6 reduces the problem to a single sort. The only other obligation is the all-zeros case, which produces `"000...0"` instead of `"0"`.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<string> v(n);
    for (auto &s : v) cin >> s;

    sort(v.begin(), v.end(), [](const string &a, const string &b) {
        return a + b > b + a;        // a before b iff concatenating a first is larger
    });

    if (v[0] == "0") {               // every element is zero
        cout << 0 << '\n';
        return 0;
    }

    for (auto &s : v) cout << s;
    cout << '\n';
    return 0;
}
```

**Input**
```text
3
3 30 34
```

**Output**
```text
34330
```

The comparator orders them `34, 3, 30`. Check the alternatives: `34303` and `34330` — the second is larger, and it is the one produced.

**Complexity.** $O(n \log n)$ comparisons, but each comparison is $O(L)$ for a string of length $L$ — so $O(n L \log n)$.

### The trap

Comparing `a + b > b + a` as **integers**. With `a = 9` and `b = 91`, `a+b = 100` and `b+a = 919`, which reverses the order. Always concatenate as strings, and always read the input as strings so leading zeros survive.

---

## 8. Sorting indices

Sometimes the values must not move, only their order of examination. Sort an index array with a comparator that looks at the values:

```cpp
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j) {
    if (a[i] != a[j]) return a[i] < a[j];
    return i < j;                     // tie-break: original position
});
```

The tie-break `return i < j` makes the sort **stable by construction**, independent of `std::sort`'s instability. That is cheaper and more predictable than calling `stable_sort`.

| Need | Technique |
|---|---|
| Values in order, positions irrelevant | Sort the values |
| Positions in order, values preserved | Sort an index array |
| Stable semantics | Explicit tie-break on index, or `stable_sort` |

---

## 9. Solved Problem 3 — Sorting Indices

> **Statement.** Given an array, output the permutation of indices that sorts it ascending, with ties resolved in favour of the smaller index.

### Why this approach

The array must stay intact (later parts of the problem read it in original order), so the output is the permutation, not the sorted values. This is the "argsort" pattern.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> a(n);
    for (auto &x : a) cin >> x;

    vector<int> idx(n);
    iota(idx.begin(), idx.end(), 0);

    sort(idx.begin(), idx.end(), [&](int i, int j) {
        if (a[i] != a[j]) return a[i] < a[j];
        return i < j;                     // stable tie-break
    });

    for (int i = 0; i < n; i++) cout << idx[i] << " \n"[i + 1 == n];
    return 0;
}
```

**Input**
```text
4
40 10 20 10
```

**Output**
```text
1 3 2 0
```

Values in sorted order are `10, 10, 20, 40`, coming from indices 1, 3, 2, 0. The two 10s keep their original relative order because of the `i < j` tie-break.

**Complexity.** $O(n \log n)$ time, $O(n)$ space.

### The trap

Capturing `a` by value in the lambda (`[a]` instead of `[&a]`) copies the whole array into the comparator object, once per comparison on some compilers. At $n = 2 \times 10^5$ that is a silent TLE with correct answers. Capture by reference.

---

## 10. Coordinate compression

Values can be huge — up to $10^{18}$ — but there are only $n$ of them. Compression replaces each value by its **rank**, so that:

- the relative order is preserved,
- every rank is in $[0, n)$,
- and a rank can be used to index an array.

```cpp
vector<long long> vals = a;                        // copy
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());

for (auto &x : a)                                   // O(n log n) total
    x = lower_bound(vals.begin(), vals.end(), x) - vals.begin();
```

After this, `vals` holds the distinct values in sorted order, and each `a[i]` is the rank of the original value. `vals[a[i]]` recovers the original.

| Question | Answer after compression |
|---|---|
| How many distinct values? | `vals.size()` |
| Rank of value $x$ | `lower_bound(vals, x) - vals.begin()` |
| Value at rank $r$ | `vals[r]` |
| Number of distinct values $\le x$ | `upper_bound(vals, x) - vals.begin()` |
| Number of distinct values $< x$ | `lower_bound(vals, x) - vals.begin()` |

**Why this is safe.** Compression preserves every comparison: $x < y \iff \text{rank}(x) < \text{rank}(y)$. So any algorithm whose decisions depend only on comparisons — sorting, two pointers, binary search, most greedy algorithms — gives identical results on the compressed data.

**What it destroys.** Arithmetic. If the problem asks for a *sum* of values, compressing first is wrong. Compression is for order, position, and identity — not for magnitude.

Three standard uses:

1. **Array indexing.** A frequency table over values up to $10^9$ is impossible; over ranks it is `n` entries.
2. **2D grids and Fenwick trees.** Compress one axis, sweep the other (Chapter 6).
3. **Union–find over values.** The vertices become $0 \ldots n-1$ (Chapter 4).

---

## 11. Solved Problem 4 — Coordinate Compression in Practice

> **Statement.** Given $n$ values, output each value's rank (1-based, among distinct values, ascending). Then answer $q$ queries: for a value $x$, how many distinct values are $\le x$? $n, q \le 2 \times 10^5$, values up to $10^9$.

### Why this approach

The ranks are exactly what compression produces. The queries then become `upper_bound` calls on the sorted distinct array — $O(\log n)$ each, with no map and no hashing.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    cin >> n;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    vector<long long> vals = a;
    sort(vals.begin(), vals.end());
    vals.erase(unique(vals.begin(), vals.end()), vals.end());

    for (int i = 0; i < n; i++) {
        long long rank = lower_bound(vals.begin(), vals.end(), a[i]) - vals.begin() + 1;
        cout << rank << " \n"[i + 1 == n];
    }

    int q;
    cin >> q;
    while (q--) {
        long long x;
        cin >> x;
        // distinct values <= x
        cout << (long long)(upper_bound(vals.begin(), vals.end(), x) - vals.begin()) << '\n';
    }

    return 0;
}
```

**Input**
```text
6
100 5 100 20 5 7
3
7
50
1000
```

**Output**
```text
4 1 4 3 1 2
2
3
4
```

Distinct values sorted are `5, 7, 20, 100` — so 100 ranks 4, 5 ranks 1, 20 ranks 3, and 7 ranks 2. Of those, two are $\le 7$, three are $\le 50$, and all four are $\le 1000$.

**Complexity.** $O(n \log n + q \log n)$.

### The trap

Using `unique` without erasing:

```cpp
sort(vals.begin(), vals.end());
unique(vals.begin(), vals.end());        // WRONG: return value discarded
```

`unique` does not remove elements — it moves the distinct ones to the front and returns the new logical end. Discarding that return value leaves the duplicates in place, and `lower_bound` then returns ranks that skip nothing and mean nothing.

---

## 12. When the comparator *is* the greedy

Some greedy algorithms are "sort by the right key, then take everything that fits". The sort key is the entire proof, and choosing the wrong key turns an optimal algorithm into a plausible one.

| Problem | Correct key | Wrong key that looks right |
|---|---|---|
| Maximum non-overlapping intervals | end time ascending | start time ascending |
| Minimum machines / rooms | start time ascending, with a heap of end times | duration ascending |
| Meeting rooms, max overlap | event sweep, $-1$ before $+1$ | sorting intervals by length |
| Fractional knapsack | value/weight descending | value descending |
| Minimise total completion time | duration ascending | duration descending |
| Job sequencing | profit descending, latest free slot | deadline ascending |

Every row is a real mistake made by experienced competitors. The discipline from Chapter 2 applies unchanged: state the key, write the exchange argument, then sort.

**One practical note.** When a greedy needs a tie-break, the tie-break usually does not affect optimality — but it can affect *which* optimal answer you produce. If the problem asks for any valid answer, ignore ties. If it asks for a specific one, read the statement again.

---

## 13. Solved Problem 5 — Merge Intervals

> **Statement.** Given $n$ closed intervals $[l_i, r_i]$, merge every group that overlaps or touches, and output the resulting disjoint intervals in increasing order. $n \le 2 \times 10^5$.

### Why this approach

Merging requires processing intervals in order of their left endpoints — otherwise an interval can extend the merged region backwards. So this is a sort chosen for *correctness*, not speed: without it, no single pass works.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<long long,long long>> iv(n);
    for (auto &p : iv) cin >> p.first >> p.second;

    sort(iv.begin(), iv.end());        // by left endpoint, then right

    vector<pair<long long,long long>> res;
    for (auto &p : iv) {
        if (res.empty() || p.first > res.back().second) {
            res.push_back(p);                    // starts a new block
        } else {
            res.back().second = max(res.back().second, p.second);
        }
    }

    cout << res.size() << '\n';
    for (auto &p : res) cout << p.first << ' ' << p.second << '\n';

    return 0;
}
```

**Input**
```text
5
1 3
2 6
8 10
9 12
15 18
```

**Output**
```text
3
1 6
8 12
15 18
```

`[1,3]` and `[2,6]` overlap, giving `[1,6]`. `[8,10]` and `[9,12]` overlap, giving `[8,12]`. `[15,18]` stands alone.

**Complexity.** $O(n \log n)$ for the sort, $O(n)$ for the sweep.

### The trap

`p.first > res.back().second` versus `>=`. With closed intervals, `[1,3]` and `[3,5]` **touch** and should merge into `[1,5]`, so the correct test for "disjoint" is `>`. With half-open intervals $[l, r)$, touching intervals are disjoint and the test flips to `>=`. Merge-interval problems differ on exactly this point, and the samples rarely distinguish them — read whether the endpoints are inclusive.

The second subtlety: `res.back().second = max(res.back().second, p.second)`, not `= p.second`. Without `max`, a nested interval like `[1,10]` followed by `[2,3]` shrinks the merged block to `[1,3]`.

---

## 14. Stability, and when it matters

`std::sort` is **not stable**: equal elements may come out in any order. `std::stable_sort` preserves input order among equals.

| | `sort` | `stable_sort` |
|---|---|---|
| Time | $O(n \log n)$ | $O(n \log^2 n)$, or $O(n \log n)$ with spare memory |
| Space | $O(\log n)$ | $O(n)$ |
| Equal elements | arbitrary order | input order preserved |

**You almost never need `stable_sort`.** Adding an explicit tie-break to the comparator gives the same guarantee at `sort`'s cost:

```cpp
// Equivalent to stable_sort by value, but cheaper
sort(idx.begin(), idx.end(), [&](int i, int j) {
    if (a[i] != a[j]) return a[i] < a[j];
    return i < j;
});
```

Use `stable_sort` when the tie-break key is not available (for example, the ordering comes from an earlier pass that must not be disturbed) or when writing the tie-break would be error-prone.

---

## 15. Complexity summary

| Operation | Time | Space |
|---|---|---|
| `sort` | $O(n \log n)$ | $O(\log n)$ |
| `stable_sort` | $O(n \log^2 n)$ | $O(n)$ |
| Comparator call | $O(1)$ typically (beware string comparison: $O(L)$) | — |
| Coordinate compression | $O(n \log n)$ | $O(n)$ |
| `lower_bound` on the compressed array | $O(\log n)$ | — |
| Merge intervals | $O(n \log n)$ | $O(n)$ |
| Argsort | $O(n \log n)$ | $O(n)$ |

---

## 16. Unsolved problems

Five problems, no solutions.

| # | Problem | Constraints |
|---|---|---|
| 1 | **Sort by Frequency.** Sort the values by how often they occur, descending; ties by first occurrence in the input. | $n \le 2 \times 10^5$ |
| 2 | **Count Merged Intervals.** Given intervals, output only how many disjoint blocks remain after merging. | $n \le 2 \times 10^5$ |
| 3 | **Rank Queries.** Given $n$ values and $q$ queries, for each query $x$ output the number of values strictly less than $x$ and the number equal to $x$. | $n, q \le 2 \times 10^5$ |
| 4 | **Smallest Concatenation.** Arrange strings so their concatenation is lexicographically smallest. | $n \le 10^5$, total length $\le 10^6$ |
| 5 | **Sort by Distance.** Given $n$ points and an origin, sort them by Euclidean distance from the origin, ties by $x$ then $y$. Use no floating point. | $n \le 2 \times 10^5$, coordinates $\le 10^9$ |

**Pointers:** problem 1 needs a `map` plus a stored first-occurrence index. Problem 2 is Solved Problem 5 without the output loop. Problem 3 is two bounds on the compressed array. Problem 4 is §6 with the inequality reversed. Problem 5 must compare squared distances in `long long`, and the overflow at $10^9$ is real — the squares reach $2 \times 10^{18}$.

---

## Key Takeaways

- A comparator answers **"strictly before"**, never "before or equal". `<=` is undefined behaviour, not a style preference.
- Every comparator should be expressible as "sort by key $K_1$, ties by $K_2$" — that form is always a strict weak ordering.
- A lambda comparator capturing the array must capture **by reference**; by-value copies are a silent TLE.
- `operator<` must be `const`, or it fails to compile in const contexts with an error that blames `std::sort`.
- `pair` and `tuple` already sort lexicographically and need no comparator when the field order matches the priority.
- Descending primary keys and ascending secondary keys belong in the explicit `if/return` form, not in `make_tuple(-x, y)`.
- The `a + b > b + a` comparator is valid because it compares infinite repetitions, which is a total order.
- Concatenation comparators operate on strings, never on parsed integers — `9` and `91` prove why.
- Sort the **indices**, not the values, when positions must survive.
- An explicit index tie-break makes `sort` behave like `stable_sort` at lower cost.
- Coordinate compression preserves order and destroys arithmetic — never compress before summing.
- `unique` returns the new end; discarding it leaves duplicates in place.
- In greedy problems the comparator *is* the proof. The wrong key produces a wrong answer with no runtime error.
- Merge-interval correctness depends on the inclusive/exclusive boundary more than on anything else.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `return a <= b;` | Undefined behaviour, out-of-bounds read | `a < b` |
| 2 | Comparator capturing the array by value | $O(n)$ copy per comparison → TLE | Capture `[&]` |
| 3 | Non-const `operator<` | Fails to compile in const contexts | Mark it `const` |
| 4 | Only sorting by one key when two are needed | Order within ties is arbitrary | Explicit tie-break |
| 5 | `make_tuple(-a, b)` on unsigned values | Negation wraps | Explicit `if/return` |
| 6 | Comparing concatenations as integers | Overflow, and wrong order | Compare strings |
| 7 | `stoi`/`stoll` on input with leading zeros | Silently loses zeros | Read as strings |
| 8 | Discarding `unique`'s return value | Duplicates stay; ranks are meaningless | `v.erase(unique(...), v.end())` |
| 9 | Compressing before a sum/count by value | Arithmetic on ranks is meaningless | Compress only for order and indexing |
| 10 | `lower_bound` on an unsorted array | Undefined results | Sort first, always |
| 11 | Merge intervals with `>=` on closed intervals | Touching intervals left unmerged | `>` for closed |
| 12 | `res.back().second = p.second` in a merge | Nested intervals shrink the block | `max(...)` |
| 13 | `stable_sort` used by reflex | $O(n \log^2 n)$ for no benefit | Tie-break in the comparator |
| 14 | Sorting indices but outputting values | Wrong output for the statement | Print the permutation |
| 15 | Comparator with side effects | Order depends on call sequence; UB | Pure function only |
| 16 | String comparator at $n = 10^5$, long strings | $O(n L \log n)$ can TLE | Accept it, or hash |

---

## Practice Problems

The five unsolved problems in §16 are the practice set.

| Order | Problem | Core skill | Why this order |
|---|---|---|---|
| 1 | §16.3 Rank Queries | Compression + both bounds | Mechanical, verifies the bounds table |
| 2 | §16.2 Count Merged Intervals | Merge sweep | Solved Problem 5 minus output |
| 3 | §16.5 Sort by Distance | Integer comparator | Teaches overflow-safe comparison |
| 4 | §16.4 Smallest Concatenation | `a+b < b+a` | Same proof, reversed inequality |
| 5 | §16.1 Sort by Frequency | Two keys + first-occurrence index | Needs a map and a stored index simultaneously |

For more practice with comparators, sort problems on Codeforces tagged *sortings* and *greedy* overlap heavily with this topic.

---

## Next Up

→ [Topic 04 — Advanced Searching and Sorting Part 3](Topic%2004%20-%20Advanced%20Searching%20and%20Sorting%20Part%203%20%2820%20Solved%20Examples%20and%2010%20Unsolved%20Problems%29.md)

The techniques are in place: two pointers, binary search, comparators, compression, greedy sorts. Part 3 is twenty problems that use them in combination, where identifying *which two* techniques a statement needs is the actual difficulty.
