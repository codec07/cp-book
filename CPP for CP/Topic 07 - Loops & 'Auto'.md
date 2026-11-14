# Topic 07 — Loops & `auto`

A contest solution is a loop wrapped in an idea. You will write between fifty and five hundred loops per contest, and almost every TLE, every wrong answer on test 3, and every "why does this never terminate" panic traces back to one of about a dozen loop mistakes. This chapter covers the three loop forms, the arithmetic of iteration counts, the off-by-one family of bugs, and the two modern tools — range-based `for` and `auto` — that make loops shorter but occasionally hide a copy or a proxy object.

> **Prerequisites:** [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems,%20Variables%20and%20Datatypes.md), [Topic 06 — Conditional Statements](Topic%2006%20-%20Conditional%20Statements%20%28If,%20else,%20else%20if,%20ternary%20operators%29.md)
> **You will be able to:**
> - Choose `for` / `while` / `do-while` deliberately instead of by habit.
> - Estimate a loop's cost from $n$ before you type it, and know instantly whether it TLEs.
> - Kill off-by-one bugs by construction, not by adding `+1` until samples pass.
> - Use range-based `for` and `auto` without paying hidden $O(n)$ copy costs or falling into the `vector<bool>` proxy trap.
> - Recognise and fix infinite loops caused by `continue`, unsigned counters, and stale iterators.

---

## 1. The three loop forms and when each is natural

All three forms compile to the same thing: a test, a body, a jump back. The difference is *where the bookkeeping lives*, and that is what makes one of them the natural fit for a given problem.

### 1.1 `for` — you know the bounds before you start

```cpp
for (init; condition; step) {
    body;
}
```

Execution order: `init` once → test `condition` → body → `step` → test → body → `step` → … The `step` runs *after* the body and *before* the next test. That single sentence explains 90% of `for` confusion.

Use `for` when the number of iterations is knowable up front: array sweeps, grid scans, "repeat $k$ times", digit positions, DP states.

### 1.2 `while` — the exit depends on the data

Use `while` when you cannot say in advance how many iterations you need: reading until EOF, shrinking a window until a condition holds, halving $n$ until it hits zero, walking a pointer until it meets another.

```cpp
while (condition) {
    body;
}
```

If `condition` is false on entry, the body never runs. That is usually what you want.

### 1.3 `do-while` — the body must run at least once

```cpp
do {
    body;
} while (condition);   // note the semicolon
```

The test happens *after* the body, so the body executes at least once **even if the condition is false from the start**. This is the gotcha, and it is a real one:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 0;
    while (n > 0) cout << "while: body ran\n";     // never runs
    do { cout << "do-while: body ran\n"; n--; } while (n > 0);   // runs once
    return 0;
}
```

```text
do-while: body ran
```

`n` is zero, the `while` body is skipped, and the `do-while` body still runs once — then `n` becomes `-1` and the loop stops. In CP, `do-while` has exactly two honest uses: interactive problems where you must print a query before reading a response, and simulation steps that must be applied once before testing for convergence. For everything else, prefer `while` — it fails safe.

| Form | Test position | Minimum iterations | Natural fit | Typical CP use |
|---|---|---|---|---|
| `for` | before body | 0 | bounded count known | array/grid sweeps, `k` repetitions |
| `while` | before body | 0 | bound unknown, data-driven | two pointers, binary search, EOF read |
| `do-while` | after body | **1** | must act before testing | interactive queries, fixpoint iteration |

> *Contest story: a `do-while` in a multi-test-case input loop is a classic wrong-answer source. If the problem has `T` test cases and you write `do { solve(); } while (--T);`, you solve it once even when `T == 0` — which some judges do test.*

---

## 2. The canonical loop, and why it is `i < n`

```cpp
for (int i = 0; i < n; i++) {
    // work on a[i]
}
```

Four decisions are packed into that line, and each one matters:

1. **`int`, not `unsigned`.** Signed counters make reverse loops and `i - 1` safe. See §6.
2. **Start at 0.** Arrays are 0-indexed; `a[0]` through `a[n-1]` are the valid cells. There are exactly $n$ of them.
3. **`i < n`, not `i <= n`.** With `i <= n` you execute $n+1$ times and touch `a[n]`, which does not exist.
4. **`i++` vs `++i`.** For `int` they are identical after optimisation; for iterators `++i` avoids a temporary. Use `++i` in loops out of habit — it is never worse.

The rule to internalise: **the number of integers in `[0, n)` is $n$.** Every off-by-one bug is a failure to count that interval correctly.

---

## 3. Off-by-one: the number one CP bug

An off-by-one is not a typo you fix by guessing. It is a counting error, and it has four recurring shapes. Learn the shapes and you will spot them in your own code in seconds.

### 3.1 Shape 1 — inclusive vs exclusive bounds on a sum

```cpp
long long sum = 0;
for (int i = 1; i <= n; i++) sum += i;      // sum of 1..n: inclusive, correct

long long sum2 = 0;
for (int i = 1; i <= n; i++) sum2 += a[i];  // WRONG: skips a[0], reads a[n]
```

The first loop wants *inclusive* bounds because the mathematical range is $1..n$; the second wants *exclusive* because the array range is $0..n-1$. Mixing the two conventions in one file is how bugs are born: pick one convention per problem and write it as a comment at the top.

### 3.2 Shape 2 — reading $n$ elements into an array of size $n$

```cpp
int a[100005];
int n; cin >> n;
for (int i = 0; i < n; i++) cin >> a[i];   // correct
for (int i = 0; i <= n; i++) cin >> a[i];  // writes a[n]: UB, and eats the next token
```

The second version reads one token too many, so on multi-test-case input it *steals* the next test's `n`. Symptom: test 1 correct, test 2 answers garbage. The fix is never "add a `+1` to the array size" — that just hides the logic error.

### 3.3 Shape 3 — binary search boundaries

```cpp
int lo = 0, hi = n - 1;          // search space [lo, hi], inclusive
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (a[mid] == target) { /* found */ break; }
    if (a[mid] < target) lo = mid + 1;
    else                 hi = mid - 1;
}
```

With `hi = n - 1` (inclusive), the loop condition must be `lo <= hi`. With `hi = n` (exclusive), it must be `lo < hi`. The pair `hi = n - 1` and `lo < hi` silently misses the last element; the pair `hi = n` and `lo <= hi` reads `a[n]`. These two errors account for a large share of failed binary searches — which is why [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md) teaches `lower_bound` instead of hand-rolled search.

### 3.4 Shape 4 — `n - 1` steps, not `n`, in graph and DP loops

A tree with $n$ vertices has exactly $n-1$ edges, so you read $n-1$ pairs:

```cpp
for (int i = 0; i < n - 1; i++) {
    int u, v; cin >> u >> v;
    adj[u].push_back(v);
    adj[v].push_back(u);
}
```

Reading $n$ edges here consumes the next test case's first vertex, and every subsequent answer is shifted. In DP, the same shape appears as "you fill `dp[1..n]` from `dp[0]`, so `dp` needs `n+1` cells": `vector<long long> dp(n + 1);` — not `dp(n)`.

> **Defensive habit:** before writing the loop, say the bounds out loud in a comment — `// i in [0, n)`, `// sum a[l..r] inclusive`. Two seconds of comment beats twenty minutes of debugging.

---

## 4. Nested loops and the multiplication rule

Iterations *multiply*. A loop of $n$ containing a loop of $m$ costs $n \cdot m$ body executions, regardless of how the loops look.

```cpp
// O(n^2): n^2 body executions
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        work(i, j);
```

Triangular loops are half: `for (j = i + 1; j < n; j++)` inside `for (i = 0; i < n; i++)` gives $\binom{n}{2} \approx n^2/2$ — still $\mathcal{O}(n^2)$, half the constant. The dangerous case is the *hidden* double loop, where the body calls something linear:

```cpp
for (int i = 0; i < n; i++)
    for (char c : t) s += c;     // O(|t|) per iteration → O(n·|t|), plus reallocation
```

### 4.1 The budget table you must memorise

The working assumption for a 1-second limit on a modern judge is **$10^8$ simple operations per second** (see §5). Anything above roughly $2 \times 10^8$ is a gamble.

| $n$ | $O(n)$ | $O(n \log n)$ | $O(n^2)$ | $O(n^3)$ | Verdict at $n$ |
|---|---|---|---|---|---|
| $10^3$ | $10^3$ | $10^4$ | $10^6$ | $10^9$ | $O(n^2)$ fine, $O(n^3)$ TLE |
| $10^4$ | $10^4$ | $1.3 \times 10^5$ | $10^8$ | $10^{12}$ | $O(n^2)$ borderline, $O(n^3)$ TLE |
| $10^5$ | $10^5$ | $1.7 \times 10^6$ | $10^{10}$ | $10^{15}$ | $O(n \log n)$ fine, $O(n^2)$ TLE |
| $10^6$ | $10^6$ | $2 \times 10^7$ | $10^{12}$ | $10^{18}$ | $O(n)$/ $O(n \log n)$ fine, $O(n^2)$ hopeless |
| $10^7$ | $10^7$ | $2.3 \times 10^8$ | $10^{14}$ | — | $O(n)$ only, and only if the body is cheap |

Read the table backwards when you read a constraint: **$n \le 10^5$ and a 1 s limit is the setter telling you "$\mathcal{O}(n \log n)$ or better"**. $n \le 10^3$ is the setter telling you "$O(n^2)$ is expected". $n \le 20$ is the setter telling you "bitmask / exponential is fine".

> *Contest story: a double loop over $n = 10^5$ runs $10^{10}$ iterations. At $10^8$ per second that is 100 seconds. No `#pragma`, no `-O2`, and no fast input will save it. The only fix is a better algorithm — and the time to notice is at the moment you read the constraints, not after the TLE.*

### 4.2 Hidden iteration counts

The table above covers visible loops. Watch for these instead: `sort` is $n \log n$ *inside* whatever loop encloses it (sorting inside a loop is often $n^2 \log n$); `string` concatenation in a loop is quadratic because each `+=` copies; `map` lookup inside a loop adds a $\log n$ factor and a pointer chase; and `v.erase(v.begin())` inside a loop is $n^2$ because every erase shifts the tail. If a loop body looks like one statement but calls a linear function, multiply.

---

## 5. The $10^8$ rule and how to budget before you code

Modern CPUs execute a few billion instructions per second, but a loop body is not one instruction: an array read, an add, a compare, a branch, plus loop overhead is 5–10 instructions. Empirically, **a tight loop over a contiguous `int` array does about $10^8$–$3 \times 10^8$ iterations per second** under `-O2`. A loop that touches `map`, `set`, or `string` does $10^7$ or fewer.

The budget procedure: read the maximum $n$ from the constraints → guess the intended complexity from the table → compute $C \cdot n^{a} \log^{b} n$ → compare with $10^8$ (or $2 \times 10^8$ with fast I/O and a trivial body). If you are over by $10\times$, change the algorithm *before* writing it. You can also measure instead of guessing — see §15.1.

Memory is the twin constraint; a rule of thumb is `int` arrays up to $2.5 \times 10^7$ elements inside a 256 MB limit. [Topic 09](Topic%2009%20-%20Arrays%20%281D,%202D%20and%203D%29.md) does the exact arithmetic.

---

## 6. Reverse iteration and the unsigned trap

The correct reverse sweep:

```cpp
for (int i = n - 1; i >= 0; i--) {
    // process a[i]
}
```

It terminates because `i` is signed: when `i` becomes `-1`, `-1 >= 0` is false.

Now make the counter unsigned, which is what happens automatically if you write `for (auto i = v.size() - 1; ...)` — `size()` returns `size_t`, an unsigned type:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {10, 20, 30};
    for (size_t i = v.size() - 1; i >= 0; --i) {   // HANGS
        cout << v[i] << ' ';
    }
    return 0;
}
```

`i >= 0` is *always true* for an unsigned type, so the loop never ends; after `i == 0` the decrement wraps to `SIZE_MAX` and you index `v[18446744073709551615]` — a segfault at best, an infinite loop at worst. Two safe forms:

```cpp
for (size_t i = v.size(); i-- > 0; ) cout << v[i] << ' ';   // idiom: post-decrement in the test

for (int i = (int)v.size() - 1; i >= 0; --i) cout << v[i] << ' ';  // signed copy
```

The first is the classic "goes down to zero" idiom: `i-- > 0` compares the old value, then decrements. When `i` is 0 the test fails, so the body sees `i == 0` exactly once and the variable never underflows. The second is easier to read and safe as long as the container is smaller than `INT_MAX`.

### 6.1 Signed/unsigned comparison warnings

```cpp
vector<int> v(10);
for (int i = 0; i < v.size(); i++) ...   // warning: comparison of integer expressions
                                        // of different signedness [-Wsign-compare]
```

It is only a warning, and in practice harmless for containers under $2^{31}$ elements. But `-Werror` judges (and clean builds) turn it into an error. Silence it honestly by casting once:

```cpp
for (int i = 0, m = (int)v.size(); i < m; i++) ...
```

Casting `v.size()` once into `m` also avoids recomputing it, and reads better than sprinkling casts.

---

## 7. `break`, `continue`, and the increment you skipped

`break` leaves the innermost enclosing loop immediately. `continue` skips the rest of the body and jumps to the *next iteration* — which for a `for` loop means the step expression still runs, but for a `while` loop means your hand-written increment is skipped.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    // for: continue still runs i++
    for (int i = 0; i < 5; i++) {
        if (i == 2) continue;
        cout << i << ' ';
    }
    cout << '\n';

    // while: continue skips the i++  → INFINITE LOOP
    int i = 0;
    while (i < 5) {
        if (i == 2) continue;   // never reaches i++
        cout << i << ' ';
        i++;
    }
    return 0;
}
```

```text
0 1 3 4
```

The program then hangs. If you must `continue` inside a `while`, put the increment at the top of the body or convert to a `for`:

```cpp
int i = 0;
while (i < 5) {
    int cur = i++;
    if (cur == 2) continue;
    cout << cur << ' ';
}
```

`break` inside nested loops only exits one level. Breaking out of two levels needs a flag, a `goto`, or a lambda — see §14.

| Statement | Effect | Runs the `for` step? | Runs the `while` body's code after it? |
|---|---|---|---|
| `break` | exit innermost loop | no (loop is gone) | no |
| `continue` | next iteration | **yes** | **no** |
| `return` | exit the function | no | no |

---

## 8. Loop variable scope

A variable declared in the `for` header lives only in the loop:

```cpp
for (int i = 0; i < n; i++) { /* i visible here */ }
// cout << i;   // compile error: i was not declared
```

That is a feature — it prevents the "I reused `i` and it still holds `n`" bug. When you need the index after the loop, keep it in an outer variable:

```cpp
int pos = -1;
for (int i = 0; i < n; i++) if (a[i] == target) { pos = i; break; }
```

Declaring `int i;` before the loop (old C style) leaks `i` into the enclosing scope and lets two unrelated loops silently share a counter. Don't. And compile with `-Wshadow` locally: it catches accidental shadowing, where an inner `int x` silently hides an outer one.

---

## 9. `for` with multiple variables: the two-pointer loop

One `for` can drive two counters. The clean case is when both pointers move in lockstep — reversing in place, or checking a palindrome:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> a = {1, 2, 3, 4, 5, 6, 7};

    // reverse in place: i walks up, j walks down, one swap per pass
    for (int i = 0, j = (int)a.size() - 1; i < j; i++, j--) {
        swap(a[i], a[j]);
    }
    for (int x : a) cout << x << ' ';
    cout << '\n';

    string s = "racecar";
    bool pal = true;
    for (int i = 0, j = (int)s.size() - 1; i < j; i++, j--) {
        if (s[i] != s[j]) { pal = false; break; }
    }
    cout << (pal ? "palindrome" : "not palindrome") << '\n';
    return 0;
}
```

```text
7 6 5 4 3 2 1
palindrome
```

The `i < j` condition is what makes this safe for both even and odd lengths: for odd length the middle element is simply never compared with itself.

When the two pointers move by *different* amounts depending on the data — the general two-pointer sum search, or a sliding window — a lockstep `for` header is the wrong tool. Use `while (i < j)` and move one pointer per branch:

```cpp
// sorted array, find a pair summing to target
int i = 0, j = n - 1;
bool found = false;
while (i < j) {
    int sum = a[i] + a[j];
    if (sum == target) { found = true; break; }
    if (sum < target) i++;      // need a bigger sum
    else              j--;      // need a smaller sum
}
```

Each iteration moves exactly one pointer inward, so the loop runs at most $n$ times — $\mathcal{O}(n)$ with $\mathcal{O}(1)$ memory.

---

## 10. Range-based `for`: the three forms and their cost

`for (decl : range)` iterates over anything with `begin()`/`end()`. The single most important decision is the *declaration type*, because it determines whether you copy.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<string> words = {"alpha", "beta", "gamma"};

    // (1) by value: copies every element  → O(n) string allocations
    for (auto w : words) w += "!";

    // (2) by const reference: no copy, read-only
    for (const auto &w : words) cout << w << ' ';
    cout << '\n';

    // (3) by non-const reference: no copy, can mutate
    for (auto &w : words) w += "!";

    for (const auto &w : words) cout << w << ' ';
    cout << '\n';

    return 0;
}
```

```text
alpha beta gamma
alpha! beta! gamma!
```

Form (1) is the trap. Each iteration constructs and destroys a `std::string`: for $n = 10^6$ strings of length 20 that is $10^6$ heap allocations, easily 100 ms+ of pure waste, and it *silently does nothing useful* because the modifications die with the copy. With `int` the copy is free (a register), which is why the bug survives testing and only shows up when someone puts a `string` or a big `struct` in the vector.

| Declaration | Copies? | Can modify? | Use when |
|---|---|---|---|
| `for (auto x : v)` | **yes**, every element | no effect on `v` | element is a scalar (`int`, `char`, `double`) |
| `for (const auto &x : v)` | no | no | default for reading anything non-trivial |
| `for (auto &x : v)` | no | yes | you need to write through `x` |
| `for (auto &&x : v)` | no | yes (binds to rvalue too) | generic code, or `v` yields proxies |
| `for (const auto &x : expr())` | no | no | iterating a temporary — lifetime is extended for the loop |

**Rule of thumb:** `const auto&` by default; drop the `const` only when you write; drop the `&` only when the type is a scalar *and* you want a copy.

Two more details worth knowing:

- Iterating a temporary is safe: `for (auto x : makeVector())` — the range expression's lifetime extends to the end of the loop, but only the *top-level* temporary.
- Range-for gives you no index; use a counter or `std::distance`. Never change the container's size inside a range-for: `push_back` may reallocate and invalidate the hidden iterators — UB, not an exception.

### 10.1 C++20: init-statement in range-for

C++20 allows an init-statement before the range, which keeps the container alive and named once:

```cpp
// C++20
for (auto data = readInput(); const auto &x : data) process(x);
```

Judges on GCC 10+ accept it; older ones do not. The C++17 equivalent is two lines: declare `data`, then loop.

---

## 11. `auto`: what it deduces, and what it refuses to deduce

`auto` asks the compiler to deduce a type from the initialiser using the same rules as template argument deduction. Those rules are the source of every `auto` surprise.

```cpp
int x = 5;
const int cx = x;
int &rx = x;

auto a = x;    // int
auto b = cx;   // int      ← const is DROPPED
auto c = rx;   // int      ← reference is DROPPED (copy!)
auto &d = cx;  // const int&  ← reference added explicitly, const kept
const auto &e = x;  // const int&
auto &&f = x;  // int&  (forwarding reference: lvalue → lvalue ref)
```

The two rules to memorise:

1. **`auto` alone drops `const` and `&`.** `auto x = v[0]` is a *copy*, even if `v[0]` is a reference-returning proxy.
2. **You must add `&` or `const&` yourself** if you want a reference. `auto&` keeps `const`; `auto` does not.

That is why `for (auto x : v)` copies: the deduced type is the element type by value.

### 11.1 When `auto` saves you real time

```cpp
map<string, vector<pair<int,int>>> adj;
for (map<string, vector<pair<int,int>>>::iterator it = adj.begin(); it != adj.end(); ++it) ...  // pain
for (auto it = adj.begin(); it != adj.end(); ++it) ...                                          // same thing
```

Iterator types are long, easy to get wrong, and change when you change the container — `auto` lets you swap `map` for `unordered_map` without touching the loop. It is also the only sane way to name a lambda object, and it is required for structured bindings (`auto [a, b] = p;`), which have no writable type name.

### 11.2 When `auto` hides a bug

**The `vector<bool>` proxy.** `vector<bool>` is a bit-packed specialisation: `v[i]` does not return `bool&`, it returns a proxy object `vector<bool>::reference` that converts to `bool` but also *writes back to the vector*.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<bool> bits = {true, false, true};

    auto proxy = bits[0];   // type is vector<bool>::reference — NOT bool
    proxy = false;          // modifies bits[0]!
    cout << boolalpha << bits[0] << '\n';

    bool copy = bits[1];    // real bool copy
    copy = true;            // bits[1] unchanged
    cout << bits[1] << '\n';

    return 0;
}
```

```text
false
false
```

`auto` deduced the proxy, so `proxy = false` wrote through to the container. If you had written `bool proxy = bits[0]`, the assignment would have been local. Neither is "wrong" — but you must know which one you got. The same trap appears in `for (auto b : bits) b = true;` (does nothing) versus `for (auto &&b : bits) b = true;` (actually flips the bits).

Other `auto` hazards worth a mental note:

| Code | Deduced type | Why it hurts |
|---|---|---|
| `auto n = v.size();` | `size_t` (unsigned) | `n - 1` underflows at 0; comparisons with `int` warn |
| `auto s = a[i] + b[j];` | `int` even if you wanted `long long` | overflow if the sum exceeds $2^{31}-1$; write `1LL * a[i] + b[j]` |
| `auto x = bits[0];` | `vector<bool>::reference` | writes through to the container, cannot take its address |
| `auto it = v.begin();` then `v.push_back(...)` | iterator | invalidated → UB |
| `auto p = mp["missing"];` | `int` (default value) | *inserts* the key, growing the map inside a read-only loop |
| `const auto &m = min({a,b,c});` | `const int&` | binds to a temporary destroyed at the end of the full expression → dangling |

The last one is worth spelling out: `std::min` with an initializer list returns a *reference into a temporary array*. Copy it immediately — `int m = min({a,b,c});` — never bind a reference.

### 11.3 `auto` with iterators and pairs

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {3, 1, 4, 1, 5};

    for (auto it = v.begin(); it != v.end(); ++it) cout << *it << ' ';   // iterator: no type noise
    cout << '\n';

    pair<int, string> p = {7, "seven"};
    auto [num, name] = p;              // C++17 structured binding
    cout << num << ' ' << name << '\n';
    return 0;
}
```

```text
3 1 4 1 5
7 seven
```

---

## 12. Looping over maps: structured bindings and key immutability

```cpp
map<string, int> mp;
for (auto &[key, value] : mp) value++;          // mutate values
for (const auto &[key, value] : mp) cout << key << ' ' << value << '\n';
```

Structured bindings (C++17) make map iteration readable. Four facts matter in contests:

- **You cannot modify `key`.** `for (auto &[k, v] : mp) k = "x";` does not compile — keys are `const` inside a `map`, because changing one would corrupt the tree order. Erase and re-insert instead.
- **Order is sorted** for `map`/`set`, unspecified for `unordered_map`. Traversal is $\mathcal{O}(n)$ total (each `++` is amortised $\mathcal{O}(1)$), but building the map cost $\mathcal{O}(n \log n)$.
- **Never insert while iterating**: `mp[newKey] = 1` can rebalance and invalidate the iterator. Buffer the new entries and insert after the loop.
- `unordered_map` traversal is ~2–3× faster when order does not matter; [Topic 13](Topic%2013%20-%20STL%20Part%202%20%28Set%20&%20Multiset,%20Map%20&%20MultiMap,%20Unordered%20Set%20and%20Unordered%20Maps%29.md) covers the trade-off.

---

## 13. Infinite loops, on purpose

An infinite loop with a `break` is the clean way to write "loop until you find the answer" without inventing a fake condition:

```cpp
while (true) {
    int x; cin >> x;
    if (x == 0) break;
    process(x);
}

for (;;) { if (done()) break; step(); }   // same thing, shorter
```

Both are idiomatic in interactive problems and simulations. The danger is an infinite loop *by accident*, and there are three causes you will actually meet: `continue` in a `while` that skips the increment (§7); an unsigned counter tested with `>= 0` (§6); and a loop whose condition depends on a variable you modified only through a *copy* (`for (auto x : v) x--;`). If a solution hangs, ask "which loop, and what changes to stop it?", then print the loop variable every $10^6$ iterations.

---

## 14. `goto`, unrolling, and pragmas: the sharp tools

### 14.1 `goto` — legal, discouraged, occasionally justified

C++ still has `goto`. It cannot jump into a scope past an initialisation, which kills most of the old C uses. The one surviving CP use is escaping nested loops:

```cpp
for (int i = 0; i < n; i++)
    for (int j = 0; j < m; j++)
        if (a[i][j] == target) goto found;
// not found
found:;
```

You can almost always do better with a flag or a function that `return`s. Use `goto` only when the alternative is harder to read — and never backwards (that is a loop, written badly).

### 14.2 Loop unrolling — a teaser

The CPU can start several independent iterations at once, so processing four elements per step sometimes helps:

```cpp
long long sum = 0;
int i = 0;
for (; i + 3 < n; i += 4) sum += a[i] + a[i+1] + a[i+2] + a[i+3];
for (; i < n; i++) sum += a[i];          // leftover elements — mandatory
```

`-O2` already unrolls simple loops, so this rarely changes contest timings outside very tight numeric kernels. Recognise it, don't reach for it; and never forget the leftover loop when $n$ is not a multiple of 4.

### 14.3 `#pragma GCC optimize` — a warning, not a technique

You will see this at the top of other people's submissions:

```cpp
#pragma GCC optimize("O3,unroll-loops")
#pragma GCC target("avx2,bmi,bmi2,popcnt,lzcnt")
```

It is **not portable**: Clang and MSVC ignore it, and judges that compile with Clang gain nothing. The judge already uses `-O2`, so the real gain is 0–20% and sometimes negative. `#pragma GCC target` can emit AVX2 instructions that **crash on an older judge CPU**, and some setters disallow pragmas outright. Write a faster algorithm instead; if you micro-optimise, keep a pragma-free copy to submit if the judge rejects it.

---

## 15. The `endl` disaster

`endl` is not "newline" — it is "newline **and flush**". Flushing means a system call per line. Fragment:

```cpp
const int N = 10000000;
for (int i = 0; i < N; i++) cout << i << '\n';    // buffered
for (int i = 0; i < N; i++) cout << i << endl;    // 10^7 system calls
```

Typical local result (your numbers will differ, the ratio will not):

```text
with \n   : 78 ms
with endl: 2140 ms
```

A 25× difference from one identifier. On a 1-second limit, `endl` in a $10^7$-line loop is an automatic TLE that will look like a slow algorithm. **Always `'\n'`.** (Full I/O treatment is in [Topic 03](Topic%2003%20-%20Input,%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20%28VERY%20IMPORTANT%29.md).)

### 15.1 Timing your loops properly

`chrono::steady_clock` is the right clock — monotonic, unaffected by system time changes. Measure, do not guess:

```cpp
auto t0 = chrono::steady_clock::now();
// ... work ...
auto ms = chrono::duration_cast<chrono::milliseconds>(
              chrono::steady_clock::now() - t0).count();
cerr << ms << " ms\n";
```

Print to `cerr` so timing never pollutes the judged answer, and remember local timings are optimistic: judge machines are slower and shared.

---

## 16. The loop-pattern cheat sheet

Learn these seven shapes cold. Almost every loop you write in a contest is one of them with different names.

| Pattern | Shape | Cost | Watch out for |
|---|---|---|---|
| Enumerate | `for (int i = 0; i < n; i++)` | $\mathcal{O}(n)$ | `i <= n` |
| Reverse | `for (int i = n - 1; i >= 0; i--)` | $\mathcal{O}(n)$ | unsigned counter |
| Step by $k$ | `for (int i = 0; i < n; i += k)` | $\mathcal{O}(n/k)$ | last partial block |
| Two pointer | `for (int i = 0, j = n-1; i < j; i++, j--)` | $\mathcal{O}(n)$ | pointers crossing |
| Sliding window | `for (int r = 0, l = 0; r < n; r++) { … while (…) l++; }` | $\mathcal{O}(n)$ | `while` must shrink, not grow |
| Digit extraction | `while (x > 0) { int d = x % 10; x /= 10; }` | $\mathcal{O}(\log_{10} x)$ | `x == 0` and negatives |
| Grid traversal | `for (int i = 0; i < n; i++) for (int j = 0; j < m; j++)` | $\mathcal{O}(nm)$ | swapped indices, cache order |

Digit extraction deserves its own line because it is the backbone of every "properties of a number" problem:

```cpp
long long x = 9876543210LL;
int digits = 0, sum = 0;
while (x > 0) { sum += x % 10; digits++; x /= 10; }
cout << digits << ' ' << sum << '\n';   // 10 45
```

Two edge cases: for `x == 0` the loop reports 0 digits (the answer is 1), and for negative `x` the loop never runs. Handle both explicitly — `if (x == 0) return 1;`, `x = llabs(x);` — rather than hoping the tests are positive.

---

## Key Takeaways

- `for` when the count is known, `while` when the exit depends on data, `do-while` only when the body must run at least once — it runs even if the condition is false initially.
- The number of integers in `[0, n)` is $n$; that sentence resolves most off-by-one bugs before they happen.
- Nested iterations multiply: $10^5$ squared is $10^{10}$, which is 100 seconds, which is a TLE no pragma can fix.
- Budget from the constraints: $n \le 10^5$ means $\mathcal{O}(n \log n)$; $n \le 10^3$ means $\mathcal{O}(n^2)$ is intended; $n \le 20$ means bitmask.
- Never write `for (unsigned i = n-1; i >= 0; --i)` — it is an infinite loop. Use `for (int i = n-1; i >= 0; --i)` or `for (size_t i = v.size(); i-- > 0; )`.
- `continue` in a `while` skips your hand-written increment; in a `for` it does not, because the step still runs.
- `for (auto x : v)` copies every element — free for `int`, ruinous for `string`. Default to `const auto&`.
- `auto` drops `const` and `&`. On `vector<bool>` it deduces a proxy that writes through to the container.
- Structured bindings (`auto &[k, v]`) are the clean way to loop over maps; keys are immutable, and you must not insert during iteration.
- Use `'\n'`, never `endl`, in any loop that produces more than a few hundred lines.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `for (i = 0; i <= n; i++)` over `a[0..n-1]` | Reads/writes `a[n]`; steals the next test's input | Bounds say `[0, n)` — condition is `i < n` |
| 2 | `for (unsigned i = n-1; i >= 0; --i)` | Unsigned never goes below 0 → infinite loop, huge index | Signed counter, or `i-- > 0` idiom |
| 3 | `continue` inside `while` before the increment | Increment skipped → infinite loop | Move increment to top of body, or use `for` |
| 4 | Double loop at $n = 10^5$ | $10^{10}$ iterations ≈ 100 s | Change algorithm; check constraints before coding |
| 5 | `for (auto s : vector<string>)` then modify | Copies each string; changes are lost | `for (auto &s : ...)` |
| 6 | `auto b = bits[i]` on `vector<bool>` | Deduces proxy; assignment writes to the vector | `bool b = bits[i]` for a copy; `auto&&` to write |
| 7 | `auto n = v.size(); n - 1` when empty | `size_t` underflow to `SIZE_MAX` | Use `(int)v.size()` or guard `if (v.empty())` |
| 8 | `const auto& m = min({a,b,c});` | Binds to a temporary destroyed at end of statement | Copy into a value: `int m = min({a,b,c});` |
| 9 | `cout << x << endl` in a $10^7$ loop | Flush per line, 20–30× slower | `'\n'` |
| 10 | `i < v.size()` without a cast | `-Wsign-compare` warning, `-Werror` failure | Cast once into a local `m` |
| 11 | Reading $n$ edges for an $n$-vertex tree | Input desynchronised for every later test | $n-1$ edges |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Print a pattern with nested loops; exact row/column counts | CSES "Introductory Problems" / any beginner sheet | Easy |
| 2 | Sum of digits, count digits, reverse a number | AtCoder ABC A/B tasks | Easy |
| 3 | FizzBuzz-style `continue`/modulo classification | Codeforces Div. 3 A | Easy |
| 4 | Two-pointer pair sum on a sorted array | Codeforces Div. 2 A/B, CSES "Sum of Two Values" | Easy–Medium |
| 5 | Longest subarray with sum $\le S$ (positive values) | AtCoder ABC (two pointers) | Medium |
| 6 | Count pairs with $a_i + a_j = k$ in $\mathcal{O}(n \log n)$ after sorting | Codeforces Div. 2 B | Medium |
| 7 | Digit DP warm-up: count numbers $\le n$ with digit sum $k$ | AtCoder Educational DP (digit DP) | Medium–Hard |
| 8 | Simulate a process until it stabilises (`while` + `break`) | Codeforces Div. 2 B (simulation) | Medium |
| 9 | Output $10^6$ lines within 1 s (fast output drill) | Any judge, self-imposed | Easy but instructive |

---

## Next Up

→ [Topic 08 — Useful Math functions & Operations on Data](Topic%2008%20-%20Useful%20Math%20functions%20&%20Operations%20on%20Data.md)