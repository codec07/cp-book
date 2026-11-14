# Topic 15 — STL Part 4: Essential Prebuilt Algorithms

Everything in the previous three STL topics was about *storing* data. This one is about *doing* things to it: sorting with a comparator you wrote, answering "first element $\ge x$" in $\log n$, enumerating permutations, compacting an array in place, and summing a range without an off-by-one. These are the most-used functions in the entire library. A contestant who has them at reflex level writes solutions in half the time and gets the boundary cases right the first time.

> **Prerequisites:** [Topic 12 — STL Part 1](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md), [Topic 13 — STL Part 2](Topic%2013%20-%20STL%20Part%202%20%28Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps%29.md), [Topic 14 — STL Part 3](Topic%2014%20-%20STL%20Part%203%20%28Stacks%2C%20Queue%20and%20Priority%20Queue%29.md), [Topic 11 — Functions, Classes and Lambdas](Topic%2011%20-%20Functions%2C%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md)
> **You will be able to:**
> - Sort anything — arrays, structs, pairs — with a correct strict-weak-ordering comparator.
> - Use `lower_bound`/`upper_bound` without an off-by-one, and binary search on the *answer*.
> - Compact a container in place with the erase–remove idiom and `unique`.
> - Enumerate permutations and find the $k$-th element in $\mathcal{O}(n)$.
> - Use `<numeric>` accumulators without the `accumulate` type trap.
> - Reach for the right function from a 45-row cheat sheet instead of writing a loop.

---

## 1. The iterator-range convention: `[first, last)`

Every algorithm in `<algorithm>` and `<numeric>` takes a **half-open range**: `first` is included, `last` is not. This is not a quirk — it is what makes the functions composable.

```cpp
vector<int> v = {5, 2, 9, 1, 7};

sort(v.begin(), v.end());          // [begin, end) — all five elements
sort(v.begin(), v.begin() + 3);    // [begin, begin+3) — first three only
```

Consequences you will use constantly:

| Expression | Means |
|---|---|
| `v.begin(), v.end()` | the whole container |
| `v.begin(), v.begin() + k` | the first `k` elements |
| `v.begin() + l, v.begin() + r` | indices $l$ to $r-1$ — **`r` is excluded** |
| `a, a + n` | a C array of `n` elements |
| `v.begin() + i, v.begin() + i` | empty range, legal, does nothing |

The half-open convention explains the two formulas you must memorise:

- The number of elements in `[l, r)` is `r - l`, with no `+1`. So `v.begin() + l, v.begin() + l + len` covers exactly `len` elements starting at index `l`.
- The end iterator is legal to *hold* but never to *dereference*. `*v.end()` is undefined.

**This is where most off-by-ones live.** When you write `sort(v.begin() + l, v.begin() + r)` with inclusive intent, you leave element `r` unsorted. Write it down once: `[l, r)` — `r` is excluded.

*Contest story: a problem asked to sort the subarray from index $l$ to $r$ inclusive, both 1-indexed. Half the accepted submissions read `l--, r--;` then called `sort(a+l, a+r)` — and half of those failed, because the second argument must be `a+r+1`. The fix is one character and the reasoning is the half-open range.*

---

## 2. `sort` in depth

```cpp
sort(v.begin(), v.end());                       // ascending, O(n log n)
sort(a, a + n);                                 // C array
sort(v.begin(), v.end(), greater<int>());       // descending
sort(v.rbegin(), v.rend());                     // descending, same effect
```

`std::sort` is **introsort**: quicksort with a median-of-three pivot, falling back to heapsort when recursion gets too deep and to insertion sort for tiny ranges. Result: $\mathcal{O}(n \log n)$ worst case, $\mathcal{O}(1)$ extra space, and cache-friendly. It is **not stable** — equal elements may be reordered.

| Function | Time | Space | Stable | Use |
|---|---|---|---|---|
| `sort` | $\mathcal{O}(n\log n)$ | $\mathcal{O}(\log n)$ | no | default |
| `stable_sort` | $\mathcal{O}(n\log n)$ (or $n\log^2 n$) | $\mathcal{O}(n)$ | yes | multi-key sorts |
| `partial_sort` | $\mathcal{O}(n\log k)$ | $\mathcal{O}(1)$ | no | top-$k$ in order |
| `nth_element` | $\mathcal{O}(n)$ average | $\mathcal{O}(1)$ | no | $k$-th smallest, median |
| `is_sorted` | $\mathcal{O}(n)$ | $\mathcal{O}(1)$ | — | validation |

### Sorting pairs and tuples

`pair` and `tuple` already have `operator<`, comparing lexicographically:

```cpp
vector<pair<int,int>> vp = {{2, 5}, {1, 9}, {2, 3}, {1, 2}};
sort(vp.begin(), vp.end());
// (1,2) (1,9) (2,3) (2,5)
```

For `pair<first, second>` this means "by first, then by second". This is why the trick `v.push_back({value, index})` works so well: sorting gives you the values in order, ties broken by index. For "sort by second, then first", a comparator is required.

### Sorting a vector of structs

```cpp
struct Player {
    string name;
    int score, penalty;
};

vector<Player> ps;

sort(ps.begin(), ps.end(), [](const Player &a, const Player &b) {
    if (a.score != b.score) return a.score > b.score;   // higher score first
    return a.penalty < b.penalty;                        // fewer penalties first
});
```

The `if (a.key != b.key) return ...` shape is the canonical multi-key comparator. Get the `!=` right: using `if (a.key > b.key) return true; return a.penalty < b.penalty;` is also correct but subtly harder to read.

### Custom comparators: three ways

```cpp
// 1. Function
bool cmp(const pair<int,int> &a, const pair<int,int> &b) { return a.second > b.second; }
sort(v.begin(), v.end(), cmp);

// 2. Lambda — the CP default
sort(v.begin(), v.end(), [](int a, int b) { return abs(a) < abs(b); });

// 3. Functor struct — needed for priority_queue and set
struct BySecond {
    bool operator()(const pair<int,int> &a, const pair<int,int> &b) const {
        return a.second > b.second;
    }
};
sort(v.begin(), v.end(), BySecond{});
```

Lambdas with captures also work — the capture is stored in the closure:

```cpp
int pivot = 100;
sort(v.begin(), v.end(), [pivot](int a, int b) {
    return abs(a - pivot) < abs(b - pivot);      // closest to pivot first
});
```

Capturing by reference inside a comparator is legal but dangerous if anything mutates during the sort. Don't.

---

## 3. The comparator contract: strict weak ordering

The comparator must behave like `<`. Concretely, for any `a`, `b`, `c`:

1. **Irreflexive:** `cmp(a, a)` is `false`. No element is less than itself.
2. **Asymmetric:** if `cmp(a, b)` then `!cmp(b, a)`.
3. **Transitive:** if `cmp(a, b)` and `cmp(b, c)` then `cmp(a, c)`.
4. **Transitive incomparability:** if `a` and `b` are equivalent and `b` and `c` are equivalent, then `a` and `c` are equivalent.

The violation that destroys contests is using `<=` where `<` belongs:

```cpp
sort(v.begin(), v.end(), [](int a, int b) { return a <= b; });   // BROKEN
```

With `<=`, `cmp(a, a)` is `true`, so introsort's partition logic can walk off the end of the array. Symptoms range from a wrong answer to a `SIGSEGV` inside `std::sort` — with no compiler warning, because the signature is valid.

```text
$ ./a.out
Segmentation fault (core dumped)
```

The rules of the road:

- **Never use `<=` or `>=` in a comparator.** Always `<` or `>`.
- **Never let the comparator depend on mutable global state** that changes during the sort.
- **Never compare with `==` and return `true` for equal elements.** Equal elements must compare `false` in both directions.
- **Return `false` for equal elements.** Sorting many equal keys with `<=` is the most likely trigger for a crash.

A comparator that compares only part of the key is fine — that is what "equivalent" means, and it is why `stable_sort` exists to break the tie deterministically.

```cpp
// Fine: all elements with the same first element are "equivalent"
sort(v.begin(), v.end(), [](auto &a, auto &b) { return a.x < b.x; });
```

---

## 4. `stable_sort`, `partial_sort`, `nth_element`, `is_sorted`

### `stable_sort`

Preserves the relative order of elements that compare equivalent. It allocates a buffer, so it is slower and uses $\mathcal{O}(n)$ memory. Use it when you have already sorted by one key and want to re-sort by another without destroying the first ordering:

```cpp
sort(ps.begin(), ps.end(), byName);         // tie-break: name
stable_sort(ps.begin(), ps.end(), byScore); // ties keep name order
```

The alternative — a comparator that checks both keys — is usually faster and clearer. `stable_sort` earns its place when the secondary key is expensive or implicit.

### `partial_sort`

Produces the first $k$ elements in sorted order and leaves the rest unspecified. $\mathcal{O}(n \log k)$, which beats a full sort when $k \ll n$:

```cpp
partial_sort(v.begin(), v.begin() + k, v.end());        // k smallest, ascending, in order
// k largest, descending:
partial_sort(v.begin(), v.begin() + k, v.end(), greater<int>());
```

For $k = 3$ out of $10^7$, this is roughly $10^7 \cdot \log 3$ comparisons instead of $10^7 \cdot 23$.

### `nth_element`

Rearranges so that the element at position `n` is the one that *would* be there after a full sort, everything before is `<=` it, and everything after is `>=`. Average $\mathcal{O}(n)$, no allocation:

```cpp
vector<int> v = {7, 2, 9, 4, 1, 8, 3};

nth_element(v.begin(), v.begin() + 3, v.end());   // v[3] is now the 4th smallest
cout << v[3] << '\n';                              // 4

// median of an odd-length array
int mid = v.size() / 2;
nth_element(v.begin(), v.begin() + mid, v.end());
cout << v[mid] << '\n';
```

Use `nth_element` whenever you need one order statistic and not the full order. It is the difference between $\mathcal{O}(n)$ and $\mathcal{O}(n \log n)$ on $10^6$ elements — 10 ms versus 120 ms.

Note the middle argument: it is the *position* you want correct, expressed as `begin() + k` for the 0-indexed $k$-th smallest.

### `is_sorted` and `is_sorted_until`

```cpp
cout << is_sorted(v.begin(), v.end()) << '\n';           // 1 or 0
auto it = is_sorted_until(v.begin(), v.end());           // first out-of-order position
if (it != v.end()) cout << "breaks at index " << (it - v.begin()) << '\n';
```

Useful for assertions and for problems that ask for the first violation.

---

## 5. Binary search: the `lower_bound` family

Setup for the whole section:

```cpp
vector<int> a = {1, 3, 3, 5, 7};
//                  0  1  2  3  4
```

| Function | Returns |
|---|---|
| `lower_bound(f, l, x)` | iterator to the **first** element $\ge x$ |
| `upper_bound(f, l, x)` | iterator to the **first** element $> x$ |
| `equal_range(f, l, x)` | `pair(lower_bound, upper_bound)` |
| `binary_search(f, l, x)` | `bool` — does $x$ exist |

| $x$ | `lower_bound` index | `upper_bound` index | count of $x$ |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 1 | 0 | 1 | 1 |
| 3 | 1 | 3 | 2 |
| 4 | 3 | 3 | 0 |
| 5 | 3 | 4 | 1 |
| 7 | 4 | 5 | 1 |
| 8 | 5 (`end()`) | 5 (`end()`) | 0 |

Four facts that make this table usable:

1. **`upper_bound - lower_bound` is exactly the count of $x$.** This is the whole reason both exist.
2. **`lower_bound` never fails.** If every element is smaller than $x$, it returns `end()`. Always convert to an index and check against `n` before dereferencing.
3. **The range must be sorted** by the same ordering the comparator uses. On unsorted data the result is a plausible-looking wrong answer, not an error.
4. **`binary_search` tells you *whether*, not *where*.** If you need the position, use `lower_bound`.

```cpp
int idx = lower_bound(a.begin(), a.end(), 3) - a.begin();   // 1
if (idx < (int)a.size() && a[idx] == 3) { /* found at idx */ }

// count of x
int cnt = upper_bound(a.begin(), a.end(), 3) - lower_bound(a.begin(), a.end(), 3);  // 2

// largest element < x
auto it = lower_bound(a.begin(), a.end(), 4);
if (it != a.begin()) { --it; /* *it == 3 */ }

// smallest element > x
auto jt = upper_bound(a.begin(), a.end(), 3);              // *jt == 5
```

That "largest element `< x`" idiom — `lower_bound` then step back — is the single most-used pattern in the family, and forgetting the `it != begin()` guard before `--it` is undefined behaviour on the front of the array.

### On a C array

```cpp
int a[5] = {1, 3, 3, 5, 7};
int *p = lower_bound(a, a + 5, 3);
cout << (p - a) << '\n';          // 1
```

### With a custom comparator

```cpp
struct Item { int key, id; };
vector<Item> items;                       // sorted by key

auto it = lower_bound(items.begin(), items.end(), 42,
                      [](const Item &e, int value) { return e.key < value; });
```

The comparator takes `(element, value)` in that order. This is the one place where the argument order trips people up — `lower_bound` calls `cmp(*it, value)`, never the reverse.

### On a `set` or `map`

**Use the member function, not the algorithm.** `std::lower_bound` on a `set` is $\mathcal{O}(n)$ because set iterators are bidirectional, not random access — the algorithm advances one step at a time. The member version is $\mathcal{O}(\log n)$ because it descends the tree.

```cpp
set<int> s = {1, 3, 5, 7};
auto it = s.lower_bound(3);       // O(log n) — correct
auto jt = std::lower_bound(s.begin(), s.end(), 3);   // O(n) — a classic TLE
```

$7 \times 10^5$ queries on a $10^5$-element set: member version ~0.05 s, algorithm version ~15 s. See [Topic 13](Topic%2013%20-%20STL%20Part%202%20%28Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps%29.md) for the full discussion.

---

## 6. Binary search on the answer

The most valuable idea in this file. When the answer is a number with a monotone predicate — "can we achieve $X$?" is true for all $X \le \text{ans}$ and false above — binary search the *answer*, not the array.

Two templates cover everything. Learn both shapes and never improvise.

### Template A — find the first `x` where `ok(x)` becomes true

`ok` is monotone: false, false, …, false, **true**, true, … true.

```cpp
long long lo = 0, hi = 1000000000LL;      // invariant: ok(hi) is true (or hi is the answer bound)
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    if (ok(mid)) hi = mid;                // mid works — try smaller
    else         lo = mid + 1;            // mid fails — must be larger
}
// answer is lo
```

`mid = lo + (hi - lo) / 2` instead of `(lo + hi) / 2` avoids overflow when both are near $10^{18}$.

### Template B — find the last `x` where `ok(x)` is true

`ok` is monotone: true, true, …, true, **false**, false, … false.

```cpp
long long lo = 0, hi = 1000000000LL;
while (lo < hi) {
    long long mid = lo + (hi - lo + 1) / 2;   // UPPER mid — the crucial difference
    if (ok(mid)) lo = mid;                   // mid works — try larger
    else         hi = mid - 1;               // mid fails — must be smaller
}
// answer is lo
```

**The `+1` in the midpoint is the whole template.** Without it, when `hi == lo + 1` you compute `mid == lo`, take the `lo = mid` branch, and loop forever.

| Symptom | Cause |
|---|---|
| Infinite loop | Template B with `(lo + hi) / 2` instead of `(lo + hi + 1) / 2` |
| Off by one on the answer | Used Template A for a "last true" predicate |
| `mid` overflows negative | `(lo + hi) / 2` with values near $2^{63}$ |
| Wrong on `hi` boundary | Invariant `ok(hi)` is true was never established |

Worked example — "minimum capacity to ship all packages within $D$ days":

```cpp
auto feasible = [&](long long cap) {
    long long days = 1, load = 0;
    for (int w : weights) {
        if (w > cap) return false;          // a single package exceeds capacity
        if (load + w > cap) { days++; load = 0; }
        load += w;
    }
    return days <= D;
};

long long lo = *max_element(weights.begin(), weights.end());
long long hi = accumulate(weights.begin(), weights.end(), 0LL);
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    if (feasible(mid)) hi = mid;
    else               lo = mid + 1;
}
cout << lo << '\n';
```

Note `lo` starting at `max` (not 0) — the lower bound of a valid answer is itself part of the reasoning. Getting `lo` right is half of getting the template right.

### Binary search on doubles

There is no "first true" for a continuum, so iterate a fixed number of times instead of testing convergence:

```cpp
double lo = 0, hi = 1e9;
for (int iter = 0; iter < 100; iter++) {      // 100 halvings is far below double precision
    double mid = (lo + hi) / 2;
    if (feasible(mid)) hi = mid;
    else               lo = mid;
}
cout << fixed << setprecision(9) << hi << '\n';
```

100 iterations shrinks the interval by $2^{-100} \approx 10^{-30}$, well past what `double` can represent. Never use `while (hi - lo > 1e-9)` — with large values that condition may never become true, because `hi - lo` bottoms out at representable precision.

---

## 7. The min/max family

```cpp
int a = 4, b = 9;
cout << min(a, b) << ' ' << max(a, b) << '\n';        // 4 9
cout << min({3, 1, 4, 1, 5}) << ' ' << max({3,1,4}) << '\n';   // 1 4

auto [lo, hi] = minmax(a, b);                          // pair of const refs
cout << lo << ' ' << hi << '\n';                       // 4 9

cout << clamp(15, 0, 10) << '\n';                      // 10 — C++17
```

`min`, `max` and `clamp` return **references** (`const T&`), so `min({a,b,c})` bound to a reference dangles — copy the result. `clamp` is undefined if `lo > hi`.

### `min_element`, `max_element`, `minmax_element`

These return **iterators**, not values. Forgetting the `*` is a compile error, which is the good outcome:

```cpp
vector<int> v = {4, 1, 9, 3};

cout << *max_element(v.begin(), v.end()) << '\n';                   // 9
cout << *min_element(v.begin(), v.end()) << '\n';                   // 1

int imax = max_element(v.begin(), v.end()) - v.begin();              // index of max
cout << imax << '\n';                                                // 2

auto [mn, mx] = minmax_element(v.begin(), v.end());
cout << *mn << ' ' << *mx << '\n';                                   // 1 9
```

`minmax_element` is not just two calls: it does $3n/2$ comparisons instead of $2n$.

**Tie-breaking:** `max_element` returns the *first* maximum; use `std::max_element` with a reversed comparison if you need the last one. Same for `min_element`. Problems that ask "print the leftmost index" are testing exactly this.

An empty range makes `max_element` return `end()` — dereferencing it is undefined behaviour. Guard with `if (v.empty())`.

---

## 8. Array manipulation

### `reverse`

```cpp
reverse(v.begin(), v.end());
reverse(a, a + n);
reverse(v.begin() + l, v.begin() + r);      // reverses [l, r)
```

$\mathcal{O}(n)$, in place, `swap`s elements. Reversing a `string` is the same call on `s.begin()`, `s.end()`.

### `rotate`

Rotates left so that `middle` becomes the new first element:

```cpp
vector<int> v = {1, 2, 3, 4, 5};
rotate(v.begin(), v.begin() + 2, v.end());
// v = {3, 4, 5, 1, 2}  — rotated left by 2
```

```cpp
int k = 2;
k %= v.size();                                        // k may exceed n
rotate(v.begin(), v.begin() + k, v.end());            // left by k
rotate(v.begin(), v.end() - k, v.end());              // right by k
```

$\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space — the three-reversal algorithm under the hood. `k %= n` is mandatory: `v.begin() + k` with `k > n` is undefined behaviour, and with `k == n` it is a no-op.

### `fill`, `fill_n`, `generate`, `iota`

```cpp
fill(v.begin(), v.end(), 7);           // all 7
fill_n(v.begin(), 3, 7);               // first 3 only

generate(v.begin(), v.end(), [n = 0]() mutable { return n++; });   // 0,1,2,...

vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);       // 0,1,2,...,n-1
```

`iota` is the clean way to build an index array for "sort the indices, not the values":

```cpp
vector<int> order(n);
iota(order.begin(), order.end(), 0);
sort(order.begin(), order.end(), [&](int i, int j) { return a[i] < a[j]; });
// order[0] is the index of the smallest element
```

This is the standard technique when you need to sort by a key but keep original positions.

### `remove`, `remove_if`, and the erase–remove idiom

`remove` **does not remove anything**. It shifts the kept elements to the front and returns an iterator to the new logical end, leaving the tail in a valid-but-unspecified state. To actually shrink the container, you erase the tail:

```cpp
vector<int> v = {1, 2, 3, 2, 4, 2, 5};

// remove every 2
v.erase(remove(v.begin(), v.end(), 2), v.end());
// v = {1, 3, 4, 5}

// remove everything that fails a predicate
v.erase(remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }), v.end());
```

Why the two-step design? Because `remove` works on any forward range, including one that cannot be resized, and because it separates "partition" from "deallocate" — which is exactly what lets it work on C arrays too (`remove(a, a+n, 2)` returns the new end pointer).

**This is $\mathcal{O}(n)$ total.** The naive `for (i...) if (bad) v.erase(v.begin()+i)` is $\mathcal{O}(n^2)$ because each erase shifts the tail, and it also *skips* elements: after erasing index `i`, the element that was at `i+1` is now at `i`, and `i++` jumps over it. See [Topic 12](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md).

For a `list`, use the member functions `lst.remove(x)` and `lst.remove_if(pred)` — same name, but they really do erase.

### `unique`

Collapses **adjacent** duplicates and returns the new logical end. It is `remove`-style: it does not shrink, and it requires the range sorted (or at least grouped) for global deduplication.

```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());       // the dedupe pair
```

| Input | After `unique` |
|---|---|
| `{1,1,2,2,2,3}` | `{1,2,3}` |
| `{1,2,1,2}` | `{1,2,1,2}` — no adjacent dups |
| `{5,5,5}` | `{5}` |

The combination `sort` + `unique` + `erase` is the standard coordinate-compression step:

```cpp
vector<int> coord = a;                              // copy
sort(coord.begin(), coord.end());
coord.erase(unique(coord.begin(), coord.end()), coord.end());

// map any value to its compressed index
int id = lower_bound(coord.begin(), coord.end(), x) - coord.begin();
```

### `replace`, `transform`, `copy`, `copy_if`, `move`

```cpp
replace(v.begin(), v.end(), 0, -1);                 // every 0 becomes -1

transform(v.begin(), v.end(), v.begin(), [](int x) { return x * x; });   // in place
transform(a.begin(), a.end(), b.begin(), out.begin(), plus<int>());      // a+b elementwise

copy(src.begin(), src.end(), dst.begin());
copy_if(src.begin(), src.end(), back_inserter(dst), [](int x) { return x > 0; });
```

`back_inserter` from `<iterator>` is what lets `copy_if` append to a growing vector. Without it, `copy_if` writes into pre-sized space and you must `resize` first.

---

## 9. Queries: `count`, `find`, and the predicates

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

count(v.begin(), v.end(), 1);                              // 2
count_if(v.begin(), v.end(), [](int x) { return x > 4; }); // 3

find(v.begin(), v.end(), 4);                               // iterator, or end()
find_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
find_if_not(v.begin(), v.end(), isdigit);

any_of(v.begin(), v.end(), [](int x) { return x > 8; });    // true
all_of(v.begin(), v.end(), [](int x) { return x > 0; });    // true
none_of(v.begin(), v.end(), [](int x) { return x < 0; });   // true

int s = accumulate(v.begin(), v.end(), 0, [](int a, int b) { return a ^ b; });  // XOR
```

| Function | Returns | On not-found |
|---|---|---|
| `find` | iterator | `end()` |
| `find_if` | iterator | `end()` |
| `count` | `ptrdiff_t` | `0` |
| `count_if` | `ptrdiff_t` | `0` |
| `any_of` | `bool` | `false` |
| `all_of` | `bool` | `true` for an empty range |
| `none_of` | `bool` | `true` for an empty range |

The `all_of`/`none_of` rows are not typos — vacuous truth. An empty range satisfies every universal statement.

**Always compare a `find` result before dereferencing:**

```cpp
auto it = find(v.begin(), v.end(), 4);
if (it != v.end()) cout << *it << '\n';       // *it is UB when it == end()
```

### `mismatch`, `search`, `adjacent_find`

```cpp
// first position where two ranges differ
auto [i1, i2] = mismatch(a.begin(), a.end(), b.begin());

// first occurrence of a subsequence
auto it = search(text.begin(), text.end(), pat.begin(), pat.end());

// first adjacent duplicate
auto jt = adjacent_find(v.begin(), v.end());      // pair of equal neighbours
```

`adjacent_find` on a sorted range detects duplicates in $\mathcal{O}(n)$ without a hash set.

---

## 10. Permutations

`next_permutation` transforms a range into the next permutation in **lexicographic** order and returns `false` when it wraps around to the sorted order.

```cpp
vector<int> v = {1, 2, 3};
sort(v.begin(), v.end());              // MUST start sorted
do {
    for (int x : v) cout << x << ' ';
    cout << '\n';
} while (next_permutation(v.begin(), v.end()));
```

```text
1 2 3
1 3 2
2 1 3
2 3 1
3 1 2
3 2 1
```

Three facts:

1. **The range must start sorted**, or you enumerate only the permutations lexicographically after the current one. The `sort` before the loop is not optional.
2. **Total cost is $\mathcal{O}(n \cdot n!)$** — every permutation is visited and each costs $\mathcal{O}(n)$. This is viable only up to $n \approx 10$, which is exactly the constraint you see on brute-force problems.
3. **Duplicates are handled correctly**: `{1,1,2}` yields 3 distinct permutations, not 6.

`prev_permutation` walks backwards; start from a descending sort.

### The $k$-th permutation in $\mathcal{O}(n^2)$

```cpp
vector<int> kthPermutation(int n, long long k) {      // k is 1-indexed
    vector<int> pool(n), res;
    iota(pool.begin(), pool.end(), 1);                // 1..n
    long long fact = 1;
    for (int i = 2; i <= n; i++) fact *= i;           // n!
    k--;                                              // 0-indexed

    for (int i = n; i >= 1; i--) {
        fact /= i;                                    // (i-1)!
        int idx = k / fact;
        k %= fact;
        res.push_back(pool[idx]);
        pool.erase(pool.begin() + idx);
    }
    return res;
}
```

The idea: the first element is determined by `k / (n-1)!`, then recurse. This is the factorial number system. With $n \le 20$ and `long long`, `fact` fits up to $20! \approx 2.4 \times 10^{18}$.

---

## 11. `<numeric>`: accumulators and math

### `accumulate` and the type trap

```cpp
vector<int> v = {1, 2, 3, 4, 5};

int s1 = accumulate(v.begin(), v.end(), 0);          // int accumulator
long long s2 = accumulate(v.begin(), v.end(), 0LL);  // long long accumulator — correct
```

**The accumulator type is the type of the initial value, not the element type.** For $10^6$ elements near $10^9$, the sum is $\approx 10^{15}$: the first line overflows `int` silently and returns garbage, while `std::accumulate`'s signature happily accepts the assignment to a `long long` afterwards. The overflow has already happened.

Rules:

- **Always pass `0LL`** for integer sums that can exceed $2^{31}$.
- For floating point, `0.0` (or `0.0L` for `long double`).
- For a custom fold, the lambda's return type is the accumulator type:

```cpp
long long prod = accumulate(v.begin(), v.end(), 1LL,
                            [](long long a, int b) { return a * b; });
```

### `partial_sum` — prefix sums

```cpp
vector<long long> P(n + 1, 0);
partial_sum(a.begin(), a.end(), P.begin() + 1);      // P[i+1] = a[0] + ... + a[i]
// P[0] stays 0, so P[r+1] - P[l] is the sum of a[l..r]
```

`partial_sum` writes the running total into the output range, so the output must be at least as long as the input. Offsetting the destination by one gives you the $\mathcal{O}(n+1)$ prefix array with the leading zero, which is the form that makes range queries uniform. See [Topic 09](Topic%2009%20-%20Arrays%20%281D,%202D%20and%203D%29.md).

### `adjacent_difference`

```cpp
vector<int> d(n);
adjacent_difference(a.begin(), a.end(), d.begin());
// d[0] = a[0]; d[i] = a[i] - a[i-1] for i >= 1
```

This is the inverse of `partial_sum` and the natural way to build a difference array for range updates.

### `inner_product`

```cpp
long long dot = inner_product(a.begin(), a.end(), b.begin(), 0LL);
// a[0]*b[0] + a[1]*b[1] + ...
```

$\mathcal{O}(n)$ dot product — the base case of matrix multiplication, and occasionally the whole answer.

### `gcd`, `lcm`, `reduce`, `scan`

```cpp
int g = gcd(12, 18);                  // 6 — <numeric>, C++17
int l = lcm(4, 6);                    // 12

// reduce: like accumulate but may reassociate (parallel-friendly)
long long total = reduce(v.begin(), v.end(), 0LL);

// exclusive_scan: like partial_sum, with an explicit init
vector<long long> P(n + 1);
exclusive_scan(a.begin(), a.end(), P.begin(), 0LL);
// P[0] = 0, P[i] = a[0] + ... + a[i-1]  — exactly the prefix-sum form
```

`exclusive_scan` is the cleanest way to build a 1-indexed prefix array. It is C++17 and available on every judge worth using.

**`lcm` overflow:** `lcm(a,b)` internally computes `a / gcd * b`, so it is safe up to the destination type — but assigning the result to an `int` when it exceeds $2^{31}$ truncates. Use `long long lcm(long long, long long)` overloads by passing `long long` arguments.

---

## 12. The `__builtin_*` family

GCC/Clang intrinsics that compile to single instructions. Available on every Codeforces, AtCoder and CSES judge (all g++).

```cpp
__builtin_popcount(x);        // number of set bits, unsigned int
__builtin_popcountll(x);      // same for unsigned long long — use this for 64-bit
__builtin_clz(x);             // leading zeros; UB when x == 0
__builtin_ctz(x);             // trailing zeros; UB when x == 0
__builtin_clzll(x);
__builtin_parity(x);          // 1 if the popcount is odd
__builtin_ffs(x);             // 1 + index of the lowest set bit; 0 if x == 0
```

| Builtin | Meaning | $\mathcal{O}$ | Trap |
|---|---|---|---|
| `popcount` | set bits | 1 instruction | `int` truncates 64-bit input |
| `clz` | leading zeros in 32 bits | 1 | UB on 0 |
| `ctz` | trailing zeros | 1 | UB on 0 |
| `parity` | popcount mod 2 | 1 | — |
| `ffs` | 1 + lowest set bit index | 1 | returns 0 on 0, not -1 |

The useful derived quantities:

```cpp
int highestBit(long long x) { return 63 - __builtin_clzll(x); }   // x > 0
bool isPowerOfTwo(long long x) { return x > 0 && (x & (x-1)) == 0; }
int lowestBitValue(long long x) { return x & -x; }                 // x > 0
```

**`clz(0)` and `ctz(0)` are undefined behaviour** — on x86 they return garbage (the instruction is undefined), not 32. Guard:

```cpp
int bits = x ? 32 - __builtin_clz(x) : 0;
```

---

## 13. The cheat sheet

Sorted roughly by how often you will use them.

| Function | Header | Complexity | Purpose |
|---|---|---|---|
| `sort(f,l)` | algorithm | $n\log n$ | ascending sort |
| `sort(f,l,cmp)` | algorithm | $n\log n$ | custom order |
| `stable_sort` | algorithm | $n\log n$ | keeps equal-element order |
| `partial_sort(f,m,l)` | algorithm | $n\log k$ | first $k$ in order |
| `nth_element(f,f+k,l)` | algorithm | $n$ avg | $k$-th smallest in place |
| `is_sorted` | algorithm | $n$ | ascending check |
| `lower_bound(f,l,x)` | algorithm | $\log n$ | first $\ge x$ |
| `upper_bound(f,l,x)` | algorithm | $\log n$ | first $> x$ |
| `equal_range` | algorithm | $\log n$ | both bounds at once |
| `binary_search` | algorithm | $\log n$ | existence only |
| `min` / `max` | algorithm | 1 | two values, same type |
| `minmax` | algorithm | 1 | both at once |
| `clamp(x,lo,hi)` | algorithm | 1 | restrict to a range |
| `min_element` | algorithm | $n$ | iterator to smallest |
| `max_element` | algorithm | $n$ | iterator to largest |
| `minmax_element` | algorithm | $1.5n$ | both, efficiently |
| `reverse` | algorithm | $n$ | in place |
| `rotate(f,m,l)` | algorithm | $n$ | left-rotate to `m` |
| `fill` / `fill_n` | algorithm | $n$ | assign a value |
| `iota` | numeric | $n$ | 0,1,2,… |
| `generate` | algorithm | $n$ | fill from a callable |
| `copy` / `copy_if` | algorithm | $n$ | duplicate a range |
| `move` | algorithm | $n$ | move a range |
| `remove` / `remove_if` | algorithm | $n$ | compact, does not erase |
| `unique` | algorithm | $n$ | collapse adjacent dups |
| `replace` | algorithm | $n$ | substitute values |
| `transform` | algorithm | $n$ | map a function over a range |
| `for_each` | algorithm | $n$ | apply for side effects |
| `count` / `count_if` | algorithm | $n$ | count matches |
| `find` / `find_if` | algorithm | $n$ | first match |
| `any_of` / `all_of` / `none_of` | algorithm | $n$ | predicate over range |
| `mismatch` | algorithm | $n$ | first difference |
| `search` | algorithm | $nm$ | subsequence search |
| `adjacent_find` | algorithm | $n$ | first adjacent duplicate |
| `next_permutation` | algorithm | $n$ | next lexicographic order |
| `prev_permutation` | algorithm | $n$ | previous order |
| `accumulate` | numeric | $n$ | fold / sum |
| `reduce` | numeric | $n$ | reassociable fold |
| `partial_sum` | numeric | $n$ | prefix sums |
| `exclusive_scan` | numeric | $n$ | prefix sums with init |
| `adjacent_difference` | numeric | $n$ | differences |
| `inner_product` | numeric | $n$ | dot product |
| `gcd` / `lcm` | numeric | $\log$ | number theory |
| `__builtin_popcountll` | — | 1 | set-bit count |
| `__builtin_clz` / `ctz` | — | 1 | bit scans (not on 0) |

---

## 14. CP recipes

### 14.1 Sort + unique for coordinate compression

```cpp
vector<int> c = a;
sort(c.begin(), c.end());
c.erase(unique(c.begin(), c.end()), c.end());

auto id = [&](int x) { return lower_bound(c.begin(), c.end(), x) - c.begin(); };
// every value is now in [0, c.size())
```

Turns arbitrary values into dense indices, which is what Fenwick trees and segment trees need. See Chapter 3.

### 14.2 Two-pointer pair sum on a sorted array

```cpp
sort(a.begin(), a.end());
int i = 0, j = n - 1;
bool found = false;
while (i < j) {
    long long s = (long long)a[i] + a[j];
    if (s == target) { found = true; break; }
    if (s < target) i++; else j--;
}
```

$\mathcal{O}(n \log n)$ including the sort — the two-pointer scan itself is $\mathcal{O}(n)$. Works only because the array is sorted.

### 14.3 Count pairs with difference $\le k$

```cpp
sort(a.begin(), a.end());
long long cnt = 0;
for (int r = 0; r < n; r++) {
    int l = lower_bound(a.begin(), a.end(), a[r] - k) - a.begin();
    cnt += r - l;                     // partners strictly before r
}
```

$\mathcal{O}(n \log n)$. The `r - l` counts pairs $(l, r)$ with $l < r$, which avoids double counting the same as counting $(r-l)$.

### 14.4 $k$-th smallest

```cpp
nth_element(a.begin(), a.begin() + k, a.end());
cout << a[k] << '\n';                 // 0-indexed k-th smallest
```

$\mathcal{O}(n)$ instead of $\mathcal{O}(n \log n)$. Do not use `sort` when one order statistic is all you need.

### 14.5 LIS in $\mathcal{O}(n \log n)$

```cpp
vector<int> tails;                     // tails[len] = smallest tail of an increasing subsequence of length len+1
for (int x : a) {
    auto it = lower_bound(tails.begin(), tails.end(), x);   // strict: <
    if (it == tails.end()) tails.push_back(x);
    else                   *it = x;
}
cout << tails.size() << '\n';
```

`lower_bound` for strictly increasing, `upper_bound` for non-decreasing. That one-character choice is the difference between the two LIS variants, and it is a favourite edge case.

### 14.6 Merge intervals

```cpp
sort(iv.begin(), iv.end());            // by start, then end
vector<pair<int,int>> out;
for (auto [l, r] : iv) {
    if (!out.empty() && l <= out.back().second)
        out.back().second = max(out.back().second, r);
    else
        out.push_back({l, r});
}
```

The `max` matters: an interval fully contained in the current one must not shrink it.

### 14.7 Brute force with `next_permutation`

```cpp
sort(p.begin(), p.end());
int best = INT_MAX;
do {
    best = min(best, cost(p));          // O(n) evaluation
} while (next_permutation(p.begin(), p.end()));
```

Valid up to $n \approx 10$ ($10! = 3.6 \times 10^6$ iterations). For $n = 11$ it is $4 \times 10^7$ — borderline. This is the intended solution on many "minimum cost ordering" problems with $n \le 8$.

### 14.8 Deduplicate while keeping first occurrences

```cpp
vector<int> seen_sort;
for (int x : a) seen_sort.push_back(x);
sort(seen_sort.begin(), seen_sort.end());
seen_sort.erase(unique(seen_sort.begin(), seen_sort.end()), seen_sort.end());
```

Or, keeping original order, with a `set` of already-seen values — $\mathcal{O}(n \log n)$ but order-preserving, which `sort`+`unique` does not.

### 14.9 Custom sort by second then first

```cpp
sort(v.begin(), v.end(), [](auto &a, auto &b) {
    if (a.second != b.second) return a.second < b.second;
    return a.first < b.first;
});
```

The canonical multi-key comparator. Note the `!=` guard: returning `a.second <= b.second` would break strict weak ordering.

### 14.10 Median without sorting

```cpp
nth_element(v.begin(), v.begin() + n/2, v.end());
int med = v[n/2];                      // upper median for even n
```

For the true median of an even-length array, average `v[n/2 - 1]` and `v[n/2]` after a `nth_element` on both, or just use the upper median if the problem defines it that way — read the statement.

---

## 15. Chapter 1 recap

You have covered the entire C++ toolkit that competitive programming runs on:

| Topics | What you now own |
|---|---|
| 01–02 | Program structure, the contest template, compiling, reading diagnostics, debugging without an IDE |
| 03 | I/O at every speed tier, from `cin` to a `fread` reader, and the traps in each |
| 04–05 | Types, ranges, overflow, number systems, and every string operation you need |
| 06–07 | Branching and iteration, including off-by-one analysis and `auto`'s two silent type changes |
| 08–09 | Math functions and arrays — the memory model, stack limits, prefix sums, difference arrays |
| 10–11 | Pointers, references, the stack/heap distinction, structs, and lambdas including recursive ones |
| 12–14 | Every container you routinely need, with the complexity and invalidation rules for each |
| 15 | The algorithm layer: sorting, binary search, compaction, permutations, accumulators |

The through-line is **predictability**: every tool here has a known cost, a known failure mode, and a known fix. Chapter 2 moves from the language to the discipline — time complexity first, then the search techniques, data structures and greedy reasoning that turn a program that compiles into a program that is accepted.

---

## Key Takeaways

- All STL algorithms take a half-open range `[first, last)`. The count is `last - first`, no `+1`.
- `sort` is introsort: $\mathcal{O}(n\log n)$ worst case, unstable, $\mathcal{O}(1)$ extra space. Use `stable_sort` only when you need tie preservation.
- A comparator must be a **strict weak ordering**: `cmp(a,a)` is `false`. Using `<=` can segfault `std::sort`.
- `nth_element` gives the $k$-th smallest in $\mathcal{O}(n)$; `partial_sort` gives the top $k$ in $\mathcal{O}(n \log k)$. Do not full-sort for one order statistic.
- `lower_bound` = first $\ge x$; `upper_bound` = first $> x$; their difference is the count of $x$. Both require a sorted range and `lower_bound` never fails.
- For `set`/`map`, use the **member** `lower_bound` — the algorithm version is $\mathcal{O}(n)$.
- Binary search on the answer: Template A (`hi = mid`) for "first true"; Template B (`mid = lo + (hi-lo+1)/2`) for "last true". The `+1` prevents an infinite loop.
- `remove` and `unique` do not erase — they compact and return a new end. Always follow with `erase(newEnd, end())`.
- `unique` only collapses **adjacent** duplicates, so `sort` first.
- `accumulate`'s accumulator type is the **initial value's type**. Pass `0LL`, never `0`, for large integer sums.
- `reverse`, `rotate`, `fill`, `iota`, `transform`, `count_if`, `any_of` replace hand-written loops and are harder to get wrong.
- `next_permutation` needs a sorted start and costs $\mathcal{O}(n \cdot n!)$ — viable to $n \approx 10$.
- `__builtin_clz(0)` and `__builtin_ctz(0)` are undefined behaviour, not 32 and not 0.
- `lower_bound` vs `upper_bound` in the LIS algorithm distinguishes strictly increasing from non-decreasing. Read the statement.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Comparator uses `<=` | Violates strict weak ordering; `sort` can segfault | `<` or `>` only |
| 2 | Comparator returns `true` for equal elements | Same | Equal keys must compare `false` both ways |
| 3 | `sort(v.begin()+l, v.begin()+r)` with inclusive intent | Element `r` is not sorted | `v.begin()+r+1`, or the `[l, r)` mindset |
| 4 | Comparator depends on mutated global state | Ordering changes mid-sort → UB | Capture copies, keep state constant |
| 5 | `lower_bound` on an unsorted range | Plausible wrong answer, no error | Sort first, or use a linear scan |
| 6 | `*lower_bound(...)` without checking `end()` | Dereferences past the array | Compare against `end()` first |
| 7 | `--it` after `lower_bound` without checking `begin()` | UB at the front of the range | `if (it != a.begin()) --it;` |
| 8 | `std::lower_bound` on a `set` | $\mathcal{O}(n)$ → TLE | `s.lower_bound(x)` |
| 9 | Template B with `(lo+hi)/2` | Infinite loop when `hi == lo+1` | `(lo+hi+1)/2` |
| 10 | `(lo + hi) / 2` with $10^{18}$ values | Signed overflow → negative `mid` | `lo + (hi - lo) / 2` |
| 11 | `while (hi - lo > 1e-9)` for doubles | May never terminate at large magnitude | Fixed 100 iterations |
| 12 | `remove(...)` without `erase(...)` | Size unchanged; the "removed" values are still there | `v.erase(remove(...), v.end())` |
| 13 | `erase` inside a `for (int i...)` loop | Skips elements, $\mathcal{O}(n^2)$ | `remove_if` or the iterator form |
| 14 | `unique` without `sort` | Only adjacent duplicates collapse | Sort first for global dedupe |
| 15 | `accumulate(v.begin(), v.end(), 0)` on big sums | `int` accumulator overflows silently | `0LL` |
| 16 | `*max_element` on an empty range | Dereferences `end()` | Guard `if (v.empty())` |
| 17 | `max_element` assumed to give the last maximum | Returns the **first** | Reverse comparator, or a manual loop |
| 18 | `rotate` with `k > n` | `begin()+k` is UB | `k %= n` first |
| 19 | `next_permutation` without a prior `sort` | Misses permutations | Sort before the `do/while` |
| 20 | `next_permutation` on $n = 15$ | $15! = 1.3 \times 10^{12}$ — hopeless | Only for $n \le 10$ |
| 21 | `__builtin_clz(0)` | Undefined behaviour, garbage result | Guard against 0 |
| 22 | `__builtin_popcount` on a 64-bit value | Truncates to 32 bits | `__builtin_popcountll` |
| 23 | `lcm(a,b)` assigned to `int` | Result exceeds $2^{31}$ | `long long` arguments and result |
| 24 | `copy_if` without `back_inserter` into an empty vector | Writes out of bounds | `back_inserter(dst)` |
| 25 | `lower_bound` in LIS where `upper_bound` was needed | Wrong answer on duplicates (non-decreasing variant) | Decide strictness from the statement |
| 26 | `nth_element` with `begin()+k` where `k == n` | `end()` is not a valid position | `k` must be `< n` |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Sort an array of structs by two keys | Codeforces Div. 3 B | Easy |
| 2 | Count occurrences of a value with `upper_bound - lower_bound` | CSES "Binary Search" set | Easy |
| 3 | First and last position of a value in a sorted array | Codeforces Div. 3 B | Easy |
| 4 | Remove duplicates from a sorted vector in place | CSES / AtCoder ABC | Easy |
| 5 | Coordinate compression, then answer index queries | Codeforces Div. 2 B/C | Easy–Medium |
| 6 | $k$-th smallest element, $n \le 10^6$ | AtCoder ABC | Easy–Medium |
| 7 | Top-$k$ elements with `partial_sort` | Codeforces Div. 3 C | Easy–Medium |
| 8 | Two-pointer pair sum on a sorted array | CSES "Sum of Two Values" | Easy–Medium |
| 9 | Minimum capacity to ship within $D$ days (binary search on answer) | Codeforces Div. 2 C / LeetCode-style | Medium |
| 10 | Aggressive cows / maximise the minimum distance | CSES "Factory Machines" family | Medium |
| 11 | Binary search on doubles: square root / maximise a ratio | Codeforces Div. 2 C | Medium |
| 12 | Merge overlapping intervals | Codeforces Div. 3 C | Medium |
| 13 | LIS in $\mathcal{O}(n\log n)$, both strict and non-decreasing variants | CSES "Increasing Subsequence" | Medium |
| 14 | Count pairs with difference $\le k$ | Codeforces Div. 2 C | Medium |
| 15 | Enumerate permutations for $n \le 8$, minimise a cost | Codeforces Div. 2 B | Medium |
| 16 | $k$-th permutation of $1..n$ | CSES "Permutations"-adjacent | Medium |
| 17 | Prefix sums with `exclusive_scan`, range-sum queries | CSES "Static Range Sum Queries" | Easy |
| 18 | `nth_element` median, then count elements above it | Codeforces Div. 2 B | Medium |
| 19 | Sort indices by value, reconstruct the original order | Codeforces Div. 3 D | Medium |
| 20 | Comparator drill: sort by absolute difference from a pivot, ties by index | Self-imposed | Medium |
| 21 | Bit-counting with `__builtin_popcountll`, sum over a range | Codeforces Div. 2 B | Easy–Medium |
| 22 | Sort + unique + `lower_bound` pipeline on $10^6$ queries | CSES "Distinct Numbers" style | Medium |

---

## Next Up

→ **Chapter 2 — CP Level 1**, starting with *Introduction to CP and ICPC*, then *Time Complexity (Basics to Advanced)*.

Chapter 1 gave you the language. Chapter 2 gives you the discipline: how to estimate whether your algorithm will pass before you write it, how to search a space completely, how sorting and sets turn $\mathcal{O}(n^2)$ into $\mathcal{O}(n \log n)$, and how to reason about a greedy choice. Everything in this file will be used there — `sort` and `lower_bound` are the backbone of almost every Chapter 2 solution.