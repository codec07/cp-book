# Topic 10 — Memory Management in C++ and Pointers

Every variable you have ever written lives somewhere in a physical layout the operating system hands to your process, and almost every "wrong answer", "runtime error" and "memory limit exceeded" verdict you will ever receive is a statement about that layout. A competitive programmer who knows where `int a[1000000]` sits, why the same line segfaults inside `main` but not at file scope, and how many bytes a `long long dp[5000][5000]` really costs, debugs in seconds what others stare at for an hour. This chapter builds that mental model, then turns it into a practical toolkit: pointers, references, `const`, and the arithmetic that decides whether your solution fits the limit.

> **Prerequisites:** [Topic 09 — Arrays](Topic%2009%20-%20Arrays%20%281D,%202D%20and%203D%29.md), [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems,%20Variables%20and%20Datatypes.md)
> **You will be able to:**
> - Draw the five-segment memory model and say which declaration lands in which segment
> - Diagnose the "local big array segfaults, global works" bug and fix it three different ways
> - Use `new`/`delete` correctly, and explain to yourself why you should almost never use them in a contest
> - Read any pointer declaration including `const int* const*` without guessing
> - Compute, before submitting, whether your DP table fits a 256 MB limit
> - Walk a six-item checklist that solves the vast majority of segfaults

---

## 1. The memory model: five segments, one picture

A running C++ program is not one flat bag of bytes. The loader maps it into regions with different lifetimes, different permissions, and wildly different size ceilings.

```text
  high addresses
  ┌────────────────────────────────────┐
  │  argv / env strings                │
  ├────────────────────────────────────┤
  │  STACK          grows downward ↓   │  ~1 MB (Windows) to 8 MB (Linux)
  │  local variables, parameters,      │  freed automatically on return
  │  return addresses, saved registers │
  │            ⋮                       │
  │            ⋮  (unmapped gap)       │
  │            ↑  grows upward         │
  │  HEAP                              │  bounded by RAM / RLIMIT_AS
  │  new / delete / malloc / free      │  freed manually (or not at all)
  ├────────────────────────────────────┤
  │  BSS   — zero-initialised globals  │  int cnt; int a[10000000];
  │          and static locals         │  costs 0 bytes in the binary
  ├────────────────────────────────────┤
  │  DATA  — initialised globals       │  int MOD = 1000000007;
  ├────────────────────────────────────┤
  │  TEXT  — machine code, read-only   │  your functions, string literals
  └────────────────────────────────────┘
  low addresses
```

| Segment | What lives there | Lifetime | Written to disk? |
|---|---|---|---|
| text | compiled instructions, string literals, `const` globals with constant initialisers | whole process | yes (read-only) |
| data | globals/statics with a **non-zero** initialiser | whole process | yes |
| bss | globals/statics with **zero** (or no) initialiser | whole process | no — just a size |
| heap | `new`, `malloc`, `vector`'s buffer, `string`'s buffer | until `delete`/`free`/destructor | no |
| stack | locals, parameters, return addresses | until the enclosing block ends | no |

Three facts follow immediately, and each one is worth a verdict:

1. **The stack is small and fixed.** Linux gives a thread 8 MB by default, Windows 1 MB. Exceed it and the guard page triggers a segfault — no exception, no message, just death.
2. **The heap is large but slow-ish and manual.** Every `new` costs an allocator call and every leak is permanent for the process's life.
3. **`.bss` is free real estate.** A zero-initialised global array costs nothing in the executable; the OS maps zero pages lazily and only charges you when you touch them.

That third point is why, in contest code, you will routinely see

```cpp
const int MAXN = 2000005;
int a[MAXN];            // 8 MB, in .bss, zeroed for free
long long pref[MAXN];   // 16 MB, also in .bss
```

sitting above `int main()` with no initialisation at all, and why the same declarations inside `main` would be suicidal.

### 1.1 The `static` keyword, in the two places CP uses it

`static` means "restrict the lifetime or the visibility", depending on where you put it.

```cpp
static int calls = 0;        // file scope: internal linkage — invisible to other .cpp files

int counter() {
    static int n = 0;        // function scope: ONE copy, lives for the whole program
    return ++n;              // returns 1, 2, 3, ... across calls
}
```

In a single-file solution, a file-scope `static` global is identical to a plain global in behaviour; its only real use is to silence "defined but not used" warnings and to avoid name clashes when you paste two snippets together. A function-scope `static` inside `solve()` is occasionally handy for a memo table you do not want to re-declare, but a global is clearer. Both have static storage duration, so both are zero-initialised and both live in `.bss`.

---

## 2. The stack: automatic storage and its ceiling

The stack is a LIFO region with one pointer, the *stack pointer*. Calling a function pushes a **stack frame** (parameters, return address, saved registers, locals); returning pops it. This is why locals cannot survive their function, and why returning `&local` is undefined behaviour.

```cpp
#include <bits/stdc++.h>
using namespace std;

int* makeBad() {
    int x = 42;
    return &x;         // x dies when the function returns
}                       // the returned address now points at dead stack memory

int main() {
    int* p = makeBad();
    cout << *p << '\n'; // undefined behaviour: may print 42, may print garbage, may crash
    return 0;
}
```

It often *works*, because nothing has overwritten that slot yet. That is the worst possible outcome: a latent bug that passes your samples and fails on the judge's third test.

### 2.1 The million-element trap

Here is the single most common runtime error among beginners. Both programs below are "the same", but only one survives.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a[1000000];          // 4 MB on the stack
    for (int i = 0; i < 1000000; i++) a[i] = i;
    cout << a[999999] << '\n';
    return 0;
}
```

```text
Segmentation fault (core dumped)
```

```cpp
#include <bits/stdc++.h>
using namespace std;

int a[1000000];              // 4 MB in .bss — survives fine

int main() {
    for (int i = 0; i < 1000000; i++) a[i] = i;
    cout << a[999999] << '\n';
    return 0;
}
```

```text
999999
```

On Linux with the default 8 MB stack the first version *sometimes* survives; add one more array, or one more level of recursion above it, and it dies. On Windows with 1 MB it dies immediately. Never rely on "it worked on my machine".

There are exactly three fixes, and you should know all three:

| Fix | Code | When to use |
|---|---|---|
| Move it to global scope | `int a[N];` above `main` | the standard CP answer; zero cost, zero initialisation |
| Use the heap via `vector` | `vector<int> a(N);` | when `N` is only known at runtime |
| Enlarge the stack | `-Wl,--stack=268435456` (Windows/g++), `ulimit -s unlimited` (bash) | local testing of deep recursion only — a judge will not do this for you |

That last row deserves a warning: **you cannot fix a judge's stack limit from inside your source file.** If your recursive DFS needs depth $10^6$, either convert it to an explicit stack (`vector<int>` as a manual stack) or accept the crash. A `#pragma comment(linker, ...)` works only with MSVC on Windows.

### 2.2 Recursion depth and stack overflow

Each frame of a simple recursive function costs roughly 32–64 bytes; frames with many locals or a large local array cost far more.

```cpp
#include <bits/stdc++.h>
using namespace std;

int depth(int n) {
    if (n == 0) return 0;
    return 1 + depth(n - 1);
}

int main() {
    cout << depth(100000) << '\n';   // usually OK: ~3-6 MB of frames
    cout << depth(1000000) << '\n';  // usually crashes: ~30-60 MB of frames
    return 0;
}
```

```text
100000
Segmentation fault (core dumped)
```

The practical rule for a 256 MB / 8 MB-stack judge: **keep recursion depth under about $2 \times 10^5$ frames**, and assume a frame with a `int tmp[1000]` local is 4 KB, so depth 2000 of *that* already blows the stack. A recursive segment tree on $n = 10^6$ has depth $\log_2 n \approx 20$ and is perfectly safe; a naive DFS on a path graph of $10^6$ vertices is not.

*In a Codeforces round, `Runtime error on test 5` on a tree problem is almost always a DFS that recursed $10^6$ deep on a path, not an algorithm bug.*

---

## 3. The heap: `new`, `delete`, and the leak you will create

The heap is a pool of memory the allocator hands out on request. In C++ the request is `new`, the release is `delete`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int* p = new int(42);            // one int, value 42
    int* arr = new int[10];          // ten uninitialised ints
    int* zero = new int[10]();       // ten ints, all zero

    for (int i = 0; i < 10; i++) arr[i] = i * i;

    cout << *p << ' ' << arr[9] << ' ' << zero[0] << '\n';

    delete p;                        // scalar delete
    delete[] arr;                    // array delete
    delete[] zero;
    return 0;
}
```

```text
42 81 0
```

### 3.1 The four ways to get it wrong

```cpp
int* a = new int[10];
delete a;          // (1) WRONG: new[] must pair with delete[] — undefined behaviour
delete[] a;        // (2) double free — undefined behaviour, often a crash

int* b = new int[10];
b = new int[10];   // (3) leak: the first block is now unreachable forever

int* c;            // (4) wild pointer: never initialised
*c = 5;            // writes to a random address → segfault or silent corruption
```

`new[]`/`delete[]` mismatch is the nastiest of the four because it usually *appears* to work. The array form records a cookie (element count) before the block, and `delete` without `[]` frees at the wrong offset — you corrupt the allocator's bookkeeping and crash minutes later, somewhere unrelated.

### 3.2 `malloc` and `free`: the C way

```cpp
#include <cstdlib>

int* a = (int*)malloc(10 * sizeof(int));   // uninitialised, no constructors run
if (a == nullptr) { /* allocation failed */ }
free(a);                                    // free(nullptr) is safe; free twice is not
```

| | `new` / `delete` | `malloc` / `free` |
|---|---|---|
| Returns | typed pointer, no cast | `void*`, needs a cast |
| Runs constructors? | yes | no |
| Failure | throws `std::bad_alloc` | returns `nullptr` |
| Zeroing | no (`new int[10]()` does) | no (`calloc` does) |
| Speed | marginally slower | marginally faster |
| CP verdict | avoid | avoid harder |

In C++ there is no reason to use `malloc` for objects with constructors, and no reason to use either one when `vector` exists.

### 3.3 Why competitive programmers avoid `new`

Because `vector` already *is* the heap, with three superpowers bolted on.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    // The new/delete way — 6 lines, 3 chances to leak or mismatch
    int n = 100000;
    int* a = new int[n];
    for (int i = 0; i < n; i++) a[i] = i;
    long long s = 0;
    for (int i = 0; i < n; i++) s += a[i];
    delete[] a;

    // The vector way — 3 lines, zero chances
    vector<int> b(n);
    iota(b.begin(), b.end(), 0);
    long long t = accumulate(b.begin(), b.end(), 0LL);

    cout << s << ' ' << t << '\n';   // 4999950000 4999950000
    return 0;
}
```

`vector` frees itself on scope exit — including during *stack unwinding* when an exception flies past, which raw `new` handles only if you remember `try`/`catch`/`delete` correctly. It also knows its own size, so you never carry a separate `int n` around and never pass the wrong length to a function. And it is the same speed: `vector<int>::operator[]` is a plain load, identical to `a[i]`.

### 3.4 `bad_alloc` and the memory-limit verdict

When the allocator cannot satisfy a request, `new` throws `std::bad_alloc`. On a judge this surfaces as `std::bad_alloc`, `Memory limit exceeded`, or an immediate `Runtime error` depending on the harness.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    try {
        vector<long long> huge(1'000'000'000);   // 8 GB
        cout << huge.size() << '\n';
    } catch (const bad_alloc& e) {
        cout << "allocation failed: " << e.what() << '\n';
    }
    return 0;
}
```

```text
allocation failed: std::bad_alloc
```

Two CP-relevant notes. First, catching `bad_alloc` does not save you: if a single structure needs 8 GB, the algorithm is wrong for the constraints, and retrying with a smaller size means you have not solved the problem. Second, **the process's peak RSS is what the judge measures**, not the sum of live objects. A `vector` that grows to $10^7$ then `clear()`s still peaked at 80 MB, and it never returns the memory to the OS. If you build several large containers in sequence, their peaks add up unless you scope them:

```cpp
{ vector<int> tmp(n); /* ... */ }   // tmp destroyed here, memory released
{ vector<int> other(n); /* ... */ }
```

---

## 4. Static vs dynamic allocation: the decision table

This is the table to memorise. It answers 95% of "how should I declare this" questions.

| Situation | Use | Why |
|---|---|---|
| Size known at compile time, $\le$ a few MB | global `int a[MAXN];` | zero cost, zero init, lives in `.bss` |
| Size known only after reading input | `vector<T> v(n);` | heap, sized at runtime, self-freeing |
| 2D grid, both dimensions from input | `vector<vector<int>> g(n, vector<int>(m));` | simple; one allocation per row |
| 2D grid, tight memory/time, $n \times m \ge 10^6$ | flat `vector<int> g(n*m);` + `g[i*m+j]` | one allocation, best locality |
| Fixed-size 2D DP, dimensions $\le 5000$ | global `long long dp[5005][5005];` | `.bss`, no pointer chasing |
| Recursive tree/graph structure | `vector<vector<int>> adj(n)` | avoids `new Node` and pointer overhead |
| Need a linked list of nodes | `struct Node` in a global array pool | far faster than `new Node` per element |
| Object with a destructor needed across scopes | `unique_ptr` / `vector` | RAII, exception-safe |

The alternative to `new Node` per element — the **array pool** — is worth internalising early, because it is what competitive programmers actually write:

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node { int val, next; };
Node pool[1000005];      // .bss: 8 MB, no allocator calls at all
int ptr = 0;

int newNode(int v) { pool[ptr] = {v, -1}; return ptr++; }

int main() {
    int head = newNode(10);
    int second = newNode(20);
    int third = newNode(30);
    pool[head].next = second;        // link 10 -> 20 -> 30
    pool[second].next = third;

    for (int i = head; i != -1; i = pool[i].next) cout << pool[i].val << ' ';
    cout << '\n';                    // 10 20 30
    return 0;
}
```

No `new`, no `delete`, no fragmentation, and the whole pool is one contiguous region the CPU can prefetch. This is the pattern behind fast treaps, link-cut trees, and any structure where a textbook would say "allocate a node".

---

## 5. Pointers: the address, and the thing at the address

A pointer is an integer that happens to be an address. Two operators do all the work:

- `&x` — *address-of*: gives you the address of `x`.
- `*p` — *dereference*: gives you the object stored at address `p`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x = 10;
    int* p = &x;          // p holds the address of x

    cout << x  << '\n';   // 10          the value
    cout << &x << '\n';   // 0x61ff08    where it lives (varies every run)
    cout << p  << '\n';   // 0x61ff08    same address
    cout << *p << '\n';   // 10          value at that address

    *p = 99;              // write through the pointer
    cout << x  << '\n';   // 99          x changed
    return 0;
}
```

`p` and `&x` are the same value; `*p` and `x` are the same object. That is the entire concept. Everything else is bookkeeping.

### 5.1 `int* p` vs `int *p` — it does not matter, but know why

```cpp
int* a, b;    // a is int*, b is int  ← the trap
int *a, *b;   // both are int*
int* a, * b;  // legal and identical to the line above
```

The `*` binds to the *declarator*, not the type name. `int* a, b;` reads as `int (*a), b;` — `a` is a pointer, `b` is a plain `int`. This is why many style guides write `int *p`. In this book we write `int* p` and **declare one variable per line**, which removes the ambiguity entirely. Never write `int* a, b;` in a contest; you will not notice the bug until the answer is wrong.

### 5.2 Null, `nullptr`, `NULL`, and `0`

| Form | Type | Introduced | Verdict |
|---|---|---|---|
| `nullptr` | `std::nullptr_t` | C++11 | **use this** |
| `NULL` | `long` or `void*` depending on headers | C | avoid; it is a macro |
| `0` | `int` | forever | avoid as a pointer value |

`NULL` is typically `#define NULL 0`, which makes overload resolution go wrong:

```cpp
void f(int);
void f(char*);

f(NULL);       // calls f(int) — surprise!
f(nullptr);    // calls f(char*)
```

**Dereferencing a null pointer is undefined behaviour, and on every real system it is a segfault at address 0.** Checking `if (p)` / `if (p != nullptr)` before dereferencing is cheap insurance when a pointer genuinely may be absent. In contest code, prefer "cannot be absent" designs: an index of `-1` is safer than a null pointer because a wrong index gives a wrong answer, not a crash.

### 5.3 Dangling, wild, and double-free

```cpp
int* dangling() {
    int x = 5;
    return &x;                 // (a) returns address of a dead local
}

int main() {
    vector<int> v = {1, 2, 3};
    int* p = &v[0];
    v.push_back(4);            // (b) may reallocate: p now dangles
    cout << *p << '\n';        // undefined behaviour (use indices, not pointers!)

    int* q;                    // (c) wild: uninitialised
    // *q = 1;                 // would write to a garbage address

    int* r = new int(1);
    delete r;
    delete r;                  // (d) double free
    return 0;
}
```

Rule of thumb for contests: **a pointer into a container is valid only until the container's next structural modification.** `vector` indices survive reallocation; pointers and iterators do not. Prefer indices.

---

## 6. Pointer arithmetic: moving by `sizeof(T)`

Pointer arithmetic is scaled by the size of the pointed-to type. `p + 1` is not "one byte later", it is "the next `T`".

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a[5] = {10, 20, 30, 40, 50};
    int* p = a;                      // array name decays to &a[0]

    cout << *p       << '\n';        // 10
    cout << *(p + 1) << '\n';        // 20
    cout << *(p + 4) << '\n';        // 50

    p += 2;                          // now points at a[2]
    cout << *p << '\n';              // 30
    cout << (p - a) << '\n';         // 2   — difference is in ELEMENTS, not bytes

    cout << sizeof(int) << '\n';     // 4
    return 0;
}
```

```text
10
20
50
30
2
4
```

Subtracting two pointers of the same type gives a `ptrdiff_t` counted in elements. Adding two pointers is illegal — there is no meaning to "address plus address".

### 6.1 Array–pointer duality

An array name, in almost every expression, decays to a pointer to its first element. Therefore:

$$a[i] \;\equiv\; *(a + i) \;\equiv\; *(i + a) \;\equiv\; i[a]$$

That last form compiles. It is legal C++ and completely unreadable; you should never write it, but you should recognise it, because it explains *why* `a[i]` is defined as `*(a+i)` and why indexing works identically on a pointer:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a[4] = {1, 2, 3, 4};
    cout << a[2] << ' ' << *(a + 2) << ' ' << 2[a] << '\n';   // 3 3 3
    return 0;
}
```

The duality has three practical consequences:

1. **`sizeof` lies inside functions.** `void f(int a[])` is really `void f(int* a)`, so `sizeof(a)` is 8 (the pointer), not the array's size. Always pass the length separately.
2. **`end()` is one-past-the-end.** `a + n` points at the element after the last one.
3. **You can iterate with pointers.** `for (int* it = a; it != a + n; ++it)` is exactly what `vector::iterator` does.

```cpp
#include <bits/stdc++.h>
using namespace std;

void broken(int a[]) { cout << sizeof(a) << '\n'; }   // prints 8, not 40
void correct(int a[], int n) {
    for (int i = 0; i < n; i++) cout << a[i] << ' ';
    cout << '\n';
}

int main() {
    int a[10] = {};
    cout << sizeof(a) << '\n';      // 40
    broken(a);                      // 8   ← the classic bug
    correct(a, 10);
    return 0;
}
```

### 6.2 One-past-the-end: legal to form, illegal to dereference

The address `a + n` is a valid pointer value; you may compute it, store it, and compare against it. You may not read or write through it.

```cpp
int a[10];
int* end = a + 10;     // fine
// int x = *end;       // UB: reads past the array
// *end = 5;           // UB: writes past the array

for (int* p = a; p != end; ++p) *p = 0;   // fine: never dereferences end
```

This is exactly the contract of `[begin, end)` half-open ranges, and why `std::find(v.begin(), v.end(), x)` is safe. It is also why an off-by-one loop like `for (int i = 0; i <= n; i++) sum += a[i];` reads `a[n]` — usually harmless, occasionally a segfault, always wrong.

---

## 7. Pointers vs references

A reference is an alias: another name for an existing object. It is implemented as a pointer under the hood but has a much stricter, much safer contract.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x = 10;
    int& r = x;        // r IS x — no & on the right, it is not an address
    r = 20;
    cout << x << '\n';  // 20

    // int& bad;        // error: a reference must be initialised
    // r = y;           // this assigns to x; it does NOT re-seat r
    return 0;
}
```

| Property | `int* p` | `int& r` |
|---|---|---|
| Can be null | yes | no — must bind to an object |
| Can be re-seated | yes (`p = &y;`) | no — always the original object |
| Must be initialised | no | yes |
| Needs `*` to access | yes | no — it *is* the object |
| Pointer arithmetic | yes | no |
| Can point at an array element and move | yes | no (binds to one element) |
| Typical CP use | iteration, optional values, `scanf` | output parameters, avoiding copies |

### 7.1 Pass by reference: two distinct reasons

**Reason 1 — avoid a copy (performance).** A `vector<int>` of $10^5$ elements is 400 KB; copying it on every call costs ~40 µs and, in a loop of $10^5$ calls, 4 seconds. Passing by reference makes it $\mathcal{O}(1)$.

**Reason 2 — return extra values (correctness/expressiveness).** C++ functions return one value. A reference parameter lets you write a second.

```cpp
#include <bits/stdc++.h>
using namespace std;

// read-only, no copy: const reference
long long sumOf(const vector<int>& v) {
    long long s = 0;
    for (int x : v) s += x;
    return s;
}

// modified in place: non-const reference
void scaleAll(vector<int>& v, int k) {
    for (int& x : v) x *= k;
}

// extra output parameter
bool tryDivide(int a, int b, int& out) {
    if (b == 0) return false;
    out = a / b;
    return true;
}

int main() {
    vector<int> v = {1, 2, 3, 4};
    cout << sumOf(v) << '\n';       // 10
    scaleAll(v, 10);
    for (int x : v) cout << x << ' ';
    cout << '\n';                    // 10 20 30 40

    int q;
    if (tryDivide(7, 2, q)) cout << q << '\n';   // 3
    return 0;
}
```

**The classic bug**: you write `void f(vector<int> v)` intending to sort in place, and the caller's vector is untouched. Signatures with a bare copy are silent — nothing warns you. The habit that prevents it: *decide the parameter's mode before writing the body* — read-only? `const T&`. Modified? `T&`. Genuinely copied (small, or you need a local scratch)? `T`.

**`const T&` rule of thumb:** for anything bigger than 16 bytes, pass by `const&`. For `int`, `double`, `char`, `bool`, and tiny structs, pass by value — it is the same speed or faster, because the compiler can keep it in a register instead of forcing it into memory.

### 7.2 Returning a reference is dangerous

```cpp
#include <bits/stdc++.h>
using namespace std;

int& elementAt(vector<int>& v, int i) { return v[i]; }   // OK: refers to caller's memory

int& bad() {
    int x = 5;
    return x;                // UB: reference to a dead local
}

int main() {
    vector<int> v = {1, 2, 3};
    elementAt(v, 1) = 99;                 // legal, and intended
    cout << v[1] << '\n';                 // 99
    return 0;
}
```

Return a reference only when the referred-to object outlives the call (a member, a container element, a global). Otherwise return by value — for `int`s and `pair`s, the copy is free after optimisation; for `vector`s, C++17's guaranteed copy elision and move semantics make `return v;` $\mathcal{O}(1)$ anyway.

---

## 8. `const` correctness: reading declarations right-to-left

`const` attaches to whatever is immediately to its **left**; if nothing is to its left, it applies to whatever is to its right. Read the declaration from the identifier outwards (right to left) and say it out loud:

| Declaration | Right-to-left reading | Meaning |
|---|---|---|
| `const int* p` | `p` is a pointer to `int` that is `const` | the *pointee* is read-only; `p` can move |
| `int* const p` | `p` is a `const` pointer to `int` | the *pointer* is fixed; `*p` can change |
| `const int* const p` | `p` is a `const` pointer to `int` that is `const` | neither can change |
| `int const* p` | identical to `const int* p` | — |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x = 1, y = 2;

    const int* p = &x;    // pointer to const int
    // *p = 10;           // error: cannot write through p
    p = &y;               // fine: p itself is not const
    cout << *p << '\n';   // 2

    int* const q = &x;    // const pointer to int
    *q = 10;              // fine: the int is writable
    // q = &y;            // error: cannot re-seat q
    cout << x << '\n';    // 10

    const int* const r = &y;
    // *r = 5;  r = &x;   // both illegal
    cout << *r << '\n';   // 2
    return 0;
}
```

CP-relevant uses of `const`:

```cpp
const int MOD = 1'000'000'007;         // compile-time constant, no accidental writes
const double PI = 3.14159265358979;
void print(const vector<int>& v);      // promises not to modify, accepts temporaries
int find(const string& s, char c);     // no copy, no mutation
```

A `const` parameter also lets you pass a temporary: `sumOf({1,2,3})` works with `const vector<int>&` but not with `vector<int>&`. And `const` on a global lets the compiler fold it into immediates — genuinely faster than a variable that might be written by something else.

---

## 9. Pointers to pointers, arrays of pointers, function pointers

### 9.1 Pointer to pointer

`int**` is the address of an `int*`. It appears in CP in three places: passing a pointer that a function should modify, C-style 2D arrays, and `argv`.

```cpp
#include <bits/stdc++.h>
using namespace std;

void allocate(int** out, int n) {      // caller wants its pointer changed
    *out = new int[n]();
}

int main() {
    int* a = nullptr;
    allocate(&a, 5);
    a[2] = 7;
    cout << a[2] << '\n';              // 7
    delete[] a;
    return 0;
}
```

`&a` has type `int**`; `*out = ...` writes to the caller's pointer. In modern C++ you would write `void allocate(vector<int>& v, int n)` or return a `vector` — the `int**` version exists mainly so you can read code that uses it.

### 9.2 Arrays of pointers

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    const char* words[] = {"alpha", "beta", "gamma"};   // array of 3 pointers
    for (const char* w : words) cout << w << ' ';
    cout << '\n';                                        // alpha beta gamma

    int a = 1, b = 2, c = 3;
    int* ptrs[3] = {&a, &b, &c};
    *ptrs[1] = 20;
    cout << b << '\n';                                   // 20
    return 0;
}
```

`int* ptrs[3]` is an array of 3 pointers (24 bytes on 64-bit); `int (*p)[3]` is a pointer to an array of 3 ints (8 bytes). The parentheses matter — without them, `[]` binds tighter than `*`.

### 9.3 Function pointers

A function's name decays to a pointer to its code, and you can store and pass it. For sorting this buys you a runtime-selected comparator with zero per-comparison overhead beyond an indirect call.

```cpp
#include <bits/stdc++.h>
using namespace std;

bool ascending(int a, int b)  { return a < b; }
bool descending(int a, int b) { return a > b; }

int main() {
    vector<int> v = {5, 1, 4, 2, 3};

    bool (*cmp)(int, int) = ascending;      // declare a function pointer
    sort(v.begin(), v.end(), cmp);
    for (int x : v) cout << x << ' ';
    cout << '\n';                            // 1 2 3 4 5

    cmp = descending;
    sort(v.begin(), v.end(), cmp);
    for (int x : v) cout << x << ' ';
    cout << '\n';                            // 5 4 3 2 1
    return 0;
}
```

`bool (*cmp)(int,int)` reads right-to-left as "`cmp` is a pointer to a function taking two `int`s and returning `bool`". Use `using Cmp = bool(*)(int,int);` for readability. In practice you will use a lambda (Topic 11) instead — it is inlined, whereas a function pointer is not.

---

## 10. `sizeof`, pointer width, and the `int*` vs `long long*` trap

On a 64-bit machine — which is what every judge uses — **every pointer is 8 bytes**, regardless of what it points to.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << sizeof(char*)      << '\n';   // 8
    cout << sizeof(int*)       << '\n';   // 8
    cout << sizeof(long long*) << '\n';   // 8
    cout << sizeof(void*)      << '\n';   // 8
    cout << sizeof(int)        << '\n';   // 4
    cout << sizeof(long long)  << '\n';   // 8
    return 0;
}
```

The trap is that the *dereference* differs even though the pointer size does not:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a[4] = {1, 2, 3, 4};

    int* p = a;
    cout << *(p + 1) << '\n';                       // 2      — jumps 4 bytes

    long long* q = reinterpret_cast<long long*>(a);  // lie about the type
    cout << *(q + 1) << '\n';                        // jumps 8 bytes: reads a[2] and a[3]
                                                     // reinterpretation, endian-dependent
    return 0;
}
```

`int*` arithmetic strides 4 bytes; `long long*` strides 8. Casting between them does not convert anything, it only changes how many bytes each step covers — so `q + 1` is already at element 2 of the original array, and `q + 2` walks off the end. Every "my pointer arithmetic gives garbage" bug is this.

**The other `sizeof` trap**, already flagged: `sizeof(array)` inside a function is `sizeof(pointer)`. And `sizeof(vector)` is a constant (~24 bytes: three pointers), not the number of elements — that is `v.size() * sizeof(T)`.

---

## 11. Smart pointers: know they exist, then do not use them

`std::unique_ptr<T>` is a pointer that deletes its object when it goes out of scope; `std::shared_ptr<T>` adds reference counting so several owners can share.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    auto p = make_unique<int>(42);
    cout << *p << '\n';          // 42

    auto q = make_shared<vector<int>>(5, 7);   // 5 elements, value 7
    auto r = q;                                 // refcount 2
    cout << q->size() << ' ' << r.use_count() << '\n';   // 5 2
    return 0;
}   // everything freed automatically
```

In competitive programming you almost never need them:

| Reason | Detail |
|---|---|
| `vector` already owns the heap | The only large allocation you make is a container |
| Trees/graphs use indices | `vector<vector<int>>` needs no per-node ownership |
| `shared_ptr` is heavy | Atomic refcount, control block, extra indirection — ~2-3x slower access, 2-3x more memory |
| Depth of recursion | A `unique_ptr` chain of $10^6$ nodes destroys recursively — stack overflow at destruction time |

Where they genuinely help: a persistent/functional data structure shared between versions (segment tree beats variants), or a long-lived local tool. In a 2-second limit with $10^6$ node visits, `shared_ptr`'s atomic increments alone can cost you 30% of your runtime. Say it once, know it, move on.

---

## 12. Memory limit arithmetic: will it fit?

The judge measures **peak resident set size**, in megabytes, including your globals (`.bss` counts as touched once written), your `vector`s, and the allocator's rounding. Budget 90% of the stated limit.

$$ \text{bytes} = N \times \text{sizeof(element)} \times \text{number of tables} $$

**Worked example.** A 256 MB limit, and you want `long long dp[5000][5000]`:

$$5000 \times 5000 \times 8 = 200{,}000{,}000 \text{ B} \approx 190.7 \text{ MiB}$$

Against 256 MiB that leaves ~65 MiB of headroom: fine if this is the only big structure, fatal if you also keep a `long long pref[5000][5000]` (another 190 MiB) or a `vector<vector<int>> g` of the same shape (100 MB, but plus ~5000 separate allocations and their overhead, plus the pointer array). The same table at 128 MB is an instant MLE.

**The size that actually bites** is the innocent-looking one: `long long pref[1000005]` is 8 MB, `vector<vector<int>>` with $10^6$ rows costs 24 MB in `vector` objects alone before storing a single element, and `map<int,int>` with $10^6$ entries costs roughly $10^6 \times 48 \approx 48$ MB.

### 12.1 Max $N$ for a given bytes-per-element and limit

$N_{\max} = \left\lfloor \sqrt{L / b} \right\rfloor$ for a square 2D table of $b$-byte cells under an $L$-byte limit:

| bytes/element | 32 MB | 64 MB | 256 MB | 512 MB |
|---|---|---|---|---|
| 1 (`char`, `bool`) | 5792 | 8192 | 16384 | 23170 |
| 2 (`short`) | 4096 | 5792 | 11585 | 16384 |
| 4 (`int`) | 2896 | 4096 | 8192 | 11585 |
| 8 (`long long`) | 2048 | 2896 | 5792 | 8192 |
| 16 (`pair<long long,long long>`) | 1448 | 2048 | 4096 | 5792 |
| 32 | 1024 | 1448 | 2896 | 4096 |

Read the table as: *under 256 MB, a `long long` DP can be at most about $5792 \times 5792$.* If your recurrence needs $N = 10^4$, you need either 8 tables' worth of memory (impossible) or a rolling-array optimisation that keeps only two rows.

### 12.2 The rolling-array trick, quantified

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    const int N = 10000;
    // Full table: 10000*10000*8 = 800 MB — MLE everywhere
    // vector<vector<long long>> dp(N, vector<long long>(N));

    vector<long long> prev(N + 1), cur(N + 1);   // 160 KB total
    for (int i = 1; i <= N; i++) {
        for (int j = 1; j <= N; j++)
            cur[j] = max(prev[j], cur[j - 1]) + (i ^ j) % 7;
        swap(prev, cur);
    }
    cout << prev[N] << '\n';
    return 0;
}
```

Same answer, 5000x less memory. Whenever a DP recurrence references only row $i-1$ and row $i$, drop the first dimension.

---

## 13. Cache and locality: the free 5x speedup

Memory is not uniform. A modern CPU fetches 64-byte **cache lines**, and a miss to main memory costs ~100 ns while an L1 hit costs ~1 ns. If your loop touches memory in the wrong order, every element access is a miss.

C++ lays out `int g[2000][2000]` in **row-major** order: `g[0][0..1999]` are contiguous, then `g[1][0..1999]`. Iterating `for i, for j` walks memory linearly; iterating `for j, for i` jumps 8000 bytes per step and touches a new cache line every time.

```cpp
#include <bits/stdc++.h>
using namespace std;
using namespace std::chrono;

int main() {
    const int N = 2000;
    static int g[N][N];                       // 16 MB, .bss
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++) g[i][j] = i + j;

    auto t0 = high_resolution_clock::now();
    long long s1 = 0;
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++) s1 += g[i][j];
    auto t1 = high_resolution_clock::now();

    long long s2 = 0;
    for (int j = 0; j < N; j++)
        for (int i = 0; i < N; i++) s2 += g[i][j];
    auto t2 = high_resolution_clock::now();

    cout << "row-major: " << duration_cast<milliseconds>(t1 - t0).count() << " ms\n";
    cout << "col-major: " << duration_cast<milliseconds>(t2 - t1).count() << " ms\n";
    cout << (s1 == s2 ? "same sum" : "different sum") << '\n';
    return 0;
}
```

```text
row-major: 3 ms
col-major: 23 ms
same sum
```

Identical arithmetic, identical answer, **7x** difference — purely from access order. On larger grids and with `long long` the ratio is typically 2–10x. The rule: **make the innermost loop index the last array dimension.**

| Access pattern | Cache lines per element | Effect |
|---|---|---|
| `g[i][j]` in `for i { for j }` | ~1/16 of a line | sequential prefetch, near-optimal |
| `g[i][j]` in `for j { for i }` | 1 full line | 16x waste on a 4-byte type |
| `vector<vector<int>>` random rows | 2 dependent loads | pointer chase, worse than flat |
| `list<int>` traversal | 1 line per element + unpredictable next | ~5-20x slower than `vector` |

Two corollaries worth memorising:

- **`vector` beats `list` almost always.** Contiguous memory means prefetching and fewer misses; `list` pays a pointer indirection per element and cannot be prefetched. A `vector` insertion in the middle costs $\mathcal{O}(n)$ memmove, but memmove runs at ~10 GB/s, so for $n \le 10^5$ it beats a linked list walk.
- **Flat 2D beats ragged 2D.** `vector<int> g(n*m)` with `g[i*m+j]` performs one allocation and has perfect locality; `vector<vector<int>>` has $n$ separate blocks scattered across the heap.

False sharing — two threads writing to different variables on the same cache line — is out of scope for now (contests are single-threaded by rule), but the underlying principle is the same one: *what you touch together should live together.*

---

## 14. The segfault debugging playbook

A segfault means the process accessed memory it does not own. In contest code there are six causes, and you should walk them in order — most bugs are found in the first two.

| # | Cause | Symptom | How to confirm | Fix |
|---|---|---|---|---|
| 1 | Out-of-bounds array/vector access | crash on large tests, fine on samples | print indices; compile with `-D_GLIBCXX_DEBUG -fsanitize=address` | widen the array, check loop bounds, use `.at()` while debugging |
| 2 | Null pointer dereference | crashes immediately, address `0x0` | print the pointer before dereferencing | guard with `if (p)`, or use indices |
| 3 | Stack overflow via deep recursion | crashes after a delay, no message, often on a path-shaped test | print depth; run with `ulimit -s` raised | convert to an explicit stack, or raise depth budget |
| 4 | Use after free | intermittent, depends on allocator | `-fsanitize=address` reports it precisely | use `vector`/indices; never keep pointers into a grown container |
| 5 | Uninitialised pointer | crash or silent corruption, varies per run | initialise everything to `nullptr` and assert | always `int* p = nullptr;` |
| 6 | Missing `&` in `scanf` | crash on the very first input | look at the format string vs arguments | add `&` — see §15 |

The single most effective tool is a local rebuild with sanitizers:

```bash
g++ -std=c++17 -O1 -g -fsanitize=address,undefined -D_GLIBCXX_DEBUG sol.cpp -o sol
```

`-fsanitize=address` catches out-of-bounds, use-after-free, double-free and leaks with a stack trace pointing at the exact line. `-D_GLIBCXX_DEBUG` turns STL out-of-bounds into an assertion with a message instead of undefined behaviour. Compile with these *before* you submit, not after. (For the full warning and sanitizer setup, see [Topic 02](Topic%2002%20-%20Understanding%20Comments,%20Errors,%20Warnings.md).)

A cheap always-on habit: put `assert` on your invariants. Asserts are compiled out under `-DNDEBUG`, so they cost nothing on the judge and everything when you need them.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 5;
    vector<int> a = {3, 1, 4, 1, 5};
    assert((int)a.size() == n);          // catches a logic error immediately
    int idx = 7;
    assert(0 <= idx && idx < n);         // fails here, with a line number, instead of segfaulting later
    return 0;
}
```

```text
Assertion `0 <= idx && idx < n' failed.
```

An assertion failure names the condition and the line. A segfault names nothing.

---

## 15. `&` in `scanf`, and why forgetting it crashes

`scanf` cannot pass by reference — it is a C function taking a variadic list of *addresses*. The format specifier `%d` tells it "read an int and **store it at the address I give you**". Give it a value instead of an address and it writes 4 bytes to a random location.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x;
    scanf("%d", x);        // WRONG: x's value is used as an address
    printf("%d\n", x);
    return 0;
}
```

`x` is uninitialised — say it happens to hold `6422304`. `scanf` writes the parsed integer to address `6422304`, which belongs to something else in your process (or to nothing). Two outcomes:

- **Crash**: address not mapped → segfault before you ever reach `printf`.
- **Silent corruption**: address happens to be writable → some unrelated variable, a return address, or the allocator's metadata is now garbage. You get a wrong answer, or a crash ten lines later in a function that did nothing wrong.

The correct form is one character longer:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x;
    long long y;
    double z;
    char s[21];

    scanf("%d %lld %lf %20s", &x, &y, &z, s);   // & for ints/doubles, NOT for arrays

    printf("%d %lld %.2f %s\n", x, y, z, s);
    return 0;
}
```

```text
7 1000000000000 3.50 hello
```

Note the asymmetry that confuses everyone: **`&` for `int`, `long long`, `double`, `char`; no `&` for `char s[]`, because `s` is already an address.** If you ever write `scanf("%d", &s)` on a `char` array, you are writing 4 bytes over the array's *pointer slot*, not into the array.

Since C++17 you can avoid the whole class of bug with `cin >> x`, which is type-safe and takes the variable directly. With fast I/O it is also comparable in speed — see [Topic 03](Topic%2003%20-%20Input,%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20%28VERY%20IMPORTANT%29.md). *A missing `&` is the single most common cause of "my solution crashes on test 1" in a beginner's first month; it is also the easiest to spot once you know the shape.*

---

## 16. `reinterpret_cast` and C-style casts: almost always a bug

`reinterpret_cast<T*>(p)` reinterprets the bits. It performs no conversion and no check. It is legal C++ and, in a contest, it is nearly always a mistake.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    double d = 3.14;

    // (1) "Reading the bits of a double" — strict-aliasing UB
    long long bits = *reinterpret_cast<long long*>(&d);
    cout << hex << bits << '\n';         // looks fine on g++, is UB by the standard

    // (2) The safe, standard-blessed version
    long long safe;
    static_assert(sizeof(safe) == sizeof(d));
    memcpy(&safe, &d, sizeof(d));
    cout << hex << safe << '\n';         // identical output, no UB

    // (3) The classic disaster: treating an int array as float
    int a[4] = {1, 2, 3, 4};
    float* f = reinterpret_cast<float*>(a);
    cout << f[0] << '\n';                 // some denormal near 1.4e-45, not 1.0
    return 0;
}
```

Why it is a bug more often than not:

| Attempt | What actually happens | Correct tool |
|---|---|---|
| `(int*)&someFloat` to "convert" | reinterprets the bit pattern as an integer address | `static_cast<int>(f)` |
| `(long long*)&arr` to read two ints | strides 8 bytes, reads past the array; endian-dependent | index twice: `a[i] + (1LL<<32)*a[i+1]` |
| `(int*)malloc(...)` in C++ | hides a missing `#include`, no constructors | `new[]` or `vector` |
| casting away `const` to write | modifies read-only memory → segfault, or UB if it was a literal | fix the design |

`static_cast` converts values. `reinterpret_cast` relabels memory. Use the first when you mean "turn this number into another type", the second never — and if you truly need to inspect an object's bytes, use `memcpy` or `std::bit_cast` (C++20).

---

## Key Takeaways

- Five segments: text (code), data (initialised globals), bss (zero globals), heap (manual), stack (automatic). Locals go on the stack; globals and statics go in data/bss and are zeroed for free.
- The stack is 1 MB (Windows) to 8 MB (Linux). `int a[1000000]` is 4 MB — always declare big arrays at global scope or use `vector`.
- Recursion depth beyond ~$2\times10^5$ frames risks stack overflow; a judge will not honour your `ulimit` or linker flag.
- `new[]` must pair with `delete[]`. Mismatch is UB and typically corrupts the allocator silently.
- Competitive programmers use `vector`, not `new`. Use a global array pool instead of `new Node` for linked structures.
- `nullptr` beats `NULL` beats `0`. `int* a, b;` declares one pointer and one `int` — declare one variable per line.
- `p + 1` advances `sizeof(T)` bytes. `a[i] == *(a+i)`; `sizeof(array)` inside a function is 8, not the array's size.
- `a + n` (one past the end) is legal to form, illegal to dereference — this is the `[begin, end)` contract.
- References cannot be null or re-seated. Use `const T&` for read-only big objects, `T&` for output parameters, `T` for small types.
- Read `const` declarations right-to-left: `const int*` protects the value, `int* const` protects the pointer.
- Every pointer is 8 bytes on 64-bit; the *stride* differs by type, which is what breaks casts.
- Before submitting, compute $N \times \text{sizeof}(T)$ against 90% of the limit. `long long dp[5000][5000]` is 190 MiB.
- Row-major iteration is 2–10x faster than column-major. Make the innermost loop the last dimension; prefer `vector` over `list`.
- Segfault checklist: out-of-bounds, null deref, deep recursion, use-after-free, uninitialised pointer, missing `&` in `scanf`.
- `reinterpret_cast` relabels memory rather than converting values; use `static_cast`, or `memcpy`/`bit_cast` for bit inspection.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Big local array `int a[1e6]` | 4 MB on an 8 MB stack, plus other frames → segfault | move to global scope, or `vector<int> a(n)` |
| 2 | Declaring `int a[1e5]` inside a recursive function | every frame carries 400 KB → overflow at depth 20 | make it global, or pass it down |
| 3 | `delete` instead of `delete[]` | allocator metadata corrupted, crash later | always match `new[]`/`delete[]`; better, avoid both |
| 4 | Keeping `int* p = &v[0]` across `push_back` | reallocation frees the old buffer | store the index, not the pointer |
| 5 | `scanf("%d", x)` | writes to a garbage address, segfault or corruption | `scanf("%d", &x)` |
| 6 | `scanf("%d", &s)` on `char s[100]` | overwrites the array's address slot with an int | `scanf("%s", s)` |
| 7 | `int* a, b;` expecting two pointers | `b` is a plain `int` | one declaration per line |
| 8 | `sizeof(a)` inside `void f(int a[])` | arrays decay to pointers; gives 8 | pass `n` explicitly |
| 9 | `for (int i = 0; i <= n; i++) a[i]` | reads one past the end | use `< n`; remember `[begin, end)` |
| 10 | `const int* p` then `*p = 5` | compile error you do not understand | read right-to-left; use `int* const` for a fixed pointer |
| 11 | Allocating `long long dp[5000][5000]` twice | 190 MiB each → MLE | use a rolling array of two rows |
| 12 | Iterating a 2D grid column-major | 5-10x cache misses → TLE on large $N$ | swap the loop order |
| 13 | Returning `&local` | dangling pointer, works by luck then fails on the judge | return by value, or return the object |
| 14 | `reinterpret_cast<int*>(&d)` to "convert" | type punning UB, wrong value | `static_cast<int>(d)` |
| 15 | Assuming the stack limit is the same everywhere | Windows 1 MB vs Linux 8 MB | test with a global; never trust "it ran locally" |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Declare a `10^7`-element `int` array and fill it; make it work on a 1 MB stack | local exercise | Easy |
| 2 | Rewrite a recursive DFS of depth `10^6` using an explicit `vector` stack | CSES "Building Roads" style | Easy–Medium |
| 3 | Read $n$ integers with `scanf` into a `vector`, deliberately omit one `&`, observe the crash | local exercise | Easy |
| 4 | Compute both row-major and column-major sums of a `2000x2000` matrix and time them | local exercise | Easy |
| 5 | Given $N \le 5000$, write a DP with `long long dp[N][N]` and report its exact MiB before running | any DP introductory problem | Medium |
| 6 | Reduce an $\mathcal{O}(n^2)$ memory DP to two rows and verify identical output | CSES "Grid Paths" / "Book Shop" style | Medium |
| 7 | Implement a queue of `10^6` nodes with an array pool instead of `new` | Codeforces Div. 2 C, BFS problems | Medium |
| 8 | Write `swap(int&, int&)`, `swap(int*, int*)` and a `const int&`-taking printer; explain each signature | local exercise | Easy |
| 9 | Sort a `vector<pair<int,int>>` by a runtime-chosen function pointer (asc/desc) | Codeforces Div. 2 B | Easy |
| 10 | Use `-fsanitize=address` to find a deliberate off-by-one in a prefix-sum loop | local exercise | Easy |
| 11 | Convert a `vector<vector<int>>` grid to a flat `vector<int>` and measure the speedup | AtCoder ABC C/D | Medium |
| 12 | Compute max $N$ for a `long long` DP under 64 MB and 256 MB, then verify by allocating | local exercise | Easy |

## Next Up

→ [Topic 11 — Functions, Classes (Basics) and Useful Lambda Methods](Topic%2011%20-%20Functions,%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md)