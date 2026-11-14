# Topic 11 — Functions, Classes (Basics) and Useful Lambda Methods

A contest solution is not one long `main`. It is a handful of functions with names that state their contract, a couple of structs that bundle data with the operations on it, and — increasingly — lambdas that keep a comparator or a DFS three lines from where it is used. Getting these three tools right is what separates code you can debug at 2 a.m. from code you rewrite from scratch. This chapter covers functions in full, the minimum of `struct`/`class` that competitive programming actually needs, and lambdas including the recursive patterns that make graph code readable without paying for `std::function`.

> **Prerequisites:** [Topic 10 — Memory Management in C++ and Pointers](Topic%2010%20-%20Memory%20Management%20in%20C++%20and%20Pointers.md), [Topic 09 — Arrays](Topic%2009%20-%20Arrays%20%281D,%202D%20and%203D%29.md)
> **You will be able to:**
> - Write a function signature that documents itself: value, `T&`, or `const T&`
> - Return two or three values without out-parameters, using `pair`, `tuple` and structured bindings
> - Write correct recursion with memoisation and know exactly when it will overflow the stack
> - Build a `struct` with constructors and `operator<` that sorts correctly and never violates strict weak ordering
> - Write lambdas with the right capture mode, including recursive DFS with zero overhead
> - Explain why `std::function` can cost you a TLE and what to use instead

---

## 1. Anatomy of a function

A function is a named, reusable transformation from arguments to a result.

```cpp
#include <bits/stdc++.h>
using namespace std;

//  return type   name   parameter list
//      │          │          │
//      ▼          ▼          ▼
long long  rectangleArea(int w, int h) {
    return 1LL * w * h;      // body
}

int main() {
    cout << rectangleArea(3, 4) << '\n';    // 12
    return 0;
}
```

Four parts, each with a CP-relevant decision:

| Part | Your choice | CP consideration |
|---|---|---|
| Return type | `int`, `long long`, `bool`, `double`, `void`, `pair`, `vector` | `1LL * w * h` prevents the 32-bit overflow that `w * h` would suffer |
| Name | verb phrase: `check`, `buildTree`, `canReach` | you will read this at 2 a.m.; be explicit |
| Parameters | type + mode (`T`, `T&`, `const T&`) | wrong mode = silent bug or a 400 KB copy per call |
| Body | the work | a function doing two things is a function you cannot test |

`void` means "returns nothing". It is not "returns garbage" — a `void` function that needs to produce a result must use a reference parameter or a member variable.

```cpp
#include <bits/stdc++.h>
using namespace std;

void printSeparated(const vector<int>& v, char sep) {
    for (size_t i = 0; i < v.size(); i++) {
        if (i) cout << sep;
        cout << v[i];
    }
    cout << '\n';
}

int main() {
    printSeparated({1, 2, 3}, ' ');
    printSeparated({1, 2, 3}, '-');
    return 0;
}
```

```text
1 2 3
1-2-3
```

---

## 2. Declaration vs definition, and why prototypes exist

- A **declaration** (prototype) says *what* the function looks like: `int gcd(int a, int b);`
- A **definition** supplies the body.
- The compiler needs the declaration at the point of call; the linker needs exactly one definition.

```cpp
#include <bits/stdc++.h>
using namespace std;

int gcd(int a, int b);              // declaration — no body, semicolon
bool isPrime(long long n);          // declare before use

int main() {
    cout << gcd(48, 18) << '\n';    // legal: compiler has the prototype
    cout << isPrime(97) << '\n';
    return 0;
}

int gcd(int a, int b) {             // definition — can come after main
    while (b) { int t = a % b; a = b; b = t; }
    return a;
}

bool isPrime(long long n) {
    if (n < 2) return false;
    for (long long d = 2; d * d <= n; d++)
        if (n % d == 0) return false;
    return true;
}
```

```text
6
1
```

In contest code there is exactly one good reason to use a prototype: **mutual recursion**, where `f` calls `g` and `g` calls `f`. Without a forward declaration, one of them is unknown at the point of use.

```cpp
#include <bits/stdc++.h>
using namespace std;

bool isOdd(int n);              // forward declaration breaks the cycle

bool isEven(int n) { return n == 0 ? true : isOdd(n - 1); }
bool isOdd(int n)  { return n == 0 ? false : isEven(n - 1); }

int main() {
    cout << isEven(10) << ' ' << isOdd(7) << '\n';   // 1 1
    return 0;
}
```

Everywhere else, define functions **above** `main` and skip the prototypes. Two copies of the signature is two places to make a typo — and a mismatched prototype is a linker error (`undefined reference to ...`) or, worse, silent implicit conversion.

Signatures to memorise for the common mistakes:

| Wrong | Right | What happens |
|---|---|---|
| `int f();` then `int f(int x) {...}` | one signature only | `undefined reference` |
| `void f(int a[100]);` | `void f(int* a)` (or `vector<int>&`) | array decays; `sizeof` is 8 |
| `long long f(int)` returning `int` | match the types | truncation, wrong answer |
| `bool cmp(a, b)` with `const` params | fine — `const` does not affect the call | overload mismatch if you change it |

---

## 3. Pass by value, by reference, and by `const&`

This is the single decision that most affects both correctness and speed.

```cpp
#include <bits/stdc++.h>
using namespace std;

void byValue(vector<int> v)              { for (int& x : v) x *= 2; }   // edits a COPY
void byRef(vector<int>& v)               { for (int& x : v) x *= 2; }   // edits the ORIGINAL
void byConstRef(const vector<int>& v)    { /* read only */ }

int main() {
    vector<int> a = {1, 2, 3};

    byValue(a);
    for (int x : a) cout << x << ' ';
    cout << '\n';                          // 1 2 3   ← "my function modified nothing"
    byRef(a);
    for (int x : a) cout << x << ' ';
    cout << '\n';                          // 2 4 6
    return 0;
}
```

That first `1 2 3` is the classic beginner bug: the function *did* double the values, in a copy that was destroyed on return. Nothing warns you, because passing by value is perfectly legal — it is just not what you meant.

### 3.1 The cost table

Copying is $\mathcal{O}(n)$ in the container's size: $n \times \text{sizeof}(T)$ bytes, moved at roughly 10 GB/s.

| Argument | Bytes copied | Copy cost (approx) | Cost in $10^4$ calls |
|---|---|---|---|
| `int` | 4 | ~0.3 ns | 3 µs |
| `pair<int,int>` | 8 | ~0.5 ns | 5 µs |
| `vector<int>` of $10^3$ | 4 KB | ~0.4 µs | 4 ms |
| `vector<int>` of $10^5$ | 400 KB | ~40 µs | **0.4 s** |
| `vector<int>` of $10^6$ | 4 MB | ~0.4 ms | **4 s** |
| `string` of $10^5$ chars | 100 KB | ~10 µs | 0.1 s |
| `vector<vector<int>>` with $10^6$ edges | 4 MB + 10^6 allocations | ~5 ms | hopeless |

For scale: a 2-second limit allows about $2 \times 10^8$ simple operations. Losing 0.4 seconds to copies that a `&` would have eliminated is a 20% tax on your entire budget for one missing character.

### 3.2 The decision rule

| Parameter | Mode | Reason |
|---|---|---|
| `int`, `long long`, `double`, `char`, `bool` | `T` | fits in a register; a reference would force memory access |
| small `struct` ($\le$ 16 bytes), `pair<int,int>` | `T` | same |
| big object you must not modify | `const T&` | no copy, compiler enforces read-only |
| big object you must modify | `T&` | no copy, changes visible to caller |
| big object you need a local scratch copy of | `T` | deliberate copy, documented by the signature |
| output parameter | `T&` | see §5 |

```cpp
#include <bits/stdc++.h>
using namespace std;

long long sumOf(const vector<int>& v) {          // read-only, zero copies
    long long s = 0;
    for (int x : v) s += x;
    return s;
}

void normalise(vector<int>& v) {                  // modified in place
    int mn = *min_element(v.begin(), v.end());
    for (int& x : v) x -= mn;
}

int main() {
    vector<int> v = {5, 9, 2, 7};
    cout << sumOf(v) << '\n';                     // 23
    normalise(v);
    for (int x : v) cout << x << ' ';
    cout << '\n';                                  // 3 7 0 5
    return 0;
}
```

`const T&` also accepts temporaries, which `T&` cannot:

```cpp
cout << sumOf({1, 2, 3, 4}) << '\n';   // fine with const&, compile error with plain &
```

*One missing `&` on a `vector` parameter, called inside a loop, is a TLE that looks like a slow algorithm. Print your function signatures before you profile your complexity.*

---

## 4. Default arguments, overloading, `inline`, `static`

### 4.1 Default arguments

```cpp
#include <bits/stdc++.h>
using namespace std;

long long power(long long a, long long b, long long mod = 1'000'000'007) {
    long long r = 1 % mod;
    a %= mod;
    while (b > 0) {
        if (b & 1) r = r * a % mod;
        a = a * a % mod;
        b >>= 1;
    }
    return r;
}

int main() {
    cout << power(2, 10) << '\n';              // 1024   (mod defaults to 1e9+7)
    cout << power(2, 10, 1000) << '\n';        // 24
    return 0;
}
```

Rules: defaults must be **trailing** (you cannot default the first parameter and supply the second), and they are filled in at the call site, not at the definition. A default that depends on runtime state is a trap — write two overloads instead.

### 4.2 Overloading

Two functions with the same name and different parameter lists are different functions; the compiler picks by the arguments' types.

```cpp
#include <bits/stdc++.h>
using namespace std;

int    add(int a, int b)          { return a + b; }
long long add(long long a, long long b) { return a + b; }
double add(double a, double b)    { return a + b; }

int main() {
    cout << add(2, 3) << '\n';               // 5      — int overload
    cout << add(2LL, 3000000000LL) << '\n';  // 3000000002
    cout << add(1.5, 2.5) << '\n';           // 4      — double overload
    return 0;
}
```

Overload resolution prefers exact matches, then promotions (`int` → `long long`), then conversions. That ordering is why `add(2, 3)` with only the `long long` and `double` versions is ambiguous-ish and why `f(NULL)` can pick the `int` overload. **`const` on a by-value parameter does not create a new overload** (`int f(int)` and `int f(const int)` are the same function), but `const` on a reference parameter does (`f(vector<int>&)` vs `f(const vector<int>&)`).

### 4.3 `inline`

`inline` is a hint — and for functions defined in a header, a requirement to avoid duplicate-symbol errors. Modern compilers inline aggressively at `-O2` regardless of the keyword, so in a single-file solution `inline` changes nothing. Do not sprinkle it; do not expect it to make a slow function fast. The only thing that reliably reduces call overhead is smaller functions and fewer virtual/indirect calls.

### 4.4 `static` functions

At file scope, `static` gives a function **internal linkage**: it is invisible outside this translation unit. In a single-file contest solution that is a no-op for behaviour, useful only to silence "unused function" warnings and to keep names from clashing when you paste two solutions together.

```cpp
static bool sameParity(int a, int b) { return (a & 1) == (b & 1); }   // file-local helper
```

---

## 5. Returning multiple values

C++ returns exactly one value. There are five ways around it, and you should reach for them in this order.

### 5.1 `pair` (two values)

```cpp
#include <bits/stdc++.h>
using namespace std;

pair<int, int> minMaxOf(const vector<int>& v) {
    int mn = v[0], mx = v[0];
    for (int x : v) { mn = min(mn, x); mx = max(mx, x); }
    return {mn, mx};
}

int main() {
    auto p = minMaxOf({3, 1, 4, 1, 5});
    cout << p.first << ' ' << p.second << '\n';      // 1 5
    return 0;
}
```

### 5.2 `tuple` + `tie` (three or more), and structured bindings (C++17)

```cpp
#include <bits/stdc++.h>
using namespace std;

tuple<int, int, long long> analyse(const vector<int>& v) {
    int mn = *min_element(v.begin(), v.end());
    int mx = *max_element(v.begin(), v.end());
    long long s = accumulate(v.begin(), v.end(), 0LL);
    return {mn, mx, s};
}

int main() {
    // C++11 style
    int mn, mx; long long s;
    tie(mn, mx, s) = analyse({3, 1, 4, 1, 5});
    cout << mn << ' ' << mx << ' ' << s << '\n';      // 1 5 14

    // C++17 style — the CP favourite
    auto [lo, hi, sum] = analyse({1, 2, 3});
    cout << lo << ' ' << hi << ' ' << sum << '\n';    // 1 3 6
    return 0;
}
```

`tie` with `ignore` is handy when you want only one component: `tie(ignore, mx, ignore) = analyse(v);`.

Structured bindings also work on `pair`, on `struct`s (public members, in declaration order) and on arrays. They **copy** by default — use `auto& [k, v] : mp` to avoid copying, or `const auto&` when reading a container of `pair`s. Forgetting the `&` in a range-for over a `map` copies every pair, which costs you both time and correctness if you then try to modify through the binding.

### 5.3 A `struct` (named, self-documenting)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Stats { int mn, mx; long long sum; };

Stats analyse(const vector<int>& v) {
    Stats r{v[0], v[0], 0};
    for (int x : v) { r.mn = min(r.mn, x); r.mx = max(r.mx, x); r.sum += x; }
    return r;
}

int main() {
    Stats s = analyse({4, 8, 1});
    cout << s.mn << ' ' << s.mx << ' ' << s.sum << '\n';   // 1 8 13
    return 0;
}
```

### 5.4 Out-parameters (the C style — use sparingly)

```cpp
bool tryFind(const vector<int>& v, int target, int& index) {
    for (size_t i = 0; i < v.size(); i++)
        if (v[i] == target) { index = (int)i; return true; }
    return false;
}
```

Out-parameters earn their place when the "second value" is only meaningful if the function succeeded. Otherwise they make the call site unreadable — the reader cannot tell which argument is input and which is output.

| Approach | Values | Call-site readability | CP verdict |
|---|---|---|---|
| `pair` | 2 | good with `auto [a,b]` | default choice |
| `tuple` | 3–4 | good with structured bindings | fine |
| `struct` | any | best (named fields) | best for >4 or when reused |
| out-parameters | any | poor | only for success/failure + result |
| global variables | any | worst | acceptable for `n`, `adj` in a single solve |

---

## 6. Recursion

A recursive function calls itself on a **smaller** instance. Two parts are mandatory:

1. **Base case** — an input small enough to answer directly, returned without recursing.
2. **Recursive case** — reduce the problem and combine the sub-results.

```cpp
#include <bits/stdc++.h>
using namespace std;

long long factorial(int n) {
    if (n <= 1) return 1;              // base case
    return n * factorial(n - 1);       // recursive case
}

int main() {
    cout << factorial(10) << '\n';     // 3628800
    return 0;
}
```

Each call pushes a frame. `factorial(5)` builds five frames, then unwinds. The recursion tree of a naive Fibonacci is the canonical demonstration of what goes wrong:

```cpp
#include <bits/stdc++.h>
using namespace std;

long long fibSlow(int n) {
    if (n <= 1) return n;
    return fibSlow(n - 1) + fibSlow(n - 2);
}

int main() {
    for (int n = 20; n <= 45; n += 5)
        cout << "fib(" << n << ") = " << fibSlow(n) << '\n';
    return 0;
}
```

```text
fib(20) = 6765
fib(25) = 75025
fib(30) = 832040
fib(35) = 9227465
fib(40) = 102334155
fib(45) = 1134903170
```

The numbers double every 5 steps because the call count grows like $\varphi^n \approx 1.618^n$: about $1.6 \times 10^9$ calls at $n = 45$. The same function with memoisation is $\mathcal{O}(n)$:

$$T(n) = T(n-1) + T(n-2) \;\Rightarrow\; \Theta(\varphi^n), \qquad T_{\text{memo}}(n) = \Theta(n)$$

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<long long> memo;

long long fib(int n) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];       // already solved: return it
    return memo[n] = fib(n - 1) + fib(n - 2); // solve once, store, return
}

int main() {
    int n = 90;
    memo.assign(n + 1, -1);
    cout << fib(n) << '\n';                   // 2880067194370816120
    return 0;
}
```

```text
2880067194370816120
```

Three details that matter:

- `memo.assign(n + 1, -1)` must happen **before** the first call, and `-1` is a valid "not computed" marker because Fibonacci values are non-negative.
- Use `long long`: `fib(93)` exceeds `long long`. If you need more, the answer is modular arithmetic, not `double`.
- The memo table can be a global so the function keeps a clean signature — this is one of the few places global state is the *better* design, because it is genuinely a property of the computation, not of the arguments.

### 6.1 Recursion depth

A frame costs roughly 32–64 bytes for a simple function, more with locals. With an 8 MB stack, that is depth $\sim 10^5$–$2\times10^5$; on Windows' 1 MB stack, $\sim 10^4$–$3\times10^4$. Two consequences, shown as fragments (both read a global `vector<vector<int>> adj`):

```cpp
// Depth is log2(n): safe for any n you can store.
void build(int node, int l, int r) {
    if (l == r) return;
    int mid = (l + r) / 2;
    build(2 * node, l, mid);
    build(2 * node + 1, mid + 1, r);
}

// Depth is n: crashes for n = 10^6 on a path graph.
void dfs(int u, int p) {
    for (int v : adj[u]) if (v != p) dfs(v, u);
}
```

| Recursion shape | Depth | Safe? |
|---|---|---|
| Divide and conquer (segment tree, merge sort) | $\log_2 n$ | always |
| Binary search, Euclid's gcd | $\log n$ | always |
| Tree DFS on a random tree | $\mathcal{O}(\log n)$ expected | usually |
| Tree DFS on a **path** | $n$ | crash at $\sim 2\times10^5$ |
| Grid flood fill | $n \cdot m$ worst case | crash on big grids |
| Naive Fibonacci | $n$ (depth), $\varphi^n$ (calls) | depth fine, time fatal |

For recursion that might be deep, either raise the limit while testing (`ulimit -s unlimited` on Linux, `-Wl,--stack=268435456` with MinGW g++) or convert to an explicit stack:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 6;
    vector<vector<int>> adj(n + 1);
    auto addEdge = [&](int u, int v) { adj[u].push_back(v); adj[v].push_back(u); };
    addEdge(1, 2); addEdge(2, 3); addEdge(3, 4); addEdge(4, 5); addEdge(5, 6);

    vector<int> order;
    vector<int> st = {1};
    vector<char> seen(n + 1, 0);
    seen[1] = 1;
    while (!st.empty()) {
        int u = st.back(); st.pop_back();
        order.push_back(u);
        for (int v : adj[u]) if (!seen[v]) { seen[v] = 1; st.push_back(v); }
    }
    for (int u : order) cout << u << ' ';
    cout << '\n';                  // 1 2 3 4 5 6
    return 0;
}
```

### 6.2 Recursion vs iteration

| | Recursion | Iteration |
|---|---|---|
| Readability for trees/graphs/divide-and-conquer | excellent | poor — you manage the stack |
| Constant factor | higher (frame setup) | lower |
| Deep inputs | stack overflow | safe |
| Backtracking (permutations, N-queens) | natural | painful |
| Tail-recursive loops | some compilers optimise, g++ at `-O2` often does | write the loop instead |

Rule: recurse when the *structure* is recursive (trees, divide-and-conquer, backtracking); iterate when the structure is linear (arrays, DP over indices).

---

## 7. `struct` and `class`: one real difference

```cpp
#include <bits/stdc++.h>
using namespace std;

struct A { int x = 1; };      // members public by default
class  B { int x = 1; };      // members private by default

int main() {
    A a;
    cout << a.x << '\n';      // 1 — accessible
    B b;
    // cout << b.x;           // error: 'int B::x' is private
    return 0;
}
```

That is it. `struct` and `class` are the same language feature with a different default access level. Everything else — constructors, destructors, methods, inheritance, operator overloading — is identical.

Competitive programming uses `struct` almost exclusively, because:

- everything you need to touch is public,
- aggregate initialisation works: `Point p{3, 4};`,
- a plain struct is trivially copyable, so it can go in a `vector`, be passed by value cheaply, and be sorted.

Use `class` only when you want to enforce an invariant (a `ModInt` that must never hold a value $\ge$ MOD). In practice even that is written as a `struct` with a private field and a checked constructor.

### 7.1 A `struct` as a data bundle

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    long long x, y;
};

struct Edge {
    int to;
    long long w;
};

int main() {
    Point p{3, 4};
    Point q = p;                    // trivially copyable
    q.x = 100;
    cout << p.x << ' ' << q.x << '\n';    // 3 100

    vector<Edge> edges = {{2, 5}, {3, 7}};
    for (const Edge& e : edges) cout << e.to << ':' << e.w << ' ';
    cout << '\n';                          // 2:5 3:7
    return 0;
}
```

For a tree or graph, the CP-idiomatic node is an index into parallel arrays, not a pointer-linked object:

```cpp
struct Node {
    int left = -1, right = -1;   // children as INDICES, -1 = absent
    long long val = 0;
    int size = 1;
    void pull() { /* recompute from children */ }
};

vector<Node> tree;               // tree[i] is node i; no new, no delete
```

Indices survive reallocation; pointers do not. Indices also serialise, debug-print, and pool-allocate for free.

---

## 8. Constructors, initialiser lists, and `this`

A constructor runs automatically when an object is created. Three kinds matter:

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    long long x, y;

    Point() : x(0), y(0) {}                        // default
    Point(long long x_, long long y_) : x(x_), y(y_) {}   // parameterised
};

int main() {
    Point a;            // (0,0)
    Point b(3, 4);      // (3,4)
    Point c{5, 6};      // (5,6) — brace init, prefers the 2-arg ctor
    Point d = b;        // copy constructor, generated automatically

    cout << a.x << a.y << ' ' << b.x << b.y << ' '
         << c.x << c.y << ' ' << d.x << d.y << '\n';
    return 0;
}
```

```text
00 34 56 34
```

### 8.1 Member initialiser lists are not optional style

```cpp
struct Bad {
    const int MOD;
    int& ref;
    Bad(int m) { MOD = m; }        // error: const member must be initialised
};
```

`const` members, reference members, and members without a default constructor **can only** be initialised in the list. And the list is faster even when it is not required, because `x = v;` in the body default-constructs `x` first and then assigns:

```cpp
struct Slow { vector<int> v; Slow(int n) { v = vector<int>(n); } };  // construct empty, then move-assign
struct Fast { vector<int> v; Slow(int n) : v(n) {} };                // construct once, correctly sized
```

Initialisation order follows **declaration order**, not the order you write in the list. Reversing them compiles with a warning and produces nonsense when one member's initialiser reads another.

### 8.2 `this`

Inside a member function, `this` is a pointer to the object the method was called on. You need it when a parameter shadows a member, and when you return the object itself for chaining.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Counter {
    int n = 0;
    Counter& add(int n) {          // parameter shadows the member
        this->n += n;              // this-> disambiguates
        return *this;              // enables chaining
    }
    int value() const { return n; }   // const: does not modify the object
};

int main() {
    Counter c;
    c.add(3).add(4).add(5);
    cout << c.value() << '\n';      // 12
    return 0;
}
```

Mark every method that does not modify the object `const`. It costs nothing and it lets you call the method on a `const` reference — which is how you will pass your structs around.

---

## 9. Operator overloading basics

You can give operators meaning for your own types. In CP, three matter: `operator<` (sorting), `operator==` (comparison, `find`), and arithmetic on a numeric type.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    long long x, y;

    Point operator+(const Point& o) const { return {x + o.x, y + o.y}; }
    Point operator-(const Point& o) const { return {x - o.x, y - o.y}; }
    long long dot(const Point& o) const { return x * o.x + y * o.y; }   // NOT operator* — ambiguous
    long long cross(const Point& o) const { return x * o.y - y * o.x; }

    bool operator==(const Point& o) const { return x == o.x && y == o.y; }
    bool operator!=(const Point& o) const { return !(*this == o); }
    bool operator<(const Point& o) const {                 // lexicographic: x, then y
        if (x != o.x) return x < o.x;
        return y < o.y;
    }
};

int main() {
    Point a{1, 2}, b{3, 4};
    Point c = a + b;
    cout << c.x << ' ' << c.y << '\n';           // 4 6
    cout << a.dot(b) << ' ' << a.cross(b) << '\n'; // 11 -2
    cout << (a == a) << ' ' << (a < b) << '\n';  // 1 1
    return 0;
}
```

### 9.1 Why `operator<` must be a strict weak ordering

`std::sort` requires a comparator that is a **strict weak ordering**. Three properties:

1. **Irreflexive**: `cmp(a, a)` is `false`.
2. **Asymmetric**: if `cmp(a, b)` then `!cmp(b, a)`.
3. **Transitive**: `cmp(a,b) && cmp(b,c)` implies `cmp(a,c)`.

Equality is defined as *incomparability*: `a` and `b` are "equivalent" when `!cmp(a,b) && !cmp(b,a)`. A comparator that returns `true` for equal elements (`<=`) claims everything is less than everything, and introsort's partition logic walks off the end of the array.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v(1000);
    iota(v.begin(), v.end(), 0);
    reverse(v.begin(), v.end());

    // sort(v.begin(), v.end(), [](int a, int b) { return a <= b; });  // WRONG
    sort(v.begin(), v.end(), [](int a, int b) { return a < b; });      // right

    cout << v.front() << ' ' << v.back() << '\n';    // 0 999
    return 0;
}
```

With `a <= b` on a 1000-element vector, g++ typically *segfaults* inside `std::sort` — it writes past the buffer. On smaller inputs it "works", which is worse. The same rule applies to `priority_queue`: a comparator that returns `true` for equality can make the heap indefinite-loop while sifting.

The diagnostic: build with `-D_GLIBCXX_DEBUG`. libstdc++ then checks the comparator's assumptions and aborts with `comparison doesn't meet irreflexivity` instead of corrupting memory.

**Sorting descending** correctly: `return a > b;` — not `a >= b`, not `!cmpAsc(a,b)` applied blindly to a comparator that has ties. For a `pair`-like struct, write the tie-breakers explicitly; a lexicographic `operator<` is automatically a strict weak ordering if each component's `<` is.

### 9.2 `operator==` and `operator!=`

Always define them as a pair, and define `!=` in terms of `==`. In C++20 you can write `bool operator==(const T&) const = default;` and get `!=` for free — mention it, but C++17 submissions still need both.

---

## 10. Worked example: a complete `ModInt`

This is the most useful struct in the entire book. It removes the `% MOD` typo class permanently.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct ModInt {
    static constexpr long long MOD = 1'000'000'007;
    long long v;

    ModInt(long long x = 0) {
        v = x % MOD;
        if (v < 0) v += MOD;
    }

    ModInt& operator+=(const ModInt& o) {
        v += o.v;
        if (v >= MOD) v -= MOD;
        return *this;
    }
    ModInt& operator-=(const ModInt& o) {
        v -= o.v;
        if (v < 0) v += MOD;
        return *this;
    }
    ModInt& operator*=(const ModInt& o) {
        v = v * o.v % MOD;
        return *this;
    }

    ModInt pow(long long e) const {
        ModInt base = *this, res = 1;
        while (e > 0) {
            if (e & 1) res *= base;
            base *= base;
            e >>= 1;
        }
        return res;
    }
    ModInt inv() const { return pow(MOD - 2); }        // MOD is prime

    friend ModInt operator+(ModInt a, const ModInt& b) { return a += b; }
    friend ModInt operator-(ModInt a, const ModInt& b) { return a -= b; }
    friend ModInt operator*(ModInt a, const ModInt& b) { return a *= b; }
    friend bool operator==(const ModInt& a, const ModInt& b) { return a.v == b.v; }
    friend bool operator!=(const ModInt& a, const ModInt& b) { return !(a == b); }
    friend ostream& operator<<(ostream& os, const ModInt& a) { return os << a.v; }
};

int main() {
    ModInt a = 1'000'000'006;      // == -1 mod p
    ModInt b = 3;

    cout << a + b << '\n';         // 2
    cout << a * a << '\n';         // 1
    cout << b.inv() << '\n';       // 333333336
    cout << ModInt(2).pow(10) << '\n';   // 1024
    cout << (ModInt(1) - ModInt(5)) << '\n';   // 1000000003
    return 0;
}
```

```text
2
1
333333336
1024
1000000003
```

Why the `friend` form for `+` and `*`: it lets `2 * a` work, because implicit conversion applies to both operands. A member `operator*` would require the left operand to already be a `ModInt`. Taking the first parameter **by value** lets you modify it in place and return it — the copy is optimised away.

Cost: identical to hand-written `long long` arithmetic — everything is inlined, no virtual calls, no allocation. `ModInt` is used in every combinatorics problem for a reason.

---

## 11. Three ways to sort with a custom order

(This section is about *comparators*. The full `std::sort` treatment — stability, `nth_element`, `lower_bound` — is [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md).)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Item { int w, v; };

// (1) A free function
bool byWeight(const Item& a, const Item& b) {
    if (a.w != b.w) return a.w < b.w;
    return a.v > b.v;                        // tie-break: value descending
}

// (2) A function object (struct with operator())
struct ByRatio {
    bool operator()(const Item& a, const Item& b) const {
        return 1LL * a.v * b.w > 1LL * b.v * a.w;   // v/w descending, no floating point
    }
};

int main() {
    vector<Item> items = {{5, 10}, {2, 3}, {5, 4}, {1, 100}};

    sort(items.begin(), items.end(), byWeight);
    for (auto& it : items) cout << it.w << ':' << it.v << ' ';
    cout << '\n';                            // 1:100 2:3 5:10 5:4

    // (3) A lambda — closest to the use site
    sort(items.begin(), items.end(), [](const Item& a, const Item& b) {
        return a.v > b.v;
    });
    for (auto& it : items) cout << it.w << ':' << it.v << ' ';
    cout << '\n';                            // 1:100 5:10 5:4 2:3

    sort(items.begin(), items.end(), ByRatio{});
    for (auto& it : items) cout << it.w << ':' << it.v << ' ';
    cout << '\n';                            // 1:100 5:10 2:3 5:4
    return 0;
}
```

| Form | Inlined? | Can capture state? | Best for |
|---|---|---|---|
| Free function | only with a known symbol at `-O2` (usually not through the pointer) | no | reusable, named orders |
| Function object (`struct`) | yes | yes (members) | parameterised order, C++03 style |
| Lambda | yes | yes (captures) | one-off, local, the CP default |
| `std::function` | no — indirect call | yes | storing in a container; avoid in sort |

Multi-key sorting: the cleanest is `tie`, which compares lexicographically and is a strict weak ordering by construction.

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return tie(a.w, a.v) < tie(b.w, b.v);       // w ascending, then v ascending
});
```

Beware `return a.w < b.w || a.v < b.v;` — with `a = (1, 9)` and `b = (2, 0)` it returns `true` (correct), but with `a = (2, 0)` and `b = (1, 9)` it also returns `true`. Both `cmp(a,b)` and `cmp(b,a)` true → not asymmetric → undefined behaviour in `sort`.

---

## 12. Lambdas

A lambda is an anonymous function object defined in place. In competitive programming it has become the default for comparators, DFS, and short helpers, because it keeps the code where the logic is.

### 12.1 Syntax

```text
[capture](parameters) -> return_type { body }
    │          │             │
    │          │             └── optional; deduced if omitted, required for some recursive forms
    │          └──────────────── like a normal parameter list
    └─────────────────────────── which outside variables the lambda can see, and how
```

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    auto square = [](int x) { return x * x; };                 // return type deduced: int
    auto half   = [](double x) -> double { return x / 2; };    // explicit
    auto greet  = [] { cout << "hello\n"; };                    // no parameters

    cout << square(7) << '\n';       // 49
    cout << half(5) << '\n';         // 2.5
    greet();
    return 0;
}
```

The lambda's type is a unique, unnamed class type. That is why you must store it with `auto` — you cannot write the type name. (Before C++11 you wrote a `struct` with `operator()`; a lambda is sugar for exactly that, and it is why the compiler can inline it perfectly.)

### 12.2 Captures

A capture list decides what the lambda can access from the enclosing scope, and whether it gets a copy or a reference.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x = 10, y = 20;

    auto byValue = [x] { return x * 2; };            // copy of x, taken HERE
    auto byRef   = [&y] { return ++y; };             // reference to y; modifies the original
    auto mixed   = [x, &y] { return x + y; };        // x copied (10), y by reference
    auto allVal  = [=] { return x + y; };            // copies of both, taken HERE
    auto allRef  = [&] { return x + y; };            // reads the live values

    x = 100;                                          // too late for the copies

    cout << byValue() << '\n';       // 20   the copy still holds 10
    cout << byRef()   << '\n';       // 21   y itself is now 21
    cout << mixed()   << '\n';       // 31   10 + 21
    cout << allVal()  << '\n';       // 30   10 + 20
    cout << allRef()  << '\n';       // 121  100 + 21
    cout << x << ' ' << y << '\n';   // 100 21
    return 0;
}
```

```text
20
21
31
30
121
100 21
```

`[=]` copies at the point the lambda is **defined**, not where it is called — so `allVal` still holds `x = 10` even though `x` became 100 before the call. Define the same lambda after `x = 100` and its copy would hold 100. `[&]` reads the live value whenever it runs, and can write to it.

| Capture | Meaning | Use when |
|---|---|---|
| `[]` | nothing | pure function, comparator |
| `[x]` | copy of `x` | value must not change under you |
| `[&x]` | reference to `x` | you want to modify `x` |
| `[=]` | copy everything used | short lambdas, read-only |
| `[&]` | reference everything used | DFS, in-place helpers (the CP default) |
| `[&, x]` | reference by default, `x` by value | loop-variable capture in async code |

**The dangling-reference trap** is real in CP when you return a lambda from a function that captured a local by reference — the lambda outlives the local. In contest code lambdas are almost always used inside `solve()`, so `[&]` is safe and is what you should write. One specific hazard: capturing a **loop variable** by reference in a lambda stored for later. `for (int i = 0; i < n; i++) v.push_back([&]{ return i; });` — every stored lambda sees the same `i`, and after the loop they all return `n`. Capture by value (`[i]`) if the lambda escapes the iteration.

### 12.3 `mutable`

By default a lambda's `operator()` is `const`, so a by-value capture cannot be modified — even though it is your own copy. `mutable` removes that restriction.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int calls = 0;
    auto counter = [calls]() mutable { return ++calls; };   // modifies its own copy
    cout << counter() << ' ' << counter() << ' ' << counter() << '\n';   // 1 2 3
    cout << calls << '\n';                                   // 0 — original untouched
    return 0;
}
```

`mutable` does not turn a copy into a reference. If you want the caller to see the change, capture by reference and drop `mutable`.

### 12.4 Immediately-invoked lambdas

A lambda you define and call in one expression. In CP it is used to initialise a `const` value with a multi-statement computation, or to build a local scope:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    const int n = [] {
        int x; 
        // pretend this reads from input
        x = 7;
        return x * (x + 1) / 2;
    }();
    cout << n << '\n';                  // 28

    // A local "block" that keeps helper names out of scope
    int total = [&] {
        int s = 0;
        for (int i = 1; i <= 10; i++) s += i;
        return s;
    }();
    cout << total << '\n';              // 55
    return 0;
}
```

In C++17 you also have `std::invoke`, but `[](){...}()` is the idiom you will see in real solutions.

### 12.5 Generic lambdas (C++14)

`auto` parameters make the lambda a template — one lambda for every type you throw at it.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    auto twice = [](auto x) { return x + x; };
    cout << twice(21) << '\n';              // 42
    cout << twice(string("ab")) << '\n';    // abab

    auto cmp = [](const auto& a, const auto& b) { return a.first < b.first; };
    vector<pair<int,int>> v = {{2, 9}, {1, 5}, {3, 1}};
    sort(v.begin(), v.end(), cmp);
    for (auto& p : v) cout << p.first << ' ';
    cout << '\n';                            // 1 2 3

    auto print = [](const auto& container) {
        for (const auto& x : container) cout << x << ' ';
        cout << '\n';
    };
    print(vector<int>{1, 2, 3});
    print(vector<string>{"a", "b"});
    return 0;
}
```

This is the workhorse for comparator lambdas: `[](const auto& a, const auto& b)` works for `pair`, `tuple`, `array`, and your own structs without repeating the type.

### 12.6 Recursive lambdas

A lambda cannot refer to itself by name inside its own body — `auto dfs = [&](int u) { dfs(...); };` fails, because `dfs`'s type is not complete until the initialiser finishes. Two working patterns exist, and the trade-off between them is significant.

**Pattern A — `std::function`.** Declare the variable's type first, so the name exists while the body is being parsed.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 6;
    vector<vector<int>> adj(n + 1);
    auto addEdge = [&](int u, int v) { adj[u].push_back(v); adj[v].push_back(u); };
    addEdge(1, 2); addEdge(2, 3); addEdge(3, 4);
    addEdge(1, 5); addEdge(5, 6);

    vector<int> depth(n + 1, 0);

    function<void(int, int)> dfs = [&](int u, int p) {
        for (int v : adj[u]) {
            if (v == p) continue;
            depth[v] = depth[u] + 1;
            dfs(v, u);                       // the name resolves via the std::function
        }
    };

    dfs(1, 0);
    for (int i = 1; i <= n; i++) cout << depth[i] << ' ';
    cout << '\n';                            // 0 1 2 3 1 2
    return 0;
}
```

**Pattern B — self-passing generic lambda (zero overhead).**

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 6;
    vector<vector<int>> adj(n + 1);
    auto addEdge = [&](int u, int v) { adj[u].push_back(v); adj[v].push_back(u); };
    addEdge(1, 2); addEdge(2, 3); addEdge(3, 4);
    addEdge(1, 5); addEdge(5, 6);

    vector<int> depth(n + 1, 0);

    auto dfs = [&](auto&& self, int u, int p) -> void {
        for (int v : adj[u]) {
            if (v == p) continue;
            depth[v] = depth[u] + 1;
            self(self, v, u);                // pass the lambda to itself
        }
    };

    dfs(dfs, 1, 0);
    for (int i = 1; i <= n; i++) cout << depth[i] << ' ';
    cout << '\n';                            // 0 1 2 3 1 2
    return 0;
}
```

| | Pattern A: `std::function` | Pattern B: self-passing `auto&&` |
|---|---|---|
| Lines to write | 1 declaration + body | 1 declaration + body, one extra parameter |
| Call cost | indirect call, no inlining across the type-erased boundary | fully inlined, identical to a normal recursive function |
| Recursion on $10^6$ nodes | ~2-10x slower; depending on capture size, `std::function` may heap-allocate | same as hand-written `dfs` |
| Compile errors | readable | notorious — a typo inside `self(self, ...)` produces pages of template errors |
| Debuggability | breakpoints work normally | breakpoints work; stack traces show the lambda type |
| C++ standard | C++11 | C++14 (generic lambda) |
| Extra note | return type deduction is fine | **must** write `-> void`: the deduced return type is not known while the body is being compiled |

The performance gap is not theoretical. For each of the $2m$ edge visits on a $10^6$-node tree, `std::function`'s `operator()` does a virtual-ish dispatch through a manager function pointer, and the call cannot be inlined into the loop. Benchmarks on typical CF hardware put the difference at 2-4x for a plain DFS, and worse when the lambda's captures are large enough to defeat the small-object buffer (then each `std::function` construction also allocates). If your DFS is the inner loop of a $10^6$-node problem with a 1-second limit, Pattern B is the difference between AC and TLE.

Use Pattern A when you want a readable signature and the graph is small (say $n \le 10^5$ and a 2-second limit); use Pattern B when the recursion is hot. Both appear constantly in editorials, so you must read both fluently.

Also note: **deep recursion through `std::function` uses more stack per frame** (the wrapper adds a frame), so a depth that survives a hand-written function may overflow with Pattern A.

### 12.7 `std::function` cost warning

`std::function<R(Args...)>` is a type-erasing wrapper: it can hold any callable with that signature, at the price of a vtable-like indirection on every call, a possible heap allocation on construction, and no ability to inline.

| Call form | Cost per call (approx, `-O2`) | Inlined? |
|---|---|---|
| Direct function call | ~1 ns | yes, often inlined away entirely |
| Lambda stored in `auto` | ~0 ns marginal | yes |
| Function pointer | ~1-2 ns | no |
| `std::function` wrapping a lambda | ~10-50 ns | no |
| `std::function` with large captures | + an allocation | no |

That is the "10-100x" number. In a loop of $10^7$ calls — which is what a graph traversal is — 20 ns per call is 0.2 seconds of pure overhead, and 50 ns is 0.5 seconds. It matters. Never use `std::function` as a comparator for `sort` on $10^6$ elements ($\sim 2\times10^7$ comparisons).

Where `std::function` is genuinely fine: storing a handful of callbacks, a menu of operations invoked once each, or a recursive lambda on a small input where clarity wins. The mistake is reaching for it in a hot path because the syntax is familiar.

---

## 13. Built-in bit functions

Three functions, backed by a single CPU instruction on any modern x86 (or a two-instruction sequence otherwise). They replace loops and are the fastest way to write a bit-count.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    unsigned int x = 0b10110100;      // 180

    cout << __builtin_popcount(x) << '\n';    // 4  — number of set bits
    cout << __builtin_clz(x) << '\n';         // 24 — leading zeros in a 32-bit unsigned
    cout << __builtin_ctz(x) << '\n';         // 2  — trailing zeros

    long long big = 1LL << 40;
    cout << __builtin_popcountll(big) << '\n';  // 1
    cout << __builtin_clzll(big) << '\n';       // 23
    return 0;
}
```

| Function | Returns | Type | Undefined for |
|---|---|---|---|
| `__builtin_popcount(x)` | count of 1 bits | `unsigned int` | — |
| `__builtin_popcountll(x)` | count of 1 bits | `unsigned long long` | — |
| `__builtin_clz(x)` | leading zeros | `unsigned int` | `x == 0` |
| `__builtin_clzll(x)` | leading zeros | `unsigned long long` | `x == 0` |
| `__builtin_ctz(x)` | trailing zeros | `unsigned int` | `x == 0` |
| `__builtin_parity(x)` | 1 if popcount is odd | `unsigned int` | — |

`clz` and `ctz` on zero are **undefined behaviour** — on x86 `bsr`/`bsf` leave the destination undefined, so you get a garbage value, not a crash, which is worse. Guard it, and remember the idiom `31 - __builtin_clz(x)` gives $\lfloor \log_2 x \rfloor$ for `x >= 1`.

These are GCC/Clang extensions; MSVC has `__popcnt` / `_BitScanReverse`. On any Codeforces, AtCoder or CSES judge you are on g++, so the built-ins are always available. For the full set (`__builtin_ffs`, `__builtin_parity`) see [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md).

---

## 14. Code organisation for CP

The pattern that wins is boring and consistent:

```cpp
#include <bits/stdc++.h>
using namespace std;

// ---- constants and global state ----
const int MAXN = 200005;

// ---- small, testable functions ----
// ---- solve() ----
// ---- main() ----

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int t = 1;
    cin >> t;                        // comment out for single-test problems
    while (t--) {
        // solve();
    }
    return 0;
}
```

Rules that pay for themselves:

1. **One function, one responsibility.** `readInput()`, `solveCase()`, `printAnswer()`. If you cannot name a function with a verb phrase, it is doing two things.
2. **`solve()` for the logic, `main()` for I/O setup and the test loop.** This makes it trivial to add the `while (t--)` wrapper, and to comment out multi-test handling while debugging.
3. **Globals for problem-sized state (`n`, `adj`, `dp`), parameters for everything reusable.** A function that reads `n` from a global is impossible to unit-test; `isPrime(x)` taking `x` is testable in one line. Deep recursion almost forces globals — accept it there.
4. **Test each function against a brute force on small random inputs.** This is the highest-value habit in this entire book.

```cpp
#include <bits/stdc++.h>
using namespace std;

// fast implementation under test
long long fastCount(long long n) {           // count of x in [1,n] with x % 3 == 0
    return n / 3;
}

// brute force reference
long long bruteCount(long long n) {
    long long c = 0;
    for (long long x = 1; x <= n; x++) if (x % 3 == 0) c++;
    return c;
}

int main() {
    mt19937 rng(12345);
    for (int iter = 0; iter < 1000; iter++) {
        long long n = uniform_int_distribution<long long>(1, 500)(rng);
        if (fastCount(n) != bruteCount(n)) {
            cout << "MISMATCH at n = " << n << ": "
                 << fastCount(n) << " vs " << bruteCount(n) << '\n';
            return 1;
        }
    }
    cout << "all 1000 random tests passed\n";
    return 0;
}
```

```text
all 1000 random tests passed
```

A stress test like this catches the off-by-one in `n / 3` versus `(n + 2) / 3`-style bugs in seconds, on inputs you would never think to write by hand. Write it for `isPrime`, for your binary search, for your DSU, for every comparator.

---

## 15. Worked examples

### 15.1 `isPrime`

```cpp
#include <bits/stdc++.h>
using namespace std;

bool isPrime(long long n) {
    if (n < 2) return false;
    if (n % 2 == 0) return n == 2;              // handle the only even prime
    for (long long d = 3; d * d <= n; d += 2)   // odd divisors only
        if (n % d == 0) return false;
    return true;
}

int main() {
    for (long long n : {1LL, 2LL, 17LL, 91LL, 1000000007LL, 1000000009LL})
        cout << n << " -> " << (isPrime(n) ? "prime" : "composite") << '\n';
    return 0;
}
```

```text
1 -> composite
2 -> prime
17 -> prime
91 -> composite
1000000007 -> prime
1000000009 -> prime
```

$\mathcal{O}(\sqrt{n})$ time, $\mathcal{O}(1)$ space. `d * d <= n` avoids `sqrt` and its floating-point rounding. For $n$ up to $10^{18}$ this takes $\sim 5 \times 10^8$ iterations — too slow; that is when you move to Miller–Rabin.

### 15.2 `power(a, b, mod)` — recursive and iterative

```cpp
#include <bits/stdc++.h>
using namespace std;

// Recursive: mirrors the definition, log2(b) frames deep
long long powerRec(long long a, long long b, long long mod) {
    if (b == 0) return 1 % mod;
    long long half = powerRec(a, b / 2, mod);
    long long res = half * half % mod;
    if (b & 1) res = res * a % mod;
    return res;
}

// Iterative: same result, no stack, the version to submit
long long powerIt(long long a, long long b, long long mod) {
    long long res = 1 % mod;
    a %= mod;
    while (b > 0) {
        if (b & 1) res = res * a % mod;
        a = a * a % mod;
        b >>= 1;
    }
    return res;
}

int main() {
    cout << powerRec(2, 10, 1'000'000'007) << '\n';   // 1024
    cout << powerIt(2, 10, 1'000'000'007) << '\n';    // 1024
    cout << powerIt(3, 0, 7) << '\n';                 // 1
    cout << powerIt(0, 0, 7) << '\n';                 // 1  (by convention)
    cout << powerIt(10, 18, 1'000'000'007) << '\n';   // 49
    return 0;
}
```

Both are $\mathcal{O}(\log b)$. The recursive one is the clearest statement of the idea; the iterative one is what you submit, because `b` can be up to $10^{18}$ — that is only 60 frames, so depth is never the issue, but the loop is smaller and branch-predictable. Note `1 % mod` rather than `1`: it keeps `mod == 1` correct.

### 15.3 `binarySearch` as a function

```cpp
#include <bits/stdc++.h>
using namespace std;

// returns an index of target in a sorted array, or -1
int binarySearch(const vector<int>& a, int target) {
    int lo = 0, hi = (int)a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;        // no overflow even for huge indices
        if (a[mid] == target) return mid;
        if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

int main() {
    vector<int> a = {1, 3, 5, 7, 9, 11};
    cout << binarySearch(a, 7) << '\n';      // 3
    cout << binarySearch(a, 4) << '\n';      // -1
    cout << binarySearch(a, 1) << '\n';      // 0
    cout << binarySearch(a, 11) << '\n';     // 5
    return 0;
}
```

`mid = lo + (hi - lo) / 2` is not paranoia: `(lo + hi) / 2` overflows when both are near `INT_MAX`, which happens in problems with $n$ up to $10^9$ and unbounded value ranges. For the "find the first index with `a[i] >= x`" variant — the one 90% of problems need — use `lower_bound` from [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md).

### 15.4 A `DSU` mini-struct

Twenty lines that show why structs earn their place: the data and the operations stay together, and the invariant (every `parent[x]` chain ends at a root) is maintained in exactly one place.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct DSU {
    vector<int> parent, sz;

    explicit DSU(int n) : parent(n + 1), sz(n + 1, 1) {
        iota(parent.begin(), parent.end(), 0);
    }

    int find(int x) {                        // path halving
        while (parent[x] != x) {
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }

    bool unite(int a, int b) {               // union by size
        a = find(a);
        b = find(b);
        if (a == b) return false;            // already connected
        if (sz[a] < sz[b]) swap(a, b);
        parent[b] = a;
        sz[a] += sz[b];
        return true;
    }

    bool connected(int a, int b) { return find(a) == find(b); }
};

int main() {
    DSU d(6);
    cout << boolalpha;
    cout << d.unite(1, 2) << ' ' << d.unite(2, 3) << ' ' << d.unite(1, 3) << '\n';  // true true false
    cout << d.connected(1, 3) << ' ' << d.connected(1, 4) << '\n';                  // true false
    d.unite(4, 5);
    cout << d.connected(1, 5) << '\n';                                              // false
    return 0;
}
```

```text
true true false
true false
false
```

`explicit DSU(int n)` prevents `DSU d = 6;` from compiling — a one-word guard against an implicit conversion you never want. With path halving plus union by size, $m$ operations on $n$ elements run in $\mathcal{O}(m\,\alpha(n))$, which is effectively $\mathcal{O}(m)$.

### 15.5 A `Timer` struct for measuring your own code

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Timer {
    chrono::high_resolution_clock::time_point start;
    string label;

    explicit Timer(string name = "run") : start(chrono::high_resolution_clock::now()), label(move(name)) {}
    ~Timer() {
        double s = chrono::duration<double>(chrono::high_resolution_clock::now() - start).count();
        cerr << "[" << label << "] " << fixed << setprecision(3) << s << " s\n";
    }
};

int main() {
    {
        Timer t("loop 1e7");
        volatile long long s = 0;
        for (int i = 0; i < 10'000'000; i++) s += i;
    }
    {
        Timer t("sort 1e6");
        vector<int> v(1'000'000);
        mt19937 rng(42);
        for (int& x : v) x = (int)(rng() % 1'000'000);
        sort(v.begin(), v.end());
    }
    return 0;
}
```

```text
[loop 1e7] 0.008 s
[sort 1e6] 0.071 s
```

The constructor starts the clock, the destructor prints to `stderr` (which never corrupts your `stdout` answer). Scope it with `{}` to time a section. This is how you find out that your $\mathcal{O}(n \log n)$ sort is not the bottleneck — the `cin >>` loop with sync still on is.

---

## Key Takeaways

- A function signature is a contract: `T` copies, `T&` aliases and can write, `const T&` aliases and cannot. Decide the mode before writing the body.
- Copying a `vector<int>` of $10^5$ costs ~400 KB and ~40 µs; done $10^4$ times that is 0.4 s — a `&` is worth more than a micro-optimisation.
- Declare functions above `main`; use a prototype only for mutual recursion.
- Default arguments must be trailing; overloading resolves by exact match, then promotion, then conversion.
- `inline` is a hint the optimiser ignores when it disagrees; it does not make slow code fast.
- Return two values as `pair`, three to four as `tuple` with structured bindings, more as a named `struct`. Use out-parameters only for "result + success flag".
- Recursion needs a base case and a strictly smaller recursive case. Naive Fibonacci is $\Theta(\varphi^n)$; memoisation makes it $\Theta(n)$.
- Depth $\sim 2\times10^5$ frames is the practical ceiling on an 8 MB stack; divide-and-conquer depths ($\log n$) are always safe, path-shaped DFS is not.
- `struct` and `class` differ only in default access. Use `struct` with public members for CP; store children as indices, not pointers.
- Initialise `const` members, references, and non-default-constructible members in the initialiser list; order follows declaration order.
- Mark non-mutating methods `const`. `this` disambiguates shadowed parameters and enables `return *this;` chaining.
- `operator<` must be irreflexive, asymmetric and transitive. `a <= b` as a comparator is undefined behaviour and typically segfaults `std::sort`.
- Lambdas: `[&]` is the CP default; `[=]` copies at definition time; `mutable` modifies the lambda's own copy only.
- Recursive lambdas: `function<void(int)>` for readability, `auto&& self` for zero overhead. The latter needs an explicit `-> void` return type.
- `std::function` costs ~10-100x a direct call; never use it as a hot comparator or in a $10^7$-iteration DFS.
- `__builtin_popcount`, `__builtin_clz`, `__builtin_ctz` are single instructions; `clz`/`ctz` on zero are undefined.
- Structure every solution as constants → helpers → `solve()` → `main()`, and stress-test each helper against a brute force on random small inputs.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `void f(vector<int> v)` intending to modify | edits a copy; caller sees nothing, no warning | `void f(vector<int>& v)` |
| 2 | `void f(vector<int> v)` for read-only big data | 400 KB copy per call → TLE in a loop | `const vector<int>&` |
| 3 | Missing base case / non-decreasing recursion | infinite recursion → stack overflow | test with the smallest input; assert progress |
| 4 | Recursive DFS on a $10^6$-node path | stack overflow, "Runtime error on test 5" | explicit stack, or raise depth budget |
| 5 | `memo` never initialised to `-1` | all-zero memo returns 0 for uncomputed states | `memo.assign(n+1, -1)` before the first call |
| 6 | `int` result from a `long long` computation | overflow at $2^{31}$, wrong answer on big tests | `1LL *`, or `long long` return type |
| 7 | `sort` comparator `return a <= b;` | violates asymmetry → out-of-bounds writes inside sort | strict `<`; build with `-D_GLIBCXX_DEBUG` |
| 8 | `return a.w < b.w \|\| a.v < b.v;` | not a strict weak ordering | `tie(a.w, a.v) < tie(b.w, b.v)` |
| 9 | `std::function` comparator in `sort` | indirect call on $2\times10^7$ comparisons | plain lambda |
| 10 | `auto dfs = [&](int u) { dfs(u); }` | name not in scope inside its own initialiser | `function<...>` or self-passing `auto&&` |
| 11 | Self-passing lambda without `-> void` | return type cannot be deduced while compiling the body | write `-> void` |
| 12 | `for (auto [k, v] : mp)` then modifying `v` | binding is a copy | `for (auto& [k, v] : mp)` |
| 13 | Lambda capturing a loop variable by reference and storing it | all stored lambdas see the final value | capture by value `[i]` |
| 14 | Constructor assigning `const`/reference members in the body | compile error | member initialiser list |
| 15 | `DSU(int)` without `explicit` | silent conversion from `int` | `explicit DSU(int n)` |
| 16 | `__builtin_clz(0)` | undefined result, silent garbage | guard `if (x)` |
| 17 | Overloading on `const` for by-value params | same function, redefinition error | `const` only differentiates reference params |
| 18 | Nested function definitions | C++ has none (GCC allows it as an extension) | use a lambda |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Write `isPrime`, `gcd`, `lcm` and stress-test each against a brute force | local exercise | Easy |
| 2 | Implement `power(a,b,mod)` twice (recursive and iterative) and verify both on random $a,b$ | CSES "Exponentiation" | Easy |
| 3 | Write `binarySearch` returning the first index with `a[i] >= x`, test against `lower_bound` | local exercise | Easy–Medium |
| 4 | Time a function taking `vector<int>` by value vs `const&` on $10^5$ elements | local exercise | Easy |
| 5 | Compute `fib(n)` with naive recursion for $n=30,35,40$ and record the times | local exercise | Easy |
| 6 | Convert a naive exponential recursion into memoised form and compare runtimes | AtCoder ABC "Fibonacci"-style DP | Easy |
| 7 | Build a `Point` struct with `+`, `-`, `dot`, `cross`, `==`, `<` and sort $10^5$ points | Codeforces Div. 2 B geometry | Medium |
| 8 | Implement `ModInt` and compute $\binom{n}{k} \bmod 10^9+7$ for $n \le 10^6$ | CSES "Binomial Coefficients" | Medium |
| 9 | Sort structs three ways (function, functor, lambda) and confirm identical output | local exercise | Easy |
| 10 | Reproduce the `a <= b` comparator crash on a 1000-element vector, then fix it | local exercise | Easy |
| 11 | Write a recursive DFS both as `std::function` and as a self-passing lambda; time both on a $10^6$-node tree | Codeforces tree-DP problems | Medium |
| 12 | Implement `DSU` with path compression + union by size; run $10^6$ unions and finds | CSES "Road Construction" | Medium |
| 13 | Write a `Timer` and find the slowest of three candidate loops in one program | local exercise | Easy |
| 14 | Solve a multi-test-case problem with `solve()` + `while (t--)` and confirm no state leaks between cases | Codeforces Div. 2 A/B | Easy |
| 15 | Compute $\lfloor \log_2 x \rfloor$ for $x$ up to $10^{18}$ using `__builtin_clzll`, verify against a loop | local exercise | Easy |

## Next Up

→ [Topic 12 — STL Part 1 (Vectors, DeQue, Lists and Forward Lists)](Topic%2012%20-%20STL%20Part%201%20%28Vectors,%20DeQue,%20Lists%20and%20Forward%20Lists%29.md)