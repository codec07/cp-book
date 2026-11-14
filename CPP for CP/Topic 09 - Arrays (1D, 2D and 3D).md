# Topic 09 — Arrays (1D, 2D and 3D)

An array is the only data structure whose performance you can predict without thinking: address $i$ is `base + i * sizeof(T)`, one multiply and one load. That is why every algorithm you will meet — prefix sums, DP tables, segment trees, adjacency matrices — is an array underneath. This chapter covers how arrays actually live in memory, why a local array of $10^6$ elements crashes while the same array declared globally does not, how to budget memory against a 256 MB limit, and the handful of array patterns that solve a surprising fraction of problems on their own.

> **Prerequisites:** [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems,%20Variables%20and%20Datatypes.md), [Topic 07 — Loops & 'Auto'](Topic%2007%20-%20Loops%20&%20'Auto'.md), [Topic 08 — Useful Math Functions & Operations on Data](Topic%2008%20-%20Useful%20Math%20functions%20&%20Operations%20on%20Data.md)
> **You will be able to:**
> - Declare, initialise and index arrays without the three classic UB traps.
> - Choose between local, global and `static` storage by reasoning about the stack limit.
> - Compute from scratch whether an array of $N$ elements fits in a given memory limit.
> - Clear arrays correctly with `memset` and `fill`, and know when `memset` lies.
> - Traverse 2D and 3D arrays in cache-friendly order and quantify the difference.
> - Build and query 1D/2D prefix sums and difference arrays.
> - Recognise the ten standard array recipes: Kadane, Boyer–Moore, two-pointer reverse, rotation, dedupe-in-place, and friends.

---

## 1. What an array is

An array is a **contiguous block** of $N$ objects of one type, all the same size, laid out back to back with no gaps and no per-element metadata.

```cpp
int a[5] = {10, 20, 30, 40, 50};
```

```text
index:     0     1     2     3     4
value:    10    20    30    40    50
address: base  +4    +8   +12   +16      (sizeof(int) = 4)
```

Two consequences follow immediately, and everything else in this chapter is a corollary:

1. **Random access is $\mathcal{O}(1)$.** `a[i]` compiles to a load from `base + i*4`. There is no search, no pointer chasing, no hashing. This is the fastest access pattern a CPU has.
2. **`sizeof(a)` is known at compile time** — for a real array, not a pointer. It is $N \times \texttt{sizeof(T)}$ exactly, with no overhead. A `vector` stores three pointers; an array stores nothing but the elements.

There is no bounds checking. `a[5]` on a five-element array compiles, runs, and reads whatever happens to be at `base + 20`. It might be another variable, it might be your return address, it might be unmapped memory and give you a `SIGSEGV`. This is undefined behaviour: the standard permits *anything*, including "works on my machine and fails on the judge".

| Property | C array | `std::vector` |
|---|---|---|
| Size known at | compile time (or VLA, non-standard) | run time |
| Overhead per array | 0 bytes | 24 bytes (3 pointers) |
| Bounds checking | none | none with `[]`, checked with `at()` |
| Resizable | no | yes |
| `sizeof` | $N \times$ element size | 24, always |
| Lives on | stack / global / heap (via `new`) | heap |
| Cache behaviour | ideal, contiguous | ideal, contiguous |
| CP verdict | fastest, use for fixed $N$ | default choice, see [Topic 12](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md) |

*Contest story: an $n \le 2 \times 10^5$ problem, tight 1 s limit. The `vector`-based solution TLE'd by 80 ms; changing the two hot arrays to global `int` arrays compiled to identical assembly but removed the indirection the compiler could not prove away. The same algorithm, 12% faster. Fixed-size arrays are not just tradition — they let the optimiser see everything.*

---

## 2. Declaration and initialisation

```cpp
int a[10];                    // uninitialised: garbage on the stack
int b[10] = {};               // all zeros (value-initialised)
int c[10] = {1, 2, 3};        // 1, 2, 3, 0, 0, 0, 0, 0, 0, 0 — rest are zero
int d[] = {1, 2, 3, 4};       // size deduced: 4
int e[5] = {0};               // all zeros — the idiom you will type most
char s[6] = "hello";          // 5 chars + '\0'
```

Four facts about the list above:

- **`int a[10];` is not zeroed.** On the stack it holds whatever the previous function left there. This is the single most common source of "works locally, WA on the judge" — your local run happened to have zeros there.
- **`= {}` and `= {0}` both zero the whole array.** `{0}` explicitly initialises the first element to 0 and value-initialises the rest, which for `int` means 0.
- **`int b[10] = {5}` sets `b[0] = 5` and the rest to 0**, not all ten to 5. To fill with 5 you need a loop or `fill`.
- **The size must be a constant expression** in standard C++. `int a[n];` with runtime `n` is a variable-length array (VLA), a GCC extension. It compiles with g++ on most judges, but it is not standard C++, MSVC rejects it, and it puts $n$ elements on the stack where large $n$ means a crash. Do not use VLAs in contest code.

```cpp
const int MAXN = 200005;
int a[MAXN];                  // global: zeroed, off the stack — the CP default
```

The CP convention is to declare the maximum size as a `const int` at the top and use it everywhere, so an off-by-one in the limit is fixed in one place.

| Form | Element values | Storage | Safe for $10^6$? |
|---|---|---|---|
| `int a[N];` inside `main` | garbage | stack | **no** — stack overflow |
| `int a[N] = {};` inside `main` | zeros | stack | **no** — stack overflow |
| `int a[N];` at global scope | zeros | BSS | yes |
| `static int a[N];` inside a function | zeros | BSS | yes |
| `vector<int> a(N);` | zeros | heap | yes |
| `new int[N]` | garbage | heap | yes, but remember to `delete[]` |

---

## 3. Indexing, and the out-of-bounds reality

Indices run from `0` to `N-1`. That is the whole contract. Everything else is undefined behaviour.

```cpp
int a[5] = {10, 20, 30, 40, 50};
cout << a[0] << ' ' << a[4] << '\n';    // 10 50
cout << a[5] << '\n';                   // UB: reads past the end
cout << a[-1] << '\n';                  // UB: reads before the start
```

Because the array is contiguous, `a[5]` reads the first four bytes after it. In a function with several locals that is usually another variable, so a read returns a plausible number and a write silently corrupts something unrelated:

```cpp
int n = 3;
int a[3] = {1, 2, 3};
a[3] = 99;            // writes over n on many builds
cout << n << '\n';    // 99 on this machine, 3 on another — genuinely undefined
```

This is why out-of-bounds writes are worse than reads: they do not crash, they corrupt, and the symptom appears far from the cause.

**Detection.** `-fsanitize=address,undefined` catches every out-of-bounds access at the cost of ~2x slowdown. Use it while debugging. `-D_GLIBCXX_DEBUG` adds checked iterators for the STL but not for raw arrays. See [Topic 02](Topic%2002%20-%20Understanding%20Comments,%20Errors,%20Warnings.md) for the full debug toolbox.

**The one-past-the-end rule.** You may *form* a pointer to `a[N]` (it is used as the end iterator everywhere: `sort(a, a+N)`), but you may not *dereference* it. `&a[N]` is legal; `a[N]` is not.

**Negative indices.** `a[-1]` is not "the last element" — it is memory before the array. C++ has no wraparound. For a cyclic index, compute it explicitly:

```cpp
int idx = ((i - k) % n + n) % n;      // n could be large; the outer +n handles negatives
```

### Association with pointers

Because an array name decays to a pointer to its first element in almost every expression:

```cpp
int a[5] = {10, 20, 30, 40, 50};
cout << *(a + 2) << '\n';      // 30
cout << 2[a] << '\n';          // 30 — legal and horrifying: a[i] == *(a+i) == i[a]
cout << sizeof(a) << '\n';     // 20 — the whole array, not a pointer
```

But inside a function the array has decayed, so `sizeof` lies:

```cpp
void f(int a[]) {              // same as int* a
    cout << sizeof(a) << '\n'; // 8 on 64-bit: the pointer, not the array
}
```

Always pass the length alongside the array. See [Topic 10](Topic%2010%20-%20Memory%20Management%20in%20C++%20and%20Pointers.md) for the full pointer treatment.

---

## 4. Where the array lives: stack, globals, heap

This is the section that saves you a runtime error on your next submission.

| Storage | Declared as | Size limit | Initial value | Lifetime |
|---|---|---|---|---|
| Stack | local variable inside a function | **~1–8 MB total** | garbage | until the function returns |
| BSS / data | global, or `static` local | up to the memory limit (256 MB typical) | zero | whole program |
| Heap | `vector`, `new`, `malloc` | up to the memory limit | garbage (`new`), zero (`vector(n)`) | until freed |

The stack limit is the trap. On Linux it defaults to 8 MB (`ulimit -s`), on Windows/MinGW often 1–2 MB. A local `int a[1000000]` is 4 MB — fine on some setups, an instant crash on others:

```cpp
int main() {
    int a[1000000];        // 4 MB on the stack — may crash before main does anything
    a[0] = 1;
    cout << a[0] << '\n';
}
```

```text
Segmentation fault (core dumped)
```

The identical code with the array moved to global scope runs:

```cpp
int a[1000000];            // 4 MB in BSS — no stack involved

int main() {
    a[0] = 1;
    cout << a[0] << '\n';
}
```

```text
1
```

The same limit applies to **recursion**: every call frame lands on the stack. A DFS that recurses $2 \times 10^5$ deep with 64 bytes of locals per frame needs 12 MB and dies. Options: increase the limit (`-Wl,--stack=268435456` on MinGW, `ulimit -s unlimited` locally), or convert to an explicit stack. See [Topic 11](Topic%2011%20-%20Functions,%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md) for recursion depth and [Topic 14](Topic%2014%20-%20STL%20Part%203%20%28Stacks%2C%20Queue%20and%20Priority%20Queue%29.md) for the iterative conversion.

**The rule, memorised:** *if it is bigger than a few thousand elements, make it global or heap-allocated.* Global is free, zero-initialised, and one word to type.

---

## 5. Memory budget arithmetic

Judges give you a memory limit (typically 256 MB, sometimes 512 MB or 1024 MB). Integers are 4 bytes, `long long` 8, `double` 8, `bool` 1, `char` 1.

$$\text{bytes} = N \times \texttt{sizeof(element)} \times (\text{number of arrays or dimensions})$$

| Declaration | Bytes | Fits 256 MB? |
|---|---|---|
| `int a[10^5]` | 0.4 MB | yes |
| `int a[10^6]` | 4 MB | yes |
| `int a[10^7]` | 40 MB | yes |
| `int a[10^8]` | 400 MB | **no** |
| `long long a[10^7]` | 80 MB | yes |
| `long long a[3*10^7]` | 240 MB | marginal |
| `int g[10^4][10^4]` | 400 MB | **no** |
| `int g[5000][5000]` | 100 MB | yes |
| `long long dp[5000][5000]` | 200 MB | marginal |
| `int g[2000][2000]` | 16 MB | yes |
| `bool vis[10^6][10^6]` | 1000 GB | absurd |
| `int t[300][300][300]` | 108 MB | yes |
| `int t[500][500][500]` | 500 MB | **no** |

Two derived rules worth carrying into every problem:

- **An `int` array can hold at most $2.5 \times 10^7$ elements inside 100 MB**, so $\approx 6 \times 10^7$ inside 256 MB. A `long long` array, half that.
- **A 2D table is quadratic in its dimension.** $5000 \times 5000$ is the practical ceiling for `int` under 256 MB, which is why so many DP problems cap $n$ at 5000.

### Compress when you cannot afford the table

If a `bool` table is too big, `vector<bool>` or `bitset` packs 8 flags per byte — a 32x saving. If a `long long` DP only ever holds values below $2^{31}$, use `int`. If a 2D table is symmetric, store a triangle. These three moves turn an MLE into an AC more often than any algorithmic change.

*A 256 MB limit is 268,435,456 bytes. Subtract ~10 MB for the runtime, and budget 250 MB. `250e6 / 8 = 3.1e7` `long long`s. Write that number down before you declare the array.*

---

## 6. Clearing arrays: `memset`, `fill`, and the lies

`memset` writes **bytes**, not values:

```cpp
int a[1000];
memset(a, 0, sizeof(a));        // correct: all bytes zero -> all ints zero
memset(a, -1, sizeof(a));       // correct: all bytes 0xFF -> every int is -1
memset(a, 1, sizeof(a));        // WRONG: every int becomes 0x01010101 = 16843009
```

Only `0` and `-1` are safe for `int` arrays, because those are the only two values whose byte pattern repeats. For anything else, the value is replicated across all four bytes.

```cpp
memset(a, 0x3f, sizeof(a));     // the "infinity" trick: every int becomes 1061109567
```

That last one is a CP idiom, not an accident: `0x3f3f3f3f` is about $1.06 \times 10^9$, and adding two of them does not overflow `int`. It is the standard "large number" for DP initialisation where you need `INF + INF` to stay positive.

`sizeof(a)` versus `sizeof(a[0]) * n`: inside a function where `a` has decayed to a pointer, `sizeof(a)` is 8 and `memset` writes 8 bytes. This is a real, silent bug.

The general answer is `std::fill`, which is typed and cannot be wrong:

```cpp
fill(a, a + n, 42);                        // C array
fill(v.begin(), v.end(), 42);              // vector
fill(&g[0][0], &g[0][0] + n * m, 0);       // 2D, flattened — legal because contiguous
```

| Goal | Correct call | Wrong call |
|---|---|---|
| Zero an `int` array | `memset(a, 0, sizeof a)` | — |
| Set all to `-1` | `memset(a, -1, sizeof a)` | `memset(a, -1, n)` |
| Set all to `1` | `fill(a, a+n, 1)` | `memset(a, 1, ...)` |
| Set all to `INF` | `fill(a, a+n, INF)` | `memset(a, INF, ...)` |
| Zero a 2D array | `memset(g, 0, sizeof g)` | `memset(g, 0, n)` |
| Reset inside a loop | `fill(...)`, or re-declare | `memset` with a `sizeof` on a decayed pointer |

*Contest story: a multi-testcase problem reinitialised its `dist` array with `memset(dist, 0, n * sizeof(int))` but `dist` was `long long`. Half the array was cleared, the other half kept the previous test's distances, and the answer was wrong only on test 2 and beyond. `fill(dist, dist+n, INF)` never has this problem.*

**Prefer re-initialisation to clearing.** Declaring the array inside the per-test loop and `fill`-ing it is slower than reusing a global, but it is correct by construction.

---

## 7. Two-dimensional arrays

```cpp
int g[1000][1000];                     // 1000 rows of 1000 ints = 4 MB, global
int h[3][4] = {{1,2,3,4}, {5,6,7,8}, {9,10,11,12}};
vector<vector<int>> vg(n, vector<int>(m, 0));   // runtime dimensions
```

`g[i][j]` is `*(*(g + i) + j)`, and because rows are contiguous, the flat offset is $i \times m + j$.

### Row-major layout and cache behaviour

C++ stores rows back to back. `g[0][0]` through `g[0][m-1]` are adjacent; `g[1][0]` follows immediately after. Traversing in row-major order touches memory sequentially, which the hardware prefetcher handles perfectly. Traversing column-major jumps `m * 4` bytes per step and defeats it.

```cpp
// Fast: sequential access
for (int i = 0; i < n; i++)
    for (int j = 0; j < m; j++)
        sum += g[i][j];

// Slow: stride of m*4 bytes per access
for (int j = 0; j < m; j++)
    for (int i = 0; i < n; i++)
        sum += g[i][j];
```

| $n = m$ | Row-major | Column-major | Ratio |
|---|---|---|---|
| 500 | 0.4 ms | 0.5 ms | 1.3x |
| 2000 | 6 ms | 19 ms | 3.1x |
| 4000 | 25 ms | 95 ms | 3.8x |

The exact numbers depend on the machine and cache size, but the shape does not: at 2000×2000 the array no longer fits in L2 and the stride cost becomes real. When a DP TLEs with the right complexity, check the loop order first.

The one legitimate column-major traversal is a transpose or a column-wise DP. When you must do it, consider storing the transpose instead, or flattening and blocking.

### Clearing and passing

```cpp
memset(g, 0, sizeof g);          // only correct at the scope where g is declared as an array
fill(&g[0][0], &g[0][0] + n * m, 0);   // works for a 2D array of known extent
```

Passing a 2D array to a function requires the inner dimension:

```cpp
void print(int g[][1000], int n) {      // second dimension must be a constant
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            cout << g[i][j] << " \n"[j == n-1];
}
```

For anything with runtime dimensions, use `vector<vector<int>>` or a flat `vector<int>` of size $n \times m$. The flat version is faster (one allocation, one indirection) and is what you should reach for in DP:

```cpp
vector<long long> dp(n * m);
auto at = [&](int i, int j) -> long long& { return dp[i * m + j]; };
at(2, 3) = 7;                    // no nested indirection
```

---

## 8. Flattening: one array, many dimensions

A flat array indexed as `i * m + j` is exactly equivalent to `g[i][j]`, but cheaper:

- One allocation instead of $n$ allocations (a `vector<vector<>>` allocates $n+1$ blocks).
- One pointer dereference per access instead of two. The inner `vector`'s buffer is elsewhere in memory, so you pay an extra load.
- `memset` and `fill` work on the whole thing.
- Cache prefetching sees a single stream.

```cpp
int n = 1000, m = 1000;
vector<int> g(n * m, 0);

// access
g[i * m + j] = 1;

// full clear
fill(g.begin(), g.end(), 0);

// the whole grid as one contiguous buffer — a single memcpy works
```

The cost is that you must carry `m` around and never forget it. Wrap it in a lambda or a tiny struct (see [Topic 11](Topic%2011%20-%20Functions%2C%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md)) if the arithmetic gets repetitive.

---

## 9. Three-dimensional arrays

Rarely needed, occasionally unavoidable: DP over `(index, count, state)`, or a 3D grid in a simulation.

```cpp
int dp[105][105][2];             // index, remaining, last-picked
for (int i = 0; i < n; i++)
    for (int k = 0; k <= K; k++)
        for (int s = 0; s < 2; s++)
            dp[i][k][s] = -1;    // "unvisited" sentinel
```

Memory is multiplicative:

$$\text{bytes} = d_1 \times d_2 \times d_3 \times \texttt{sizeof(T)}$$

| Dimensions | Element | Bytes | Fits 256 MB? |
|---|---|---|---|
| `100 x 100 x 100` | `int` | 4 MB | yes |
| `300 x 300 x 300` | `int` | 108 MB | yes |
| `500 x 500 x 500` | `int` | 500 MB | **no** |
| `100 x 100 x 100` | `long long` | 8 MB | yes |
| `1000 x 1000 x 100` | `int` | 400 MB | **no** |

The practical ceiling is $10^8$ `int` cells = 400 MB, so **$5 \times 10^7$ cells is the safe maximum** under 256 MB. If a problem needs more, the intended solution rolls one dimension (keep layer `i-1`, discard `i-2`) — a technique you will meet in DP chapters.

Traversal order matters even more in 3D: iterate the **last** index innermost so the innermost loop walks contiguous memory.

---

## 10. Prefix sums

The single most useful array technique. Precompute cumulative sums so any range sum is $\mathcal{O}(1)$.

### 1D

$$P[0] = 0, \qquad P[i] = P[i-1] + a[i-1], \qquad \sum_{l}^{r} a = P[r+1] - P[l]$$

```cpp
vector<long long> P(n + 1, 0);
for (int i = 0; i < n; i++) P[i + 1] = P[i] + a[i];

// sum of a[l..r], inclusive, 0-indexed
long long rangeSum(int l, int r) { return P[r + 1] - P[l]; }
```

Two decisions that prevent bugs:

- **Size $n+1$ with `P[0] = 0`.** Then the formula is uniform and `l = 0` needs no special case. This one convention removes the entire class of "empty prefix" bugs.
- **Use `long long` for `P`** even when `a` is `int`. Sums of $10^5$ values near $10^9$ reach $10^{14}$ and overflow `int` silently.

Build cost $\mathcal{O}(n)$, query $\mathcal{O}(1)$. This turns "answer $q$ range-sum queries" from $\mathcal{O}(nq)$ into $\mathcal{O}(n + q)$.

### 2D

$$P[i][j] = g[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1]$$

The sum over the rectangle with corners $(r_1,c_1)$ to $(r_2,c_2)$ inclusive:

$$S = P[r_2+1][c_2+1] - P[r_1][c_2+1] - P[r_2+1][c_1] + P[r_1][c_1]$$

```cpp
int n, m;
long long g[1005][1005], P[1005][1006];

// build (1-indexed working copy)
for (int i = 1; i <= n; i++)
    for (int j = 1; j <= m; j++)
        P[i][j] = g[i][j] + P[i-1][j] + P[i][j-1] - P[i-1][j-1];

// rectangle sum, inclusive, 1-indexed
long long rect(int r1, int c1, int r2, int c2) {
    return P[r2][c2] - P[r1-1][c2] - P[r2][c1-1] + P[r1-1][c1-1];
}
```

The inclusion–exclusion is worth deriving once on paper. The prefix at $(i,j)$ covers everything up and left; subtracting the two strips removes the overlap twice, so you add the corner back once. Sign errors here are the most common 2D bug — test with a $1 \times 1$ rectangle (the answer must be `g[r1][c1]`) and with the full grid (the answer must be the total).

Prefix sums appear again in Chapter 3; here you only need the mechanics.

---

## 11. Difference arrays: range updates in $\mathcal{O}(1)$

The inverse of prefix sums. To add $v$ to every element of `a[l..r]` in $\mathcal{O}(1)$ instead of $\mathcal{O}(r-l+1)$:

```cpp
vector<long long> D(n + 2, 0);

void addRange(int l, int r, long long v) {   // inclusive, 0-indexed
    D[l] += v;
    D[r + 1] -= v;
}

// after all updates, prefix-sum D to recover the values
long long cur = 0;
for (int i = 0; i < n; i++) {
    cur += D[i];
    a[i] += cur;
}
```

With $q$ updates this is $\mathcal{O}(n + q)$ instead of $\mathcal{O}(nq)$.

**The `n + 1` index is not optional.** An update to `[l, n-1]` writes `D[n]`, so `D` needs $n+1$ slots or you corrupt the next variable. Size it `n + 2` and stop worrying.

The 2D version applies the same idea with four corner updates per rectangle. You will want it for "paint $q$ rectangles on a grid, then count painted cells".

---

## 12. Arrays as frequency tables

A fixed array indexed by value is the fastest hash map that exists — when the value range is small.

```cpp
int freq[26] = {};                       // letters
for (char c : s) freq[c - 'a']++;

int cnt[100001] = {};                    // values 1..10^5
for (int x : a) cnt[x]++;

// count elements <= k in O(1) after O(V) build
for (int i = 1; i <= V; i++) cnt[i] += cnt[i-1];
```

| Range of values | Structure | Cost per op |
|---|---|---|
| $V \le 10^6$ | `int freq[V]` | $\mathcal{O}(1)$, 4 bytes per slot |
| $V \le 10^8$ | `int freq[V]` | $\mathcal{O}(1)$, 400 MB — usually too much |
| $V$ up to $10^{18}$ | `map` / `unordered_map` | $\mathcal{O}(\log n)$ / $\mathcal{O}(1)$ average |
| Sparse, but order matters | sorted `vector` + `lower_bound` | $\mathcal{O}(\log n)$ |

The threshold is memory, not speed: $10^6$ `int`s is 4 MB and always worth it; $10^8$ is 400 MB and almost never.

Remember that `int freq[V]` must be global or inside a `vector` if $V$ is large, and that a per-testcase `memset` over the full $V$ can dominate the runtime when $T$ is large. Either size the clear to the actual maximum value used, or record which indices you touched and reset only those.

---

## 13. `std::array`: the modern fixed-size array

`<array>` gives you a fixed-size array that does not decay to a pointer, knows its own size, and works with STL algorithms.

```cpp
#include <array>
array<int, 5> a = {1, 2, 3, 4, 5};

cout << a.size() << '\n';       // 5 — always correct, even inside a function
sort(a.begin(), a.end());
for (int x : a) cout << x << ' ';
```

| Feature | `int a[5]` | `array<int,5>` |
|---|---|---|
| `sizeof` inside a function | wrong (pointer) | correct |
| `.size()` | no | yes |
| `.begin()/.end()` | via free functions | members |
| Pass by value (copy) | no | yes |
| Bounds check | no | `a.at(i)` throws |
| Zero-init with `{}` | yes | yes |
| Works with range-for | yes | yes |

In contests, plain C arrays remain the most common choice because they are shorter to type and every template in circulation uses them. `std::array` is the safer default in code that will be read later. Both are equally fast.

---

## 14. The standard array recipes

### 14.1 Max, min, and their indices

```cpp
int mx = a[0], imax = 0;
for (int i = 1; i < n; i++)
    if (a[i] > mx) { mx = a[i]; imax = i; }
```

Use `>` for the first occurrence, `>=` for the last. That single character is the answer to "print the leftmost maximum" versus "rightmost maximum", and setters do test it.

### 14.2 Reverse in place

```cpp
for (int i = 0, j = n - 1; i < j; i++, j--) swap(a[i], a[j]);
```

`i < j` (not `i <= j`) — with `<=` the middle element swaps with itself, which is harmless for an int and wrong for anything with a side effect.

### 14.3 Rotate left by $k$

```cpp
k %= n;                                  // k can exceed n
for (int i = 0; i < k; i++) {            // O(n*k) — only for tiny k
    int first = a[0];
    for (int j = 0; j + 1 < n; j++) a[j] = a[j+1];
    a[n-1] = first;
}
```

The $\mathcal{O}(n)$ version uses the three-reversal trick: reverse the first $k$, reverse the rest, reverse everything. `std::rotate(a, a+k, a+n)` does exactly this — see [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md).

### 14.4 Remove duplicates in place (sorted array)

```cpp
int j = 0;
for (int i = 0; i < n; i++)
    if (i == 0 || a[i] != a[i-1]) a[j++] = a[i];
n = j;                                   // new logical length
```

The array is sorted, so duplicates are adjacent. `j` is the write pointer, always at or behind `i`. This is the two-pointer compaction pattern; `std::unique` does the same thing.

### 14.5 Kadane: maximum subarray sum

```cpp
long long best = a[0], cur = a[0];
for (int i = 1; i < n; i++) {
    cur = max((long long)a[i], cur + a[i]);   // extend or restart
    best = max(best, cur);
}
```

$\mathcal{O}(n)$, handles all-negative arrays if you initialise to `a[0]` rather than `0`. Initialising `best = 0` returns 0 for an all-negative array — wrong when the statement requires a non-empty subarray.

### 14.6 Boyer–Moore majority vote

```cpp
int cand = a[0], cnt = 0;
for (int i = 0; i < n; i++) {
    if (cnt == 0) { cand = a[i]; cnt = 1; }
    else if (a[i] == cand) cnt++;
    else cnt--;
}
// verify: a majority element exists only if count(cand) > n/2
```

$\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space. **The verification pass is mandatory** — the algorithm always returns a candidate, even when no majority exists.

### 14.7 Equilibrium index

The first index where the sum of the left part equals the sum of the right part:

```cpp
long long total = accumulate(a, a+n, 0LL), left = 0;
for (int i = 0; i < n; i++) {
    if (left == total - left - a[i]) { cout << i << '\n'; break; }
    left += a[i];
}
```

$\mathcal{O}(n)$ with $\mathcal{O}(1)$ extra space, instead of the $\mathcal{O}(n^2)$ double loop.

### 14.8 Check sorted

```cpp
bool sorted = true;
for (int i = 1; i < n; i++) if (a[i] < a[i-1]) { sorted = false; break; }
```

Start at `i = 1`, never `i = 0`. `is_sorted(a, a+n)` is the library version.

### 14.9 Count occurrences of a value

```cpp
int c = 0;
for (int i = 0; i < n; i++) c += (a[i] == x);    // branchless
```

On a sorted array, `upper_bound - lower_bound` is $\mathcal{O}(\log n)$.

### 14.10 Spiral / ring traversal of a grid

```cpp
int top = 0, bottom = n-1, left = 0, right = m-1;
while (top <= bottom && left <= right) {
    for (int j = left; j <= right; j++)  cout << g[top][j] << ' ';
    top++;
    for (int i = top; i <= bottom; i++)  cout << g[i][right] << ' ';
    right--;
    if (top <= bottom) { for (int j = right; j >= left; j--) cout << g[bottom][j] << ' ';
                         bottom--; }
    if (left <= right) { for (int i = bottom; i >= top; i--) cout << g[i][left] << ' ';
                         left++; }
}
```

The two `if` guards prevent printing a row or column twice when the remaining rectangle is degenerate. Remove them and a $3 \times 3$ grid works while a $3 \times 4$ grid repeats its middle row.

---

## Key Takeaways

- An array is contiguous, so `a[i]` is $\mathcal{O}(1)$ and `sizeof(a)` is exact — until it decays to a pointer inside a function.
- `int a[N];` on the stack is **uninitialised and may crash** for large $N$; make large arrays global (zeroed, off the stack) or heap-allocated.
- The stack is only ~1–8 MB. That limit also caps recursion depth.
- Budget memory before declaring: `bytes = N × sizeof(T) × dimensions`. Under 256 MB you get ~$6 \times 10^7$ `int`s or ~$3 \times 10^7$ `long long`s.
- `memset` writes bytes: only `0` and `-1` are safe for `int` arrays. `0x3f` gives the standard $\approx 1.06 \times 10^9$ infinity. Otherwise use `fill`.
- C++ 2D arrays are row-major. Iterate rows outermost for sequential access; the wrong order costs 3–4x on a 2000×2000 grid.
- Flattening a 2D array to `i*m + j` removes an indirection and makes whole-array `fill`/`memcpy` legal.
- Prefix sums: $\mathcal{O}(n)$ build, $\mathcal{O}(1)$ range query. Always size `n+1` with `P[0] = 0`, and always use `long long`.
- 2D prefix sums are inclusion–exclusion with four terms; test with a $1 \times 1$ rectangle and with the full grid.
- Difference arrays give $\mathcal{O}(1)$ range updates; size them `n+1` so an update ending at `n-1` is legal.
- A frequency array is the fastest possible map when the value range is $\le 10^6$ — the limit is memory, not speed.
- Kadane needs `best = a[0]` (not `0`) to survive all-negative input. Boyer–Moore needs a verification pass.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `int a[1000000];` inside `main` | 4 MB on a ~1–8 MB stack → crash | Declare globally or use `vector` |
| 2 | `int a[10];` expecting zeros | Stack memory is garbage | `int a[10] = {};` or global |
| 3 | `int a[10] = {5};` expecting all 5s | Only index 0 is 5 | Loop or `fill(a, a+10, 5)` |
| 4 | `memset(a, 1, ...)` on `int` | Each int becomes `0x01010101` | `fill` |
| 5 | `memset(a, 0, sizeof a)` on a decayed pointer | Clears 8 bytes | Pass the length; use `fill` |
| 6 | `sizeof(a)` inside a function | Returns pointer size, not array size | Pass `n`, or use `std::array` |
| 7 | `for (i = 0; i <= n; i++)` | Reads `a[n]`, one past the end | `< n` |
| 8 | Trusting `a[n]` to crash | It usually reads a plausible garbage value | Enable `-fsanitize=address` |
| 9 | `int a[n];` with runtime `n` | VLA: non-standard, stack-limited | `vector<int> a(n)` |
| 10 | `for (int i = n-1; i >= 0; i--)` with `unsigned i` | `i >= 0` never false → infinite loop | Use `int`, or `i --> 0` carefully |
| 11 | Column-major traversal of a 2D array | 3–4x slower past L2 | Rows outermost |
| 12 | `memset` a `long long` array with `n * sizeof(int)` | Half the bytes stay dirty | `sizeof(long long)`, or `fill` |
| 13 | 2D prefix sums with the wrong signs | Off-by-one rectangles | Test $1 \times 1$ and the full grid |
| 14 | `long long P[]` built from `int` | Sum overflows before assignment | Cast each term, or build from `long long` |
| 15 | Difference array sized `n` | Update ending at `n-1` writes `D[n]` | Size `n+1` (or `n+2`) |
| 16 | `best = 0` in Kadane | Wrong on all-negative input | `best = a[0]` |
| 17 | Boyer–Moore without verification | Reports a candidate that is not a majority | Second pass, count > n/2 |
| 18 | Spiral traversal without degenerate-rectangle guards | Middle row/column printed twice | `if (top <= bottom)`, `if (left <= right)` |
| 19 | Clearing a `10^6` frequency array per test case | $T \times 10^6$ dominates the runtime | Reset only touched indices |
| 20 | `int g[10^4][10^4]` | 400 MB → MLE | `5000×5000` max for `int`, or compress |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Reverse an array in place, print with indices | AtCoder ABC A / Codeforces Div. 4 A | Easy |
| 2 | Count occurrences and mode of an array | Codeforces Div. 3 A | Easy |
| 3 | Rotate an array left/right by $k$ | Codeforces Div. 3 B | Easy |
| 4 | Check if an array is sorted, find first violation | AtCoder ABC B | Easy |
| 5 | Remove duplicates from a sorted array in place | Codeforces Div. 3 B / CSES | Easy |
| 6 | Range sum queries ($q$ queries, no updates) | CSES "Static Range Sum Queries" | Easy |
| 7 | 2D prefix sums, $q$ rectangle-sum queries | CSES "Forest Queries" | Easy–Medium |
| 8 | Difference array: $q$ range increments, then report the array | CSES "Range Update Queries" style | Easy–Medium |
| 9 | Maximum subarray sum (Kadane) | CSES "Maximum Subarray Sum" | Medium |
| 10 | Majority element with verification | Codeforces Div. 2 B | Medium |
| 11 | Equilibrium / pivot index | Codeforces Div. 2 A/B | Easy–Medium |
| 12 | Spiral order traversal of an $n \times m$ grid | Codeforces Div. 3 C | Medium |
| 13 | 2D difference array: paint rectangles, count painted cells | Codeforces Div. 2 C | Medium |
| 14 | Row-major vs column-major timing experiment on a 3000×3000 grid | Self-imposed | Medium |
| 15 | Memory-budget drill: which of these declarations MLE at 256 MB? | Self-imposed | Easy but instructive |

---

## Next Up

→ [Topic 10 — Memory Management in C++ and Pointers](Topic%2010%20-%20Memory%20Management%20in%20C++%20and%20Pointers.md)

You now know where an array lives. Topic 10 makes that explicit: the stack/heap/global model with real numbers, why recursion overflows, what `new` and `delete` actually cost, and pointers and references — including the six causes of a segmentation fault, which is the error you will spend the most time debugging for the rest of your career.