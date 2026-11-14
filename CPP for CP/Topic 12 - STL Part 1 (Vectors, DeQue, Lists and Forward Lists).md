# Topic 12 — STL Part 1: `vector`, `deque`, `list`, `forward_list`

Ninety percent of every competitive programming solution you will write is a `vector`. The other ten percent is knowing exactly when *not* to use one. This chapter builds `vector` from the metal up — memory model, capacity machinery, the amortised-growth proof, the iterator-invalidation rules that silently corrupt answers — then gives a numbers-backed verdict on `deque`, `list` and `forward_list`.

> **Prerequisites:** [Topic 09 — Arrays](Topic%2009%20-%20Arrays%20%281D%2C%202D%20and%203D%29.md) (contiguous memory), [Topic 10 — Memory Management in C++ and Pointers](Topic%2010%20-%20Memory%20Management%20in%20C%2B%2B%20and%20Pointers.md) (heap vs stack), [Topic 11 — Functions, Classes and Lambdas](Topic%2011%20-%20Functions%2C%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md) (templates, references).
> **You will be able to:**
> - Declare a `vector` in all six forms without guessing what the elements contain.
> - Prove `push_back` is $\mathcal{O}(1)$ amortised and use `reserve` to make it worst-case safe.
> - Delete elements in a loop correctly, using all three idioms.
> - Recite the iterator-invalidation table.
> - Build 2D grids and adjacency lists that fit the memory limit.
> - Choose between `vector`, `deque`, `list` and `forward_list` in five seconds.

---

## 1. What the STL Actually Is

Four pieces of machinery, joined by one contract.

```text
        ┌──────────────────────────────────────┐
        │  ALGORITHMS  sort, lower_bound,      │  ← Topic 15
        │  reverse, unique, rotate, accumulate │
        └───────────────┬──────────────────────┘
                        │ speak only iterators
        ┌───────────────┴──────────────────────┐
        │  ITERATORS   begin(), end(), ++, *,  │
        │  random / bidirectional / forward    │
        └───────────────┬──────────────────────┘
                        │ implemented by
        ┌───────────────┴──────────────────────┐
        │  CONTAINERS  vector deque list       │
        │  forward_list set map stack queue    │
        └──────────────────────────────────────┘
```

The design promise is narrow and powerful: **an algorithm never knows what container it walks.** `std::sort` demands random-access iterators; `std::find` needs only forward iterators. That is why `sort(v.begin(), v.end())` compiles for a `vector` and fails for a `list` — the requirement is checked at compile time, not runtime.

Two consequences matter in a contest:

1. **Everything is templated.** No virtual dispatch, no boxing, no runtime type tag. A `vector<int>` has the same layout a raw array would. Abstraction costs compile time, not runtime.
2. **Everything is header-only.** `<vector>` carries the full implementation; nothing is linked. This is why `<bits/stdc++.h>` compiles slowly (~2 MB of headers) yet runs fast — the compiler inlines `push_back` down to a store plus a bounds check.

| Family | Members | Job |
|---|---|---|
| Sequence | `vector`, `deque`, `list`, `forward_list`, `array` | Elements in an order **you** choose |
| Associative | `set`, `map`, `multiset`, `multimap` | Sorted keys, $\mathcal{O}(\log n)$ (Topic 13) |
| Unordered | `unordered_set`, `unordered_map` | Hash table, $\mathcal{O}(1)$ average (Topic 13) |
| Adaptors | `stack`, `queue`, `priority_queue` | One discipline over an existing container (Topic 14) |

Row four are not containers. They wrap one and expose only `push`/`pop`/`front`. That is why `stack` has no iterators: exposing them would let you break the LIFO contract.

---

## 2. `vector` — Declaration and Initialisation

A `vector` is exactly **three pointers** — begin, end-of-size, end-of-capacity. That is 24 bytes on 64-bit, *regardless of element count*. Elements live on the heap.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v;                        // (1) empty: size 0, capacity 0
    vector<int> a(5);                     // (2) FIVE zeros — not "capacity 5"
    vector<int> b(5, -1);                 // (3) five copies of -1
    vector<int> c = {1, 2, 3};            // (4) initialiser list
    vector<int> d(c);                     // (5) deep copy of c
    vector<int> e(c.begin(), c.end());    // (6) from another container's iterators

    cout << v.size() << ' ' << a.size() << ' ' << b.size() << ' '
         << c.size() << ' ' << d.size() << ' ' << e.size() << '\n';
    cout << a[0] << ' ' << a[4] << ' ' << b[0] << '\n';
    return 0;
}
```

```text
0 5 5 3 3 3
0 0 -1
```

Form (2) value-initialises — for `int` that means zero, for `string` five empty strings. If you wanted capacity without elements, write `vector<int> a; a.reserve(5);`. **This one confusion produces more wrong answers than any other line in the chapter.**

Two more traps:

```cpp
vector<int> v();     // a FUNCTION declaration, not an empty vector
vector<int> v;       // what you meant
```

```cpp
int n = 3, m = 4;
vector<vector<int>> good(n, vector<int>(m, 0));   // n x m of zeros
// vector<vector<int>> bad(n, m);                 // compile error — see §9
```

Use short aliases — `using vi = vector<int>;`, `using pii = pair<int,int>;`, `using vpii = vector<pii>;` — they prevent miscounted angle brackets. (The `>>` in `vector<vector<int>>` was a syntax error before C++11; you will meet old code writing `> >`.)

---

## 3. Size, Capacity, and Growth

`size()` counts elements that exist; `capacity()` counts how many fit before reallocation.

```cpp
vector<int> v;
for (int i = 0; i < 9; i++) {
    v.push_back(i);
    cout << v.size() << '/' << v.capacity() << ' ';
}
```

```text
1/1 2/2 3/4 4/4 5/8 6/8 7/8 8/8 9/16
```

Capacities go $1,2,4,8,16,\dots$: libstdc++ **doubles**. MSVC grows by $1.5$: $1,2,3,4,6,9,\dots$. Both are legal — the standard mandates only amortised $\mathcal{O}(1)$. Never write code that depends on the factor.

| Call | Effect on `size()` | Effect on `capacity()` | Cost | Use when |
|---|---|---|---|---|
| `v.size()` / `v.empty()` | — | — | $\mathcal{O}(1)$ | Counting; prefer `empty()` for the zero test |
| `v.capacity()` | — | — | $\mathcal{O}(1)$ | Diagnosing reallocation |
| `v.reserve(k)` | unchanged | $\ge k$ | $\mathcal{O}(n)$ if $k>$cap | Final size known up front |
| `v.resize(k)` | becomes $k$ | $\ge k$ | $\mathcal{O}(\lvert k-n\rvert)$ | You want $k$ *real* elements |
| `v.clear()` | 0 | **unchanged** | $\mathcal{O}(n)$ | Reusing the buffer next test case |
| `v.shrink_to_fit()` | unchanged | $\approx$ size | $\mathcal{O}(n)$ | Release RAM after a big erase |

Three subtleties: `resize` **down keeps capacity**; `reserve` **never shrinks** (it is a lower bound); `shrink_to_fit` is a *request*, not a guarantee — `vector<int>().swap(v)` forces it.

**The multi-test-case pattern** — the highest-value capacity trick in CP:

```cpp
int T; cin >> T;
vector<int> v;
v.reserve(200005);              // ONE allocation, outside the loop
while (T--) {
    int n; cin >> n;
    v.clear();                  // size 0, capacity intact — no malloc
    for (int i = 0; i < n; i++) { int x; cin >> x; v.push_back(x); }
}
```

With $T = 10^4$ small cases, the naive version performs thousands of `malloc`/`free` pairs; this performs one.

---

## 4. Element Access

| Expression | Checked? | Out of range | Cost |
|---|---|---|---|
| `v[i]` | No | Undefined behaviour | One load |
| `v.at(i)` | **Yes** | Throws `std::out_of_range` | Load + compare + throw path |
| `v.front()` / `v.back()` | No | UB if empty | One load |
| `v.data()` | No | `nullptr`-safe pointer to first element | $\mathcal{O}(1)$ |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {10, 20, 30};
    cout << v[0] << ' ' << v.at(1) << ' ' << v.front() << ' ' << v.back() << '\n';
    cout << v.data()[2] << '\n';                 // raw pointer view
    try { cout << v.at(3) << '\n'; }
    catch (const out_of_range &e) { cout << "caught" << '\n'; }
    return 0;
}
```

```text
10 20 10 30
30
caught
```

**Use `[]` in hot loops, `at()` when hunting an index bug.** Better still, compile with `-D_GLIBCXX_DEBUG -fsanitize=address,undefined`: 2–10x slower, but it turns "wrong answer on test 47" into a line number. And `memset(v.data(), 0, v.size()*sizeof(int))` is legal only because `vector` is contiguous — undefined on a `deque`, which has no `data()` at all.

---

## 5. Modifiers

| Call | Effect | Cost |
|---|---|---|
| `push_back(x)` / `emplace_back(args)` | Append | Amortised $\mathcal{O}(1)$ |
| `pop_back()` | Destroy last | $\mathcal{O}(1)$ |
| `insert(pos, x)` | Insert before `pos`, shift tail right | $\mathcal{O}(n)$ |
| `insert(pos, cnt, x)` / `insert(pos, first, last)` | Multiple / range | $\mathcal{O}(n + \text{added})$ |
| `erase(pos)` / `erase(first, last)` | Remove, shift tail left | $\mathcal{O}(n - k)$ |
| `assign(cnt, x)` | Replace contents | $\mathcal{O}(n + cnt)$ |
| `swap(u)` | Exchange three pointers, no element moves | $\mathcal{O}(1)$ |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {1, 2, 3};
    v.emplace_back(4);                    // {1,2,3,4}
    v.insert(v.begin() + 1, 99);          // {1,99,2,3,4}
    v.erase(v.begin() + 1);               // {1,2,3,4}
    v.insert(v.begin(), 2, 0);            // {0,0,1,2,3,4}
    v.erase(v.begin(), v.begin() + 2);    // {1,2,3,4}
    for (int x : v) cout << x << ' ';
    cout << '\n';

    vector<int> w = {7, 8};
    v.swap(w);                            // O(1): three pointers exchanged
    for (int x : v) cout << x << ' '; cout << "| ";
    for (int x : w) cout << x << ' '; cout << '\n';
    return 0;
}
```

```text
1 2 3 4
7 8 | 1 2 3 4
```

### `push_back` vs `emplace_back`

Both append. `push_back` takes a *value*; `emplace_back` takes *constructor arguments* and builds the object in place, skipping a temporary.

```cpp
struct Point { int x, y; Point(int a, int b) : x(a), y(b) {} };

vector<Point> p;
p.push_back(Point(1, 2));   // temporary + move
p.emplace_back(1, 2);       // constructed in the slot

vector<pair<int,int>> edges;
edges.emplace_back(u, v);   // cleaner than push_back({u, v})
```

For `int` they compile identically. Prefer `emplace_back` for `string`, `pair` and `tuple`. One caution: `emplace_back`'s arguments are forwarded greedily, so on a `vector<vector<int>>`, `emplace_back(5)` builds a **row of five zeros**, not an empty row.

---

## 6. Why `push_back` Is Amortised $\mathcal{O}(1)$

Growing by one would copy $1+2+\dots+n = \Theta(n^2)$ elements — for $n = 10^6$ that is $5\times10^{11}$ moves, minutes not milliseconds.

Doubling fixes it. Reallocation happens on pushes $1,2,3,5,9,17,\dots$ and the $k$-th reallocation copies $2^{k-1}$ elements, so

$$C(n) = \sum_{k=1}^{\lceil \log_2 n \rceil} 2^{k-1} = 2^{\lceil \log_2 n \rceil} - 1 < 2n .$$

Total copies stay **below $2n$**, and the amortised cost is

$$\frac{C(n) + n}{n} < 3 = \mathcal{O}(1).$$

The accounting argument: each element is copied at most twice more — once into a buffer of size $2^k$, once into $2^{k+1}$ — so charge 3 credits per push (1 for the write, 2 saved) and every reallocation is prepaid. In general a growth factor $g$ costs $n\cdot\frac{g}{g-1}$ copies: $2n$ at $g=2$, $3n$ at $g=1.5$.

**Amortised is not worst-case.** The single `push_back` that triggers reallocation is $\mathcal{O}(n)$. If that spike must not happen, `reserve`.

| Workload, $n = 10^7$ `int` | Time (`-O2`, modern laptop) | Reallocations |
|---|---|---|
| `push_back` with `reserve(n)` | ~25–35 ms | 1 |
| `push_back` without `reserve` | ~55–80 ms | ~24 |
| `resize(n)` then index-assign | ~25–35 ms | 1 |

The 2x gap is the amortisation working, not failing — but 40 ms is 4% of a 1-second limit, free to reclaim.

Iterating those same $10^7$ elements takes ~8 ms for a `vector`, ~35 ms for a `deque` and ~180 ms for a `list` — identical $\mathcal{O}(n)$, 20x wall-clock difference. That gap is the whole subject of §11.

*Contest story: a Div. 2 C left `vector<int> v;` inside a loop over $2\times10^5$ test cases. Every case malloc'd a fresh buffer; runtime 1.4 s against a 1 s limit. Hoisting the declaration and adding `reserve(10)` gave 0.5 s with zero algorithmic change.*

---

## 7. The `erase`-in-a-Loop Trap

The bug that ends more contest runs than any other container mistake. It compiles, runs, and produces plausible — wrong — output.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {2, 4, 6, 8};
    for (int i = 0; i < (int)v.size(); i++)
        if (v[i] % 2 == 0) v.erase(v.begin() + i);
    for (int x : v) cout << x << ' ';
    cout << '\n';
    return 0;
}
```

```text
4 8
```

| Step | `i` | `v` before | `v[i]` | After erase |
|---|---|---|---|---|
| 1 | 0 | `{2,4,6,8}` | 2 | `{4,6,8}` |
| 2 | 1 | `{4,6,8}` | 6 | `{4,8}` |
| 3 | 2 | `{4,8}` | — | loop ends |

`erase` shifts the tail left, so the element that *was* at `i+1` now sits at `i` — and `i++` steps over it. **After an erase, do not increment `i`.**

### Idiom A — index without increment

```cpp
for (int i = 0; i < (int)v.size(); ) {
    if (v[i] % 2 == 0) v.erase(v.begin() + i);   // i stays put
    else               i++;
}
```

Correct but $\mathcal{O}(n^2)$: every `erase` shifts the tail. At $n=10^5$ with many erases that is $10^{10}$ moves — a guaranteed TLE.

### Idiom B — iterator reassignment

```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);   // erase returns the NEXT valid iterator
    else              ++it;
}
```

`vector::erase` returns the iterator following the erased element. Never write `v.erase(it); ++it;` — `it` is dangling and `++it` is undefined behaviour.

### Idiom C — erase–remove (the default)

```cpp
v.erase(remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }), v.end());
```

`remove_if` removes nothing. It **compacts survivors to the front**, preserving order, returning an iterator to the first dead slot; one `erase` then truncates the tail — $\mathcal{O}(n)$ regardless of how many elements die. The array goes `[1][2][3][4][5][6]` → `[1][3][5][?][?][?]` (size still 6) → `[1][3][5]` (size 3, capacity 6).

`remove` (no `_if`) deletes every copy of a value: `v.erase(remove(v.begin(), v.end(), 7), v.end());`

| Situation | Use | Cost |
|---|---|---|
| Remove by predicate | `erase(remove_if(...), end())` | $\mathcal{O}(n)$ |
| Remove one known position | `v.erase(v.begin()+k)` | $\mathcal{O}(n-k)$ |
| Extra work per survivor | `it = v.erase(it)` loop | $\mathcal{O}(n)$ + shifts |
| Most elements die, heavy elements | build `keep`, then `v.swap(keep)` | $\mathcal{O}(n)$, best locality |

`remove`, `remove_if` and `unique` are [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort%2C%20Binary%20Search%2C%20Min%20Max%2C%20Array%20Manipulation%2C%20Permutations%2C%20Math%20%26%20Accumulators%29.md)'s territory; here you need only the two-line pattern.

---

## 8. Iterator Invalidation

An iterator is a dressed-up pointer. When a `vector` reallocates, the old buffer is freed and every iterator into it — including one you saved three lines ago — dangles. Using it is undefined behaviour, not an exception.

| Operation | Invalidates | Why |
|---|---|---|
| `push_back` / `emplace_back` | **All** on reallocation; else only `end()` | Buffer swapped |
| `pop_back` | Erased element and `end()` | Destroyed |
| `insert` / `emplace` | **All** on reallocation; else from the insertion point on | Tail shifts right |
| `erase` | From the erase point on (plus `end()`) | Tail shifts left |
| `resize` (grow) | All on reallocation; else `end()` | Appends |
| `resize` (shrink) | Erased elements and `end()` | Destructors run |
| `reserve` / `shrink_to_fit` | **All** if capacity changes | New buffer |
| `clear` / `assign` | **All** | Contents replaced |
| `swap` | None — but they now refer to the *other* container | Pointers exchanged |

```cpp
// TRAP 1: reference across growth
int &r = v[0];
v.push_back(4);        // may reallocate
cout << r;             // UB if it did

// TRAP 2: iterator across reserve
auto it = v.begin();
v.reserve(1000);       // reallocates
cout << *it;           // UB

// TRAP 3: end() cached in the loop condition
for (auto it = v.begin(); it != v.end(); ++it)
    v.push_back(*it);  // both it and end() die
```

Trap 3 bites even without reallocation, because `end()` moves. Fix by capturing the count:

```cpp
size_t n = v.size();
for (size_t i = 0; i < n; i++) v.push_back(v[i]);   // intentional doubling
```

**Escape hatch:** if you only need indices, use indices. `for (int i = 0; i < (int)v.size(); i++)` re-reads `size()` each iteration and is immune to iterator invalidation — though not to the shifting bug of §7.

---

## 9. 2D vectors, Locality, and `vector<bool>`

Both shapes below come up in every graph and grid problem: a dense rectangle, and a ragged adjacency list.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 3, m = 4;
    vector<vector<int>> g(n, vector<int>(m, 0));   // n rows of m zeros
    g[1][2] = 7;
    for (auto &row : g) {
        for (int x : row) cout << x << ' ';
        cout << '\n';
    }

    vector<vector<int>> adj(n);                    // ragged: adjacency list
    adj[0] = {1, 2};
    adj[1].push_back(3);
    cout << adj[0].size() << ' ' << adj[1].size() << ' ' << adj[2].size() << '\n';
    return 0;
}
```

```text
0 0 0 0
0 0 7 0
0 0 0 0
2 1 0
```

### Why `g(n, m)` does not compile

The fill constructor is `vector(size_type n, const T& value)`. `T` is `vector<int>`, and `vector<int>`'s size constructor is **`explicit`**, so `m` (an `int`) will not implicitly convert. You would be asking for "n copies of the integer 4". Write `g(n, vector<int>(m, 0))` — construct **one prototype row**, copy it $n$ times. That copy is also why every row has the same length.

Use `for (auto &row : g)` (note the `&`) — without it you copy each row.

### The cost

| Layout | Allocations | Row addresses | Column walk `g[i][j]`, varying `i` |
|---|---|---|---|
| `vector<vector<int>> g(n, vector<int>(m))` | $n+1$ | scattered | one pointer chase **per row** |
| `vector<int> g(n*m)` with `g[i*m+j]` | 1 | one block | pure arithmetic, cache-friendly |
| `vector<array<int,4>> g(n)` | 1 | one block | row inline |

For $1000\times1000$, the nested version does 1001 `malloc`s and may scatter rows megabytes apart; the flat version is one contiguous 4 MB block. Still prefer nested for readability — switch to flat (`vector<int> flat(n*m); flat[i*m+j]`) when $n,m \ge 2000$, when the traversal is column-ish, or when $n$ is huge with tiny $m$ (a million allocations).

### `vector<bool>` — the container that is not a container

`vector<bool>` is a **specialisation** that packs bits, eight per byte. Its elements are not addressable objects.

```cpp
vector<bool> vb(5, true);
vb[0] = false;
auto x = vb[1];                       // std::_Bit_reference, NOT bool
cout << boolalpha << x << '\n';       // true
// bool *p = &vb[1];                  // error: no address of a bit
// bool &r = vb[1];                   // error: cannot bind bool& to a proxy
cout << sizeof(vb[0]) << ' ' << vb.size() << '\n';   // 16 5
```

`sizeof(vb[0])` printing `16` says everything: `[]` hands back a temporary proxy class that overloads `operator bool` and `operator=`. Everything else about it is a lie:

| You expect | Reality |
|---|---|
| `auto b = vb[i]` is `bool` | It is `vector<bool>::reference` |
| `bool* p = &vb[i]` | Compile error |
| `bool& r = vb[i]`, `auto& r = vb[i]` | Compile error |
| Pass to `f(bool&)` | Compile error |
| Pass to a template expecting `T&` | Breaks |

**Feature:** $10^8$ booleans cost 12.5 MB instead of 100 MB — the difference between AC and MLE on a 64 MB limit. **Bug:** it poisons generic code and slows every access with bit arithmetic. If you need bit-packing, prefer `bitset<N>` (fixed size, Topic 04) or `vector<char>`; if you need speed, `vector<char>` is often *faster* despite being 8x larger.

---

## 10. `deque` — Double-Ended Queue

$\mathcal{O}(1)$ push/pop at **both** ends, plus $\mathcal{O}(1)$ random access — via a **map of fixed-size blocks**.

```text
   map:  ┌────┬────┬────┬────┬────┐     (array of block pointers)
         │    │ p1 │ p2 │ p3 │    │
         └────┴─┬──┴─┬──┴─┬──┴────┘
        ┌────────▼┐ ┌─▼─────┐ ┌▼──────┐
blocks: │[a][b]  │ │[c][d] │ │[e][f] │   each block = 512 bytes (libstdc++)
        └────────┘ └───────┘ └───────┘
          ^begin                    ^end
```

Pushing at the front writes into the spare slot before `a`; when that block fills, a new block is allocated and a pointer is pushed onto the front of the map. **No element ever moves.**

```cpp
deque<int> dq;
dq.push_back(3); dq.push_back(4);     // {3,4}
dq.push_front(2); dq.push_front(1);   // {1,2,3,4}
cout << dq.front() << ' ' << dq.back() << ' ' << dq[2] << '\n';   // 1 4 3
dq.pop_front(); dq.pop_back();        // {2,3}
```

### The non-contiguity tax

| Property | `vector` | `deque` |
|---|---|---|
| `push_front` / `pop_front` | $\mathcal{O}(n)$ | $\mathcal{O}(1)$ |
| `push_back` / `pop_back` | Amortised $\mathcal{O}(1)$ | $\mathcal{O}(1)$ |
| `d[i]` | One load | Two loads (map → block → element) |
| Contiguous / `data()` | Yes | **No / no `data()`** |
| Iterator | Raw pointer | Class with 4 pointers |
| `sizeof` (64-bit) | 24 B | 80 B |
| Iteration | Baseline | 2–4x slower |

A `deque` iterator is a small object, not a pointer: every `++`, `*` and comparison runs code. And each 512-byte block wastes up to 511 bytes of slack, so tiny-element deques are memory-hungry.

### Why `stack` and `queue` default to `deque`

Both adaptors declare `class Container = deque<T>` as their second template parameter. It is contract, not speed: `queue` needs `pop_front`, which `vector` cannot do in $\mathcal{O}(1)$; `stack` needs only back operations, so `stack<int, vector<int>>` is often marginally faster when the max depth is known — but `deque` never pays a reallocation spike, which is why it is the default. (Topic 14 covers both.)

| Your workload | Pick | Reason |
|---|---|---|
| Append, then read in order | `vector` | Contiguous, fastest scan |
| Append + random access in a hot loop | `vector` | One load, not two |
| Push/pop both ends | `deque` | Both ends $\mathcal{O}(1)$ |
| Sliding-window max/min, BFS frontier | `deque` | Needs `pop_front` |
| Must call `data()` / pass to C | `vector` | `deque` has none |
| Unknown size, no spikes allowed | `deque` | Never copies on growth |

---

## 11. `list` — Doubly Linked

Insert/erase **given an iterator** is $\mathcal{O}(1)$ — pointer surgery. But *finding* that iterator is $\mathcal{O}(n)$, so `list` never turns an $\mathcal{O}(n)$ search into $\mathcal{O}(1)$.

```cpp
list<int> l = {10, 20, 30};
auto it = l.begin(); ++it;            // -> 20
l.insert(it, 15);                     // {10,15,20,30}  O(1)
l.erase(it);                          // {10,15,30}     O(1), it dangles
l.push_front(5); l.push_back(99);     // {5,10,15,30,99}
l.reverse();                          // {99,30,15,10,5}
l.sort(); l.unique();                 // members, not std::
```

### `splice` — the one genuinely unique power

`splice` moves nodes between lists by rewiring pointers. No element is copied, so it is $\mathcal{O}(1)$.

```cpp
list<int> a = {1, 2, 3, 4, 5}, b;
auto mid = a.begin();
advance(mid, 3);                         // -> 4
b.splice(b.begin(), a, mid, a.end());    // move {4,5} to b — O(1)
// a == {1,2,3}, b == {4,5}
```

A `vector` cannot do this at any price. In competitive programming you almost never need it.

### Why `std::sort` refuses to compile on a `list`

`std::sort` requires **random-access** iterators because introsort indexes into the middle (`first + n/2`). A `list` iterator supports only `++`/`--`. Hence `list` ships members: `sort()` (bottom-up merge sort, stable, $\mathcal{O}(n\log n)$), `reverse()`, `unique()` (which *actually erases*), `remove()`, `merge()`. Note the asymmetry: `std::unique` on a `vector` only compacts and needs the erase dance.

### The cache-miss tax

| Metric, $10^6$ `int` | `vector` | `list` |
|---|---|---|
| Memory | 4 MB | ~32 MB |
| Full traversal | ~1–2 ms | ~15–25 ms |
| Insert at iterator | $\mathcal{O}(n)$ move (~0.2 ms) | $\mathcal{O}(1)$ (~0.1 µs) |
| Insert by value in middle | $\mathcal{O}(n)$ search too | $\mathcal{O}(n)$ search too |
| `sort` | ~80 ms | ~180 ms |

Look at the last two rows: even where `list` wins theoretically, the *search* to reach that iterator dominates. And a middle `vector` insert is a `memmove` of contiguous bytes at ~10 GB/s, not $n$ random writes.

> **Verdict: in competitive programming you almost never use `list`.** Use it only when you hold long-lived iterators across many insert/erase operations on a small container, or you need `splice`. Otherwise `vector` wins on constants, `deque` when both ends matter, `set` when you need order plus $\mathcal{O}(\log n)$.

---

## 12. `forward_list` — Singly Linked

One pointer per node instead of two — and the entire "insert before" API disappears, because you cannot walk backwards to find the predecessor.

- No `push_back`, `pop_back`, or `back()`.
- `insert_after(pos, x)` — you give the node *before* the insertion point.
- `erase_after(pos)` — you give the node *before* the victim.
- `before_begin()` — a valid, non-dereferenceable sentinel that is the predecessor of `begin()`.

```cpp
forward_list<int> fl = {10, 20, 30};
fl.push_front(5);                        // {5,10,20,30}
fl.insert_after(fl.begin(), 7);          // {5,7,10,20,30}
auto it = fl.begin(); advance(it, 2);    // -> 7
fl.erase_after(it);                      // removes 10  -> {5,7,20,30}
fl.insert_after(fl.before_begin(), 1);   // prepend     -> {1,5,7,20,30}
```

### Why the API is awkward

Every singly-linked operation needs the predecessor, and the library cannot produce one for `begin()`. Hence the `before_begin()` sentinel. Since C++ has no "iterator before this one", loops must look ahead:

```cpp
auto prev = fl.before_begin(), cur = fl.begin();
while (cur != fl.end()) {
    if (*cur % 2 == 0) cur = fl.erase_after(prev);   // prev stays put
    else { prev = cur; ++cur; }
}
```

Compare that two-iterator dance with the `vector` erase–remove one-liner.

> **Verdict: `forward_list` is essentially never used in competitive programming.** Its 8-bytes-per-node saving is irrelevant against the `vector`'s 8x memory advantage, and the `insert_after`/`before_begin` API doubles the fiddliness. Its real home is intrusive lists inside allocators and kernels.

---

## 13. Decision Matrix

| Operation | `vector` | `deque` | `list` | `forward_list` |
|---|---|---|---|---|
| `push_back` | $\mathcal{O}(1)^*$ | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | — |
| `pop_back` | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | — |
| `push_front` / `pop_front` | $\mathcal{O}(n)$ | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ |
| Random access `c[i]` | $\mathcal{O}(1)$ fast | $\mathcal{O}(1)$ slow | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ |
| Insert/erase at iterator | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ |
| Search by value | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ |
| Iterate all | Fastest | Medium | Slow | Slow |
| Iterator stability | Fragile | Ends stable | **Stable** | **Stable** |

$^*$ Amortised; an individual call may be $\mathcal{O}(n)$.

| Container | Pick this when… |
|---|---|
| **`vector`** | …you need a resizable array to index, scan, sort or iterate. **The default — 95% of the time.** |
| **`deque`** | …you need $\mathcal{O}(1)$ at both ends *and* `d[i]`: sliding window, BFS, monotonic deque. |
| **`list`** | …you hold long-lived iterators across many inserts/erases, or need `splice`. |
| **`forward_list`** | …you need minimal per-node memory and only ever insert/erase after a known node. |

## 14. Memory and Size (libstdc++, x86-64)

| Type | `sizeof` container | Per-element overhead | $10^6$ `int` |
|---|---|---|---|
| `vector<int>` | 24 B | 0 (up to 100% growth slack) | ~4 MB |
| `array<int,1000>` | 4000 B | 0 | — |
| `deque<int>` | 80 B | 0 + 512 B blocks | ~4 MB + map |
| `list<int>` | 24 B | 16 B + 8–16 B header | ~32 MB |
| `forward_list<int>` | 8 B | 8 B + header | ~24 MB |
| `set<int>` | 48 B | 32 B + header | ~48 MB (Topic 13) |
| `unordered_set<int>` | 56 B | 16 B + buckets | ~32 MB (Topic 13) |

Translation for a 256 MB limit: $6\times10^7$ `int`s fit in a `vector`; the same count in a `list` needs 2 GB and dies. That is not micro-optimisation — it is the verdict.

---

## 15. CP Recipes

Each block below is a fragment meant to be dropped into `int main()`.

**1 — Read $n$ elements, fast.** `for (int &x : a)` writes straight into the vector, avoiding a copy.

```cpp
int n; cin >> n;
vector<int> a(n);
for (int &x : a) cin >> x;
for (int x : a) cout << x << ' ';
```

**2 — Prefix sums.** Use `long long`: $10^5$ elements of $10^9$ sum to $10^{14}$.

```cpp
vector<int> a = {3, 1, 4, 1, 5, 9, 2, 6};
int n = a.size();
vector<long long> pre(n + 1, 0);
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + a[i];

int l = 2, r = 5;
cout << pre[r + 1] - pre[l] << '\n';     // 19  (sum of a[2..5])
```

**3 — Adjacency list.** $\mathcal{O}(n+m)$ memory, unlike an $n\times n$ matrix.

```cpp
vector<vector<int>> adj(n);
auto add = [&](int u, int v) { adj[u].push_back(v); adj[v].push_back(u); };
add(0, 1); add(0, 2); add(1, 3);
for (int u = 0; u < n; u++)
    for (int v : adj[u]) cout << u << " -> " << v << '\n';
```

**4 — Coordinate compression.** Maps values to $0\ldots k-1$ so you can index a plain array instead of a `map`.

```cpp
vector<int> a = {100, 5, 100, 42, 7, 5};
vector<int> s = a;
sort(s.begin(), s.end());                                   // Topic 15
s.erase(unique(s.begin(), s.end()), s.end());               // Topic 15
for (int &x : a)
    x = lower_bound(s.begin(), s.end(), x) - s.begin();      // Topic 15
// a == {3, 0, 3, 2, 1, 0}
```

**5 — Sliding-window maximum.** Keep indices in a `deque` with strictly decreasing values; the front is always the window max. $\mathcal{O}(n)$ total.

```cpp
vector<int> a = {1, 3, -1, -3, 5, 3, 6, 7};
int k = 3;
deque<int> dq;                  // indices
vector<int> best;
for (int i = 0; i < (int)a.size(); i++) {
    while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
    dq.push_back(i);
    if (dq.front() <= i - k) dq.pop_front();       // evict expired index
    if (i >= k - 1) best.push_back(a[dq.front()]);
}
// best == {3, 3, 5, 5, 6, 7}
```

**6 — BFS with a `deque`.** Swap `pop_front` for `pop_back` and it becomes DFS.

```cpp
vector<vector<int>> adj = {{1, 2}, {3}, {3}, {}};
vector<int> dist(adj.size(), -1);
deque<int> q{0};
dist[0] = 0;
while (!q.empty()) {
    int u = q.front(); q.pop_front();
    for (int v : adj[u])
        if (dist[v] == -1) { dist[v] = dist[u] + 1; q.push_back(v); }
}
// dist == {0, 1, 1, 2}
```

**7 — Erase-remove.** The wrong loop leaves `2` behind; the idiom does not.

```cpp
vector<int> a = {1, 2, 2, 3, 2, 4, 2};
vector<int> bad = a;
for (int i = 0; i < (int)bad.size(); i++)
    if (bad[i] == 2) bad.erase(bad.begin() + i);     // {1,2,3,4,2}  WRONG

a.erase(remove(a.begin(), a.end(), 2), a.end());     // {1,3,4}     right
```

**8 — Reverse and rotate.** `rotate` runs in $\mathcal{O}(n)$ with zero allocation; `k %= n` guards $k > n$.

```cpp
vector<int> v = {1, 2, 3, 4, 5};
reverse(v.begin(), v.end());                 // {5,4,3,2,1}

int k = 2;
k %= (int)v.size();
rotate(v.begin(), v.begin() + k, v.end());   // left by 2 -> {3,2,1,5,4}
```

---

## Key Takeaways

- A `vector` is three pointers (24 B on 64-bit) plus a heap buffer; `clear()` frees elements but never capacity.
- `vector<int> v(5)` is five zeros; `v.reserve(5)` is empty-with-room; `vector<int> v()` is a function declaration.
- Doubling copies fewer than $2n$ elements in total, so `push_back` is $\mathcal{O}(1)$ amortised — but one call may still be $\mathcal{O}(n)$. `reserve` removes the spike.
- `erase` inside `for (i...)` skips the element that shifts into slot `i`. Default to `erase(remove_if(...), end())`.
- `v.erase(it)` returns the next valid iterator: always `it = v.erase(it)`, never `v.erase(it); ++it;`.
- Any growth, `insert`, `reserve`, `resize` or `shrink_to_fit` that reallocates invalidates **all** iterators, pointers and references.
- `g(n, m)` does not compile for a 2D grid because `vector<int>`'s size constructor is `explicit`; use `g(n, vector<int>(m, 0))`.
- `vector<bool>` packs bits and returns a proxy; it saves 8x memory and breaks `bool&`/`auto&`/templates. Prefer `bitset` or `vector<char>`.
- `deque` is $\mathcal{O}(1)$ at both ends, random-access, non-contiguous, no `data()` — the default under `stack` and `queue`.
- `list`/`forward_list` give stable iterators but lose on cache and memory; in CP you essentially never use them.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `vector<int> v();` | Parsed as a function declaration | `vector<int> v;` |
| 2 | Reading `v(5)` as capacity | It is five zero elements | `v.reserve(5)` for capacity only |
| 3 | No `reserve` across $T$ cases | Thousands of malloc/free pairs | Declare + `reserve` outside the loop, `clear()` inside |
| 4 | `if (bad) v.erase(v.begin()+i)` in a `for` | Skips the shifted element | `erase(remove_if(...), end())` |
| 5 | `v.erase(it); ++it;` | `it` is invalidated | `it = v.erase(it);` |
| 6 | Reference held across `push_back` | Reallocation frees the buffer | `reserve` first, or re-acquire |
| 7 | `vector<vector<int>> g(n, m);` | Fill value must be `vector<int>` | `g(n, vector<int>(m, 0))` |
| 8 | `vector<int>` for prefix sums | Overflows near $2.1\times10^9$ | `vector<long long>` |
| 9 | `for (auto b : vb)` on `vector<bool>` with `f(bool&)` | Proxy has no address | Take by value; or use `vector<char>` |
| 10 | $n\times n$ matrix for a sparse graph | $\mathcal{O}(n^2)$ memory, MLE | `vector<vector<int>> adj(n)` |
| 11 | Passing `deque` to an `int*` API | Non-contiguous, no `data()` | Copy into a `vector` |
| 12 | `sort(l.begin(), l.end())` on a `list` | Needs random-access iterators | `l.sort()` |
| 13 | `v.back()` on an empty vector | UB, garbage read | Guard with `!v.empty()` |
| 14 | Relying on `shrink_to_fit` | Standard says "request" | `vector<int>().swap(v)` |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Read $n$ ints, print reversed | Warm-up | Easy |
| 2 | Range sum queries via prefix sums | CSES "Static Range Sum Queries" | Easy |
| 3 | Delete all occurrences of `x` in place | Any judge | Easy |
| 4 | Build adjacency list, print degrees | Codeforces Div. 3 A/B | Easy |
| 5 | Rotate array left by $k$ with no extra array | AtCoder ABC | Easy |
| 6 | Sliding-window maximum of size $k$ | CSES "Sliding Window Maximum" | Medium |
| 7 | Compress coordinates, then count distinct | Codeforces Div. 2 B | Medium |
| 8 | Spiral traversal of a `vector<vector<int>>` | Codeforces Div. 3 | Medium |
| 9 | Grid BFS with a `deque` frontier | CSES "Counting Rooms" | Medium |
| 10 | Offline dynamic array with insert/erase | Codeforces Div. 2 C | Hard |
| 11 | $10^8$-boolean sieve under a 64 MB limit | Any judge | Medium |

## Next Up

→ [Topic 13 — STL Part 2 (Set & Multiset, Map & MultiMap, Unordered Set and Unordered Maps)](Topic%2013%20-%20STL%20Part%202%20%28Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps%29.md)