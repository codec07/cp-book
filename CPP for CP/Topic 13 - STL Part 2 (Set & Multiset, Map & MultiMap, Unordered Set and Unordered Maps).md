# Topic 13 — STL Part 2: `set`, `multiset`, `map`, `multimap`, `unordered_*`

A `vector` answers "what is at index 5?" in one instruction. It cannot answer "which element is closest to 42?" without scanning. The ordered associative containers can — in $\mathcal{O}(\log n)$ — and that single capability unlocks an enormous class of problems: predecessor/successor queries, dynamic medians, interval merging, sweep lines, deduplication, annotation. This chapter gives you the exact API, the member-vs-free `lower_bound` trap that causes more unexplained TLEs than any other STL mistake, the multiset `erase` bug, and the anti-hash defence every Codeforces regular eventually needs.

> **Prerequisites:** [Topic 12 — STL Part 1](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md) (iterators, `vector`, invalidation), [Topic 11 — Functions, Classes and Lambdas](Topic%2011%20-%20Functions%2C%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md) (`operator<`, functors, lambdas), [Topic 08 — Useful Math Functions](Topic%2008%20-%20Useful%20Math%20functions%20%26%20Operations%20on%20Data.md) (modular arithmetic — not strictly needed).
> **You will be able to:**
> - Explain why every `set`/`map` operation costs $\mathcal{O}(\log n)$ and why that constant is large.
> - Use `insert`'s `pair<iterator,bool>` return to test-and-insert in one pass.
> - Never write `std::lower_bound(s.begin(), s.end(), x)` again.
> - Delete one copy from a `multiset` instead of all of them.
> - Defend `unordered_map` against crafted collision attacks with a seeded custom hash.
> - Choose ordered vs unordered with a defensible argument, not a superstition.

---

## 1. What "Ordered" Actually Costs You

`set`, `multiset`, `map` and `multimap` are all thin wrappers around the **same** data structure: a red-black tree — a self-balancing binary search tree where every root-to-leaf path has length within a factor of two of every other.

```text
                [ 8 ]                     each node stores
              /       \                   ┌──────────────┐
           [ 3 ]     [ 12 ]               │ left  pointer│
           /   \     /    \               │ right pointer│
        [ 1 ] [ 5 ] [10]  [ 15 ]          │ parent ptr   │
                \                         │ colour (1 bit)│
                [ 6 ]                     │ value        │
                                          └──────────────┘
        height <= 2*log2(n+1)
```

Three consequences follow directly, and every rule in this chapter is one of them:

| Consequence | Why | What it means for you |
|---|---|---|
| Every operation is $\mathcal{O}(\log n)$ | Tree height is $\le 2\log_2(n+1)$ | $10^6$ elements → ~40 comparisons; $10^6$ queries → ~4×10^7 comparisons |
| Iteration is **sorted** | In-order traversal | `for (int x : s)` yields ascending order for free |
| Every element is a separate heap node | Pointers, not an array | Terrible locality; ~40 bytes per `int` |

### The node-size estimate

A libstdc++ `_Rb_tree_node` is three pointers (24 B) plus a colour bit stored in a low pointer bit, padded to **32 bytes**, followed by the value:

| Container | Node header | Value | Total per element | $10^6$ elements |
|---|---|---|---|---|
| `set<int>` | 32 B | 4 B → padded 8 | ~40 B | ~40 MB (≈48 with allocator headers) |
| `map<int,int>` | 32 B | 8 B | ~40 B | ~40 MB |
| `set<long long>` | 32 B | 8 B | ~40 B | ~40 MB |
| `set<string>` | 32 B | 32 B (`string`) | ~64 B + heap | ~64 MB + text |

Compare `vector<int>`: 4 bytes per element. **A `set<int>` costs ten times the memory of a `vector<int>`.** With $10^6$ elements and a 64 MB limit, a `set` is already at the edge; two of them is an instant MLE.

*Contest story: a Div. 2 D needed "distinct values with delete". The author's reference used `set<int>`; a contestant stored the same $10^6$ values in a sorted `vector` plus a `multiset` of the changed ones and passed in 0.3 s. The tree was not the bottleneck — 200 MB of nodes hit the limit first.*

---

## 2. `set` — The Core API

A `set` holds **unique** keys in sorted order.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    set<int> s;
    s.insert(5);
    s.insert(1);
    s.insert(3);
    s.insert(1);                 // ignored — already present

    for (int x : s) cout << x << ' ';   // 1 3 5
    cout << "| size=" << s.size() << '\n';

    auto [it, inserted] = s.insert(7);   // C++17 structured binding
    cout << *it << ' ' << inserted << '\n';

    auto again = s.insert(7);
    cout << *again.first << ' ' << again.second << '\n';   // 7 0

    cout << s.count(3) << ' ' << s.count(4) << '\n';       // 1 0
    return 0;
}
```

```text
1 3 5 | size=3
7 1
7 0
1 0
```

### `insert` tells you whether it was new

`insert` returns `pair<iterator, bool>`. The `bool` is `true` only if the element was actually added. This lets you detect a duplicate **in the same $\mathcal{O}(\log n)$ pass**, with no second lookup:

```cpp
set<int> seen;
for (int x : a) {
    if (!seen.insert(x).second) {
        cout << "duplicate: " << x << '\n';   // first repeat wins
        break;
    }
}
```

That is the cleanest duplicate detector in the language. `if (s.count(x)) { ... } s.insert(x);` does two tree descents; `insert(...).second` does one.

### Erase, count, find, contains

| Call | Effect | Cost |
|---|---|---|
| `s.erase(key)` | Remove by value; returns count removed (0 or 1 for `set`) | $\mathcal{O}(\log n)$ |
| `s.erase(it)` | Remove at iterator; returns the **next** iterator (C++11) | Amortised $\mathcal{O}(1)$ |
| `s.erase(first, last)` | Remove a range | $\mathcal{O}(\log n + k)$ |
| `s.count(key)` | 0 or 1 for `set` | $\mathcal{O}(\log n)$ |
| `s.find(key)` | Iterator to the element, or `s.end()` | $\mathcal{O}(\log n)$ |
| `s.contains(key)` | `bool` — **C++20 only** | $\mathcal{O}(\log n)$ |
| `s.clear()` | Remove all | $\mathcal{O}(n)$ |

```cpp
set<int> s = {1, 2, 3, 4, 5};

s.erase(3);                    // by value
auto it = s.find(4);
if (it != s.end()) s.erase(it);          // by iterator

// C++17 portable test: count != 0
if (s.count(2)) cout << "has 2\n";

// C++20 alternative, not available on older judges:
// if (s.contains(2)) cout << "has 2\n";
```

> **Portability note:** `contains` is C++20. Codeforces' default is GNU++17 (and C++20 is selectable), but many judges are stuck on 17. `count(x)` is the safe habit — it compiles everywhere and the compiler folds the `!= 0` test away.

Never write `s.find(x) != s.end()` and then dereference a *stale* iterator after an `erase`. Erase returns the next iterator precisely so you do not have to compute it.

---

## 3. Member `lower_bound` vs `std::lower_bound` — The Classic TLE

This is the single most expensive misunderstanding in the STL.

```cpp
set<int> s = {10, 20, 30, 40, 50};

auto good = s.lower_bound(25);                        // member: O(log n)
auto bad  = lower_bound(s.begin(), s.end(), 25);      // free function: O(n)
// *good == *bad == 30
```

Both compile. Both give the right answer. One is roughly **50,000x slower** on a million-element set, and the difference is invisible in the source.

### The mechanism

`std::lower_bound` is a *generic* binary search. It needs to jump to the middle of the range, and the only tool every iterator category provides is `++`. So the implementation does:

$$T(n) = T(n/2) + \underbrace{\Theta(n/2)}_{\texttt{std::advance}} \quad\Longrightarrow\quad T(n) = \Theta(n).$$

At each of the $\log n$ steps it walks $\tfrac{n}{2}, \tfrac{n}{4}, \dots$ nodes — and on a red-black tree each step is also a cache miss. The member `lower_bound` instead walks the tree by comparing at each node: left, right, left — $\log n$ steps, $\log n$ comparisons, no jumping.

| Call | Method | Complexity on $n = 10^6$ | Time per query |
|---|---|---|---|
| `s.lower_bound(x)` | Tree descent | $\mathcal{O}(\log n)$, ~20 steps | ~1 µs |
| `lower_bound(s.begin(), s.end(), x)` | Repeated `advance` | $\mathcal{O}(n)$ | ~10–50 ms |
| `std::lower_bound` on a `vector<int>` | Pointer arithmetic | $\mathcal{O}(\log n)$ | ~0.1 µs |

So the free function is perfect on a `vector` (random access makes the jump free) and useless on a `set`. **Rule: on any node-based container, use the member function.**

The same distinction exists for:

| Container | Members to prefer |
|---|---|
| `set` / `multiset` | `lower_bound`, `upper_bound`, `equal_range`, `count`, `find` |
| `map` / `multimap` | the same, keyed on the key |
| `list` | `sort`, `reverse`, `unique`, `remove`, `merge`, `splice` |
| `vector` / `deque` | free functions are fine — random access |

---

## 4. Iterating, Min, Max, Neighbours

Because iteration is in-order, the extremes are one step from the ends.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    set<int> s = {4, 1, 9, 7};

    for (int x : s) cout << x << ' ';         // 1 4 7 9
    cout << '\n';

    cout << "min=" << *s.begin() << '\n';      // 1
    cout << "max=" << *s.rbegin() << '\n';     // 9

    auto it = s.find(4);
    cout << *prev(it) << ' ' << *next(it) << '\n';   // 1 7

    s.erase(s.begin());                        // remove min
    s.erase(prev(s.end()));                    // remove max
    for (int x : s) cout << x << ' ';
    cout << '\n';
    return 0;
}
```

```text
1 4 7 9
min=1
max=9
1 7
4 7
```

Guards you must respect: `*s.begin()` and `*s.rbegin()` are **undefined on an empty set**, and `prev(s.begin())` is undefined always (there is nothing before the first element). `next(s.rbegin())` is fine and equals `s.end()`.

`std::next` / `std::prev` are `#include <iterator>` (pulled in by `<bits/stdc++.h>`). They are $\mathcal{O}(1)$ for a `set` because the iterator is bidirectional, and `++`/`--` on a red-black tree node uses the parent pointer.

| Expression | Meaning | Cost |
|---|---|---|
| `*s.begin()` | Smallest | $\mathcal{O}(1)$ |
| `*s.rbegin()` | Largest (`*prev(s.end())`) | $\mathcal{O}(1)$ |
| `*next(it)` | Successor | $\mathcal{O}(1)$ amortised |
| `*prev(it)` | Predecessor | $\mathcal{O}(1)$ amortised |
| `s.erase(s.begin())` | Delete smallest | $\mathcal{O}(\log n)$ |

*Deleting the minimum in a loop is the standard way to drain a set — but remember each `erase` is $\mathcal{O}(\log n)$, so draining $n$ elements costs $\mathcal{O}(n\log n)$, not $\mathcal{O}(n)$.*

---

## 5. The Two Queries You Will Write Forever

**Ceiling — smallest element $\ge x$:**

```cpp
set<int> s = {10, 20, 30, 40};

auto it = s.lower_bound(25);          // first element >= 25
if (it == s.end()) cout << "none\n";  // EVERY element is < 25
else               cout << *it << '\n';   // 30
```

`lower_bound` is defined as "first element **not less than** $x$" — so on a hit it is the ceiling; on `end()` there is no ceiling.

**Floor — largest element $\le x$:**

```cpp
auto it = s.upper_bound(25);          // first element > 25  -> 30
if (it == s.begin()) cout << "none\n";    // every element is > 25
else                 cout << *prev(it) << '\n';   // 20
```

The off-by-one is the point. `lower_bound(x)` gives the first element $\ge x$, which is the element *after* the floor candidate. `upper_bound(x)` gives the first element $> x$; step back one and you land on the largest element $\le x$. Using `lower_bound` and `--it` instead would give the largest element $< x$ — wrong when $x$ is present in the set.

| Want | Call | Guard | On empty set |
|---|---|---|---|
| Smallest $\ge x$ | `s.lower_bound(x)` | `it == s.end()` | returns `end()` → guard catches it |
| Largest $\le x$ | `prev(s.upper_bound(x))` | `it == s.begin()` | `begin() == end()` → guard catches it |
| Smallest $> x$ | `s.upper_bound(x)` | `it == s.end()` | safe |
| Largest $< x$ | `prev(s.lower_bound(x))` | `it == s.begin()` | safe |

Both guards are necessary and both are correct on an empty set: `s.begin() == s.end()`, so the `end()` test and the `begin()` test each fire.

---

## 6. `multiset` — Duplicates, and the Number-One Bug

A `multiset` is a `set` that keeps equal keys. Ordering is stable among equal keys (insertion order, in practice), but that is not guaranteed by the standard — do not rely on it.

```cpp
multiset<int> ms = {1, 2, 2, 2, 3};
cout << ms.count(2) << '\n';      // 3
```

`count` on a `multiset` is $\mathcal{O}(\log n + k)$ where $k$ is the number of matches: it finds the first, then walks the run. Counting an element that appears $10^5$ times costs $10^5$, not $\log n$.

### The bug

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    multiset<int> ms = {1, 2, 2, 2, 3};

    cout << "before: ";
    for (int x : ms) cout << x << ' ';
    cout << '\n';

    ms.erase(2);                      // deletes ALL THREE copies!
    cout << "erase(2): ";
    for (int x : ms) cout << x << ' ';
    cout << " size=" << ms.size() << '\n';

    ms = {1, 2, 2, 2, 3};
    auto it = ms.find(2);
    if (it != ms.end()) ms.erase(it);  // deletes exactly ONE copy
    cout << "erase(find(2)): ";
    for (int x : ms) cout << x << ' ';
    cout << " size=" << ms.size() << '\n';
    return 0;
}
```

```text
before: 1 2 2 2 3
erase(2): 1 3  size=2
erase(find(2)): 1 2 2 3  size=4
```

`erase(key)` is the *value* overload — it removes every element equal to the key and returns how many it removed. `erase(iterator)` removes one node.

| Goal | Write | Cost |
|---|---|---|
| Delete **all** copies of `x` | `ms.erase(x)` | $\mathcal{O}(\log n + k)$ |
| Delete **one** copy of `x` | `ms.erase(ms.find(x))` | $\mathcal{O}(\log n)$ |
| Delete one copy, with a guard | `auto it = ms.find(x); if (it != ms.end()) ms.erase(it);` | $\mathcal{O}(\log n)$ |
| Delete one copy from the low end | `ms.erase(ms.begin())` | $\mathcal{O}(\log n)$ |

> **Never** write `ms.erase(ms.find(x))` without checking `find(x) != end()`. Erasing `end()` is undefined behaviour — and in debug mode it aborts, which at least tells you the truth.

Multiset is the right tool for "keep a collection where I may insert or delete one occurrence of a value and always want the minimum/maximum" — the sliding-window median, the k-th smallest with deletions, the top-$k$ tracker. In those problems `erase(value)` instead of `erase(iterator)` silently deletes all duplicates and produces a wrong answer that survives small tests.

---

## 7. `map` — Sorted Key–Value

A `map` is a `set` of `pair<const Key, Value>`, ordered by key. Iteration is by ascending key.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    map<string, int> age;

    age["alice"] = 30;                 // operator[] INSERTS if missing
    age.insert({"bob", 25});           // no-op if key exists
    age.emplace("carol", 41);          // constructs in place
    age["bob"] = 26;                   // overwrites

    for (auto &[name, a] : age)        // C++17 structured binding
        cout << name << '=' << a << ' ';

    cout << "\nsize=" << age.size() << '\n';

    cout << age.count("dave") << '\n';        // 0
    if (auto it = age.find("dave"); it == age.end())
        cout << "dave not found, and size is still "
             << age.size() << '\n';           // find does NOT insert
    return 0;
}
```

```text
alice=30 bob=26 carol=41
size=3
0
dave not found, and size is still 3
```

Note the `const` in `pair<const Key, Value>`: the key is immutable inside the container, which is why `it->first = "x"` does not compile (§13).

### `operator[]` inserts. This is the silent-bloat bug.

`m[k]` is defined as "return a reference to the value for `k`, **default-constructing and inserting it if absent**". So:

```cpp
map<int, int> cnt;
for (int q : queries) {
    if (cnt[q] == 0) {          // BUG: creates an entry for every q
        /* count it as new */
    }
    cnt[q]++;
}
```

Every query that is *not* present creates a node with value 0. With $10^5$ distinct queries you get $10^5$ nodes you never wanted — 4 MB of tree plus $\log n$ work each. On a 64 MB limit with a $10^6$ domain, that is a quiet MLE.

| Intent | Safe call | Inserting call |
|---|---|---|
| Read without creating | `m.find(k)` or `m.at(k)` | `m[k]` |
| Read with a fallback, no insert | `auto it = m.find(k); int v = it == m.end() ? 0 : it->second;` | — |
| Count occurrences | `m[k]++` (insert is intended) | — |
| Test membership | `m.count(k)` | `m[k] == 0` |
| Insert only if absent | `m.insert({k, v})` / `m.emplace(k, v)` | `m[k] = v` |

`m.at(k)` throws `std::out_of_range` when absent, which is a good debugging choice and a bad contest choice (exception overhead and an abort if uncaught).

### Sparse array pattern

A `map<long long, int>` is the standard sparse array when the index domain is $10^{18}$ but you touch only $10^5$ cells. `m[x]++` costs $\mathcal{O}(\log n)$ — slower than `unordered_map` by a constant, but iteration comes out sorted, which is often exactly what the next step needs.

### Frequency counting: three ways

| Method | Insert | Lookup | Iteration order | Memory per entry |
|---|---|---|---|---|
| `vector<int> cnt(D)` | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | index order | 4 B — **only if $D$ is small** |
| `map<int,int>` | $\mathcal{O}(\log n)$ | $\mathcal{O}(\log n)$ | **sorted** | ~40 B |
| `unordered_map<int,int>` | $\mathcal{O}(1)$ avg | $\mathcal{O}(1)$ avg | arbitrary | ~24 B + buckets |
| sorted `vector` + `lower_bound` | $\mathcal{O}(n\log n)$ once | $\mathcal{O}(\log n)$ | sorted | 4 B |

The right answer for "count occurrences of $10^6$ values in $[1,10^6]$" is a `vector<int>` of size $10^6$ — 4 MB and one load per access. Reach for a `map` only when the domain is large or the keys are not integers.

---

## 8. `multimap` — Duplicate Keys, No `operator[]`

A `multimap` allows repeated keys. Because `operator[]` has no sensible meaning ("which of the three values for key 5?"), it does not exist.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    multimap<string, int> scores;
    scores.emplace("ann", 90);
    scores.emplace("ann", 85);
    scores.emplace("bob", 70);

    cout << scores.count("ann") << '\n';        // 2

    auto [lo, hi] = scores.equal_range("ann");
    for (auto it = lo; it != hi; ++it)
        cout << it->first << '=' << it->second << ' ';

    cout << '\n';
    return 0;
}
```

```text
2
ann=90 ann=85
```

`equal_range(k)` returns `{lower_bound(k), upper_bound(k)}` — the half-open range of all entries with that key — in one $\mathcal{O}(\log n)$ pass, which is strictly better than calling the two bounds separately. Self-consistent keys mean the range is contiguous in iteration order.

**Verdict: you rarely need it.** The idiomatic CP alternative is `map<K, vector<V>>`: same sorted-key iteration, `m[k].push_back(v)` reads better than `emplace`, and you get `size()` of the bucket for free. Use `multimap` when you need the entries *interleaved* in key order and never want a vector allocation.

---

## 9. `unordered_set` / `unordered_map` — Hash Tables and the Anti-Hash Attack

These are **separate chaining** hash tables: an array of buckets, each bucket a linked list of nodes. Average $\mathcal{O}(1)$, worst case $\mathcal{O}(n)$ per operation.

```text
 buckets          nodes
 ┌────┐
 │  •─┼──► [ h%7=2 | 9 ] ──► [ 16 ] ──► null
 ├────┤
 │  •─┼──► [ 23 ]
 ├────┤
 │null│
 └────┘
```

| Operation | Average | Worst case |
|---|---|---|
| `insert` | $\mathcal{O}(1)$ | $\mathcal{O}(n)$ |
| `find` / `count` | $\mathcal{O}(1)$ | $\mathcal{O}(n)$ |
| `erase` | $\mathcal{O}(1)$ | $\mathcal{O}(n)$ |
| Iteration | $\mathcal{O}(n + \text{buckets})$ | same |

The worst case occurs when every key lands in the same bucket: each operation walks the whole chain. That is not a theoretical curiosity — it is a **deliberate attack**.

### The attack

`std::hash<int>` is the identity function on libstdc++, and the bucket count is a prime. So the bucket for key $k$ is $k \bmod p$. A setter who knows $p$ (it is deterministic for a given insert sequence) can submit $10^5$ keys all congruent mod $p$:

```cpp
// What an adversarial test does to unordered_map<int,int>:
// choose keys k with k % BUCKETS == 0 for the bucket count the judge's
// libstdc++ will pick, then submit 10^5 of them as queries.
// Every lookup becomes a full chain walk -> O(n) per query -> O(n^2) total.
```

With $n = 2\times10^5$ queries and chains of length $n$, you perform $4\times10^{10}$ pointer chases. Time limit exceeded, every time, on a solution that is algorithmically correct.

*On Codeforces the hack phase makes this routine. A "correct" `unordered_map` solution that passes pretests can be hacked into a TLE by any Div. 2 participant who knows the trick.*

### The defence: a seeded custom hash

Randomise the hash per run so the setter cannot precompute collisions. This is the splitmix64 construction, the standard CP answer:

```cpp
#include <bits/stdc++.h>
using namespace std;

struct custom_hash {
    static uint64_t splitmix64(uint64_t x) {
        // 64-bit finaliser: full avalanche, no collisions in practice
        x += 0x9e3779b97f4a7c15ULL;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
        x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
        return x ^ (x >> 31);
    }

    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED_RANDOM =
            chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + FIXED_RANDOM);
    }
};

int main() {
    unordered_map<long long, int, custom_hash> safe;
    unordered_set<long long, custom_hash> seen;

    safe[1'000'000'007] = 42;
    seen.insert(12345);
    cout << safe[1'000'000'007] << ' ' << seen.count(12345) << '\n';
    return 0;
}
```

```text
42 1
```

`custom_hash` is the **third** template argument of both containers: `unordered_map<Key, Value, Hash>`. Because `FIXED_RANDOM` is drawn from the clock at first use, the adversary cannot predict which keys collide — their precomputed set becomes random noise, restoring average $\mathcal{O}(1)$.

Two notes:

- The argument type is `uint64_t`. Keys that are `int`, `long long`, `unsigned`, or implicitly convertible work. For `string` keys you need a different hash; `std::hash<string>` is already randomised in libstdc++? It is not — but string-hash attacks require knowing the exact implementation's seed, and are far rarer in practice.
- Some judges use libc++ (macOS/clang) where `std::hash<int>` differs. The custom hash is compiler-independent, which is another reason to keep it in your template.

A `pair` key needs a combiner:

```cpp
struct pair_hash {
    size_t operator()(const pair<int,int> &p) const {
        return ((size_t)p.first << 32) ^ (size_t)p.second;
    }
};
// unordered_map<pair<int,int>, int, pair_hash> m;
```

That is not seeding — it is deterministic. For a fully safe pair hash, run `splitmix64` on the combined value.

---

## 10. `reserve` and `max_load_factor`

An unordered container rehashes when `size() > bucket_count() * max_load_factor()`. Rehashing moves every node into a new bucket array — $\mathcal{O}(n)$ — and invalidates all iterators.

```cpp
unordered_map<int, int, custom_hash> m;
m.reserve(200000);                 // size the bucket array for 2e5 elements
m.max_load_factor(0.7);            // default 1.0; lower = fewer collisions
```

`reserve(n)` sets `bucket_count` so that `n` elements fit under the current load factor. Call it **before** inserting. `max_load_factor(0.7)` trades ~40% more memory for ~30% fewer collisions; worthwhile when your keys collide often and memory is not tight.

| Setting | Buckets for $10^6$ | Memory (buckets alone) | Effect |
|---|---|---|---|
| Default `lf = 1.0` | ~1.06M | ~8.5 MB | Baseline |
| `lf = 0.7` | ~1.5M | ~12 MB | Fewer collisions |
| No `reserve`, 24 growth steps | final ~1.06M | — | ~24 rehashes, $\mathcal{O}(n)$ total |

The rehash cost is amortised, exactly like `vector` growth — but rehashing also *invalidates iterators*, so reserve up front if you hold any.

---

## 11. Ordered vs Unordered

| Question | Answer | Container |
|---|---|---|
| Do you need `lower_bound` / `upper_bound` / `equal_range`? | Yes | **ordered** |
| Do you need sorted iteration (output, greedy, sweep)? | Yes | **ordered** |
| Do you need `prev`/`next` neighbours of a key? | Yes | **ordered** |
| Need deterministic output across runs? | Yes | **ordered** |
| Pure membership / counting, $10^6$ queries? | No | **unordered** (with custom hash) |
| Keys are `pair`/`tuple`/`string` and you only look up? | No | **unordered** + custom hash |
| Memory limit is tight and $n \ge 10^6$? | Either — measure | `vector` if the domain allows |

**Speed, order of magnitude** for $10^6$ `int` keys (typical):

| Operation | `unordered_map` | `map` | Ratio |
|---|---|---|---|
| 1e6 inserts | ~0.10 s | ~0.55 s | 5x |
| 1e6 lookups | ~0.05 s | ~0.40 s | 8x |
| 1e6 erases | ~0.09 s | ~0.55 s | 6x |

The gap is cache locality, not asymptotics: `map` does ~20 dependent pointer dereferences per query, each a potential miss; the hash table does one modulo and one pointer hop.

> **Determinism matters more than you think.** If two solutions differ only in iteration order and you print the last one seen, `unordered_map` gives a different answer per compiler. When in doubt, sort the keys or use an ordered container.

---

## 12. Custom Comparators and Non-`int` Keys

### Descending order

```cpp
set<int> inc  = {3, 1, 2};                 // 1 2 3
set<int, greater<int>> dec = {3, 1, 2};    // 3 2 1

// now *dec.begin() is the MAXIMUM, and dec.lower_bound works in reverse order
```

`greater<int>` is `std::greater` from `<functional>`; it defines a strict weak ordering, which is all the tree needs.

### `pair` and `tuple` keys: ordering is built in

`std::pair` and `std::tuple` compare **lexicographically** — first element, then second, and so on. No comparator is needed:

```cpp
set<pair<int,int>> sp;
sp.insert({1, 5});
sp.insert({1, 2});
sp.insert({0, 9});
for (auto &[a, b] : sp) cout << '(' << a << ',' << b << ") ";
// (0,9) (1,2) (1,5)

auto it = sp.lower_bound({1, 3});           // first pair >= (1,3) -> (1,5)
```

This is the workhorse of competitive programming: `set<pair<value, index>>` gives you a sorted multiset with tie-breaking, and `set<tuple<int,int,int>>` sorts a 3-D key for free.

### A struct in a `set` needs `operator<`

The tree needs exactly one thing: a **strict weak ordering**. Provide `operator<` and everything else (`find`, `insert`, `lower_bound`) works.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    int x, y;
    bool operator<(const Point &o) const {      // MUST be const
        if (x != o.x) return x < o.x;           // compare x, then y
        return y < o.y;
    }
};

int main() {
    set<Point> pts;
    pts.insert({3, 4});
    pts.insert({1, 2});
    pts.insert({1, 1});
    for (auto &p : pts) cout << '(' << p.x << ',' << p.y << ") ";
    cout << '\n';
    return 0;
}
```

```text
(1,1) (1,2) (3,4)
```

A one-liner for the same thing: `return tie(x, y) < tie(o.x, o.y);` using `std::tie`. Do not compare `x + y` or `x*y` — that is not a valid ordering and the tree will misbehave.

### A functor comparator

```cpp
struct BySecond {
    bool operator()(const pair<int,int> &a, const pair<int,int> &b) const {
        if (a.second != b.second) return a.second < b.second;
        return a.first < b.first;               // tie-break or it is UB-ish
    }
};
set<pair<int,int>, BySecond> byVal;
```

The comparator must be `const`-callable and must impose a total order. **Always tie-break.** Two elements that compare "equivalent" are treated as duplicates by a `set`, so a comparator that only looks at `.second` silently drops distinct pairs.

A lambda works too, but needs the type spelled out (or `decltype`):

```cpp
auto cmp = [](int a, int b) { return a > b; };
set<int, decltype(cmp)> s(cmp);
```

---

## 13. Iterator Invalidation and Immutable Keys

### Erasing while iterating

`erase(it)` returns the next iterator, so the loop is:

```cpp
set<int> s = {1, 2, 3, 4, 5, 6};
for (auto it = s.begin(); it != s.end(); ) {
    if (*it % 2 == 0) it = s.erase(it);     // erase returns the successor
    else              ++it;
}
// s == {1,3,5}
```

Every other loop shape is a bug. `s.erase(it); ++it;` dereferences a freed node. The range-for version (`for (int x : s) if (bad) s.erase(x);`) is worse: it erases by *value* while holding an iterator into the erased node — undefined behaviour, and on a `multiset` it deletes all copies.

Invalidation rules for node-based containers are far gentler than `vector`'s:

| Operation | `set`/`map`/`multiset`/`multimap` | `unordered_*` |
|---|---|---|
| `insert` | **Nothing** invalidated | All iterators on **rehash** only |
| `erase(it)` | Only the erased iterator | Only the erased iterator |
| `erase(key)` | Only iterators to erased elements | Same |
| `find`/`count` | Nothing | Nothing |
| Rehash (implicit) | N/A | **All** iterators |
| `swap` | Iterators follow the elements | Same |

Because insertion never invalidates, you can hold an iterator into a `set` across a million inserts. That is the property `vector` lacks and the reason node-based containers exist at all.

### You cannot modify a key in place

```cpp
set<int> s = {1, 2, 3};
auto it = s.begin();
// *it = 10;      // compile error: iterator is const-qualified
```

The value type of a `set` iterator is `const T`, and for a `map` it is `pair<const Key, Value>`. Mutating a key in place would break the tree invariant — the node would sit in the wrong place and every subsequent lookup would lie.

The workaround is erase + insert:

```cpp
auto it = s.find(5);
if (it != s.end()) {
    s.erase(it);
    s.insert(6);          // O(log n) + O(log n)
}
```

For a `map` you *can* modify the mapped value: `it->second = 7;` compiles, because only the key is const.

### `set` vs `priority_queue`, briefly

A `priority_queue` gives you max (or min with `greater`) in $\mathcal{O}(1)$ peek and $\mathcal{O}(\log n)$ push/pop, in a contiguous heap — much faster constants, far less memory. It cannot delete an arbitrary element, cannot iterate, and cannot do `lower_bound`. So: **static set of insertions with repeated "give me the max" → `priority_queue`; arbitrary deletions or predecessor queries → `set`.** (Topic 14 owns the priority queue.)

---

## 14. CP Recipes

**1 — Count distinct elements.** `set` is $\mathcal{O}(n\log n)$; sort + `unique` is faster and smaller.

```cpp
vector<int> a = {5, 3, 5, 1, 3, 9};
set<int> s(a.begin(), a.end());
cout << s.size() << '\n';                     // 4

// Usually faster and smaller: sort + unique (Topic 15)
vector<int> b = a;
sort(b.begin(), b.end());
b.erase(unique(b.begin(), b.end()), b.end());
```

**2 — Find duplicates, reporting the first repeat.**

```cpp
set<int> seen;
int firstDup = -1;
for (int x : a)
    if (!seen.insert(x).second) { firstDup = x; break; }
```

**3 — Next greater element.** For each position, the smallest later value greater than `a[i]`. Query a `set` of the values seen so far with `upper_bound`.

```cpp
vector<int> a = {2, 7, 3, 5, 4, 6};
set<int> have;
vector<int> ans(a.size(), -1);
for (int i = (int)a.size() - 1; i >= 0; i--) {
    auto it = have.upper_bound(a[i]);          // smallest value > a[i]
    if (it != have.end()) ans[i] = *it;
    have.insert(a[i]);
}
// ans == {3, -1, 4, 6, 6, -1}
```

**4 — Dynamic median teaser (two multisets).** Keep the lower half in a max-heap-shaped multiset and the upper half in a min-heap-shaped one, rebalancing so their sizes differ by at most one.

```cpp
multiset<int> low, high;                       // low: smaller half, high: larger half
auto add = [&](int x) {
    if (low.empty() || x <= *low.rbegin()) low.insert(x);
    else                                   high.insert(x);
    if (low.size() > high.size() + 1) { high.insert(*low.rbegin()); low.erase(prev(low.end())); }
    if (high.size() > low.size())     { low.insert(*high.begin());  high.erase(high.begin()); }
};
// median is *low.rbegin() when low.size() > high.size(), else average of the two
```

Deletion is why this needs multisets rather than priority queues — and why `erase(iterator)` (§6) matters.

**5 — Group anagrams.**

```cpp
vector<string> words = {"eat", "tea", "tan", "ate", "nat", "bat"};
map<string, vector<string>> groups;
for (const string &w : words) {
    string key = w;
    sort(key.begin(), key.end());              // "eat"/"tea"/"ate" -> "aet"
    groups[key].push_back(w);
}
for (auto &[k, g] : groups) {
    cout << k << ':';
    for (auto &w : g) cout << ' ' << w;
    cout << '\n';
}
// aet: eat tea ate
// abt: bat
// ant: tan nat
```

**6 — Two-sum with a map.** Store value → index while scanning; look for the complement before inserting, so an element is never paired with itself.

```cpp
vector<int> a = {2, 7, 11, 15}, ans;
int target = 9;
unordered_map<int,int, custom_hash> pos;
for (int i = 0; i < (int)a.size(); i++) {
    auto it = pos.find(target - a[i]);
    if (it != pos.end()) { ans = {it->second, i}; break; }
    pos[a[i]] = i;
}
// ans == {0, 1}
```

**7 — Coordinate compression with a `map`.** Handy when you want the mapping available afterwards.

```cpp
vector<int> a = {100, 5, 100, 42};
map<int,int> id;
for (int x : a) id[x];                          // inserts 0 for each distinct key
int nxt = 0;
for (auto &[val, idx] : id) idx = nxt++;        // ascending order -> 0,1,2
for (int &x : a) x = id[x];
// a == {2, 0, 2, 1}
```

**8 — Mode / maximum frequency.**

```cpp
map<int,int> freq;
for (int x : a) freq[x]++;
int mode = -1, best = 0;
for (auto &[v, c] : freq)
    if (c > best) { best = c; mode = v; }       // strict > keeps the smallest value
```

**9 — Ordered pairs as a priority index.** `set<pair<int,int>>` sorts by value then index, giving you a "delete this exact occurrence" structure that a `priority_queue` cannot offer.

```cpp
set<pair<int,int>> pq;                          // (dist, node)
pq.insert({5, 1});
pq.insert({2, 3});
pq.insert({2, 7});                              // tie on distance, ordered by node
auto [d, u] = *pq.begin();                      // (2,3) — the global minimum
pq.erase({2, 7});                               // delete EXACTLY one entry, O(log n)
```

This is Dijkstra-with-decrease-key's lazy alternative, and the standard trick when a `priority_queue` would let stale entries pile up.

---

## Key Takeaways

- `set`, `multiset`, `map`, `multimap` are red-black trees: $\mathcal{O}(\log n)$ everything, sorted iteration, ~40 bytes per `int`.
- `insert` returns `pair<iterator,bool>`; use `.second` to test-and-insert in one $\mathcal{O}(\log n)$ descent.
- On any node-based container, use the **member** `lower_bound`/`upper_bound`/`equal_range`. The free `std::lower_bound` is $\mathcal{O}(n)$ there — a classic TLE that compiles and looks right.
- Ceiling of $x$ is `s.lower_bound(x)` guarded by `== end()`; floor of $x$ is `prev(s.upper_bound(x))` guarded by `== begin()`.
- `ms.erase(x)` deletes **all** copies; `ms.erase(ms.find(x))` deletes one. This is the number-one multiset bug.
- `map::operator[]` inserts a default value when the key is missing; `find`, `at` and `count` never insert.
- `pair` and `tuple` are ordered lexicographically — no comparator needed. A custom `struct` needs a `const` `operator<` with a full tie-break.
- `unordered_*` is $\mathcal{O}(1)$ average, $\mathcal{O}(n)$ worst case; the worst case is reachable by a crafted anti-hash test. Defend with a seeded splitmix64 hash as the third template argument.
- `reserve(n)` before bulk insertion avoids rehashes and iterator invalidation.
- Erase in a loop only as `it = s.erase(it)`; insertion into a node-based container invalidates nothing.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `lower_bound(s.begin(), s.end(), x)` on a `set` | $\mathcal{O}(n)$ via repeated `advance` | `s.lower_bound(x)` |
| 2 | `ms.erase(x)` to remove one copy | Deletes every duplicate | `ms.erase(ms.find(x))` |
| 3 | `ms.erase(ms.find(x))` when absent | Erases `end()` → UB | Guard `it != end()` |
| 4 | `m[k]` for a pure membership test | Inserts a default-constructed entry | `m.find(k)` / `m.count(k)` |
| 5 | Comparator with no tie-break | Distinct elements compare equivalent, silently dropped | Add `if (a.x != b.x) …; return a.y < b.y;` |
| 6 | `operator<` not marked `const` | `set` fails to compile or misbehaves | `bool operator<(const T&) const` |
| 7 | `*it = x` on a `set` | Key is `const`; compile error | `erase(it); insert(x);` |
| 8 | `s.erase(it); ++it;` | Uses a freed node | `it = s.erase(it);` |
| 9 | `for (int x : s) if (bad) s.erase(x);` | Erases by value mid-iteration | Iterator loop with reassignment |
| 10 | Default `unordered_map` on Codeforces | Hackable to $\mathcal{O}(n^2)$ | Seeded `custom_hash` |
| 11 | No `reserve` on `unordered_map` | ~24 rehashes, invalidated iterators | `m.reserve(n)` before inserting |
| 12 | `map<string,int>` with $10^6$ keys | ~64 MB of nodes, MLE | `unordered_map` or a sorted `vector` |
| 13 | Writing `s.begin()->first` for a max | Wrong end; `begin()` is the *min* | `*s.rbegin()` |
| 14 | Assuming `multiset` keeps equal keys in insertion order | Not guaranteed by the standard | Store `pair<value, id>` |
| 15 | `count` on a `multiset` inside a loop | $\mathcal{O}(\log n + k)$ each, $\mathcal{O}(nk)$ total | Cache the count, or use a `map<K,int>` |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Count distinct values in an array | CSES "Distinct Numbers" | Easy |
| 2 | Detect the first duplicate | Any judge, warm-up | Easy |
| 3 | Next greater element to the right | CSES "Next Greater Element" | Easy |
| 4 | Frequency table, print in key order | Codeforces Div. 3 B | Easy |
| 5 | Group anagrams | Codeforces / LeetCode-style | Medium |
| 6 | Two-sum with indices | Any judge | Medium |
| 7 | Sliding-window median with two multisets | CSES "Sliding Window Median" | Medium |
| 8 | Delete one occurrence, keep the max queryable | Codeforces Div. 2 C | Medium |
| 9 | Coordinate compression then count inversions | Codeforces Div. 2 C/D | Medium |
| 10 | Interval merging with `map<int,int>` | Codeforces Div. 2 D | Hard |
| 11 | Hacked `unordered_map` — survive the hack | Codeforces (any Div. 2 hack phase) | Medium |

## Next Up

→ [Topic 14 — STL Part 3 (Stacks, Queue and Priority Queue)](Topic%2014%20-%20STL%20Part%203%20%28Stacks%2C%20Queue%20and%20Priority%20Queue%29.md)