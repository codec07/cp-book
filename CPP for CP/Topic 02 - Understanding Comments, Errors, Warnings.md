# Comments, Errors, Warnings, and Debugging Without an IDE

You will spend more contest time reading compiler output than writing code. A judge gives you one line — `Compilation error on test 1` — and the rest is on you. The difference between a 1400-rated and a 1900-rated coder is often not algorithms: it is that the 1900-rated coder reads `expected ';' before '}' token`, looks at line 47, and fixes it in eight seconds. This chapter turns red text into a diagnosis, and turns silent wrong answers into loud, located crashes.

> **Prerequisites:** [Topic 01 — C++ Basics and Syntax](Topic%2001%20-%20CPP%20Basics%20and%20Syntax%20Introduction.md) (you can compile and run a file).
> **You will be able to:**
> - Classify any diagnostic as preprocessor, syntax, semantic, or linker — and know which one your fix belongs to.
> - Read a compiler message bottom-up, including template instantiation backtraces.
> - Recognise the eight compile errors that cause 90% of CP rejections.
> - Use `assert`, `cerr`, `#ifdef LOCAL`, sanitizers and `-D_GLIBCXX_DEBUG` to find bugs the samples hide.
> - Build a stress-test harness that finds the counterexample for you.
> - Decide, with reasons, whether `#define int long long` belongs in your template.

---

## 1. Comments: `//` and `/* */`

```cpp
// Line comment: everything to the end of this line is ignored.

/*
   Block comment.
   Spans lines. Does NOT nest.
*/

int x = 10;   // trailing comment — the most useful kind in contest code
```

The rule that bites: **block comments do not nest.** The moment you "comment out" a block that already contains a `/* ... */`, the inner `*/` closes the outer comment and the rest of your file becomes code again.

```cpp
/*
   int f() {
       /* this inner close ends the comment early */
       return 1;
   }
*/
```

That produces something like `error: 'return' outside function` far below the comment — a genuinely confusing diagnostic. Debugging habit: comment out with `//` on each line, or wrap in `#if 0 ... #endif`, which *does* nest.

```cpp
#if 0
    // anything here is skipped, and it can contain /* comments */ freely
#endif
```

### 1.1 When to comment in contest code

| Situation | Comment? | Why |
|---|---|---|
| A non-obvious invariant (`// a[1..i] is sorted`) | **Yes** | You will re-read this at minute 100 and not remember |
| Why a loop bound is `<=` and not `<` | **Yes** | Off-by-one bugs are re-introduced by "fixing" them |
| A magic constant (`1e18`, `998244353`) | **Yes** | `// LINF: 4e18 < LLONG_MAX` |
| Mask/bitset meanings | **Yes** | `// bit j = item j taken` |
| `// read input`, `// loop`, `// print answer` | No | Pure noise; it teaches nothing |
| Restating the problem statement | No | The statement is open in another tab |
| Commented-out old code | No | Delete it. It compiles to nothing but it *reads* as noise under time pressure |

The governing principle: comment the **why**, never the **what**. `i < n` does not need a comment; `i < n - 1` often does.

### 1.2 The comment that saves a submission

```cpp
// If the answer can be "impossible", we print -1.
// Edge case: n == 1 -> answer is 0, not -1.  (cost me a WA on 2024-xx-xx)
```

That last parenthetical is the highest-value comment style in competitive programming. When you lose a submission to an edge case, write the edge case into your template notes.

---

## 2. The four classes of diagnostics

Compilation is a pipeline (Chapter 01, §6). Each stage produces a *different species* of error, and the fix lives in a different place.

| Class | Emitted by | Signature in the message | Typical cause |
|---|---|---|---|
| **Preprocessor** | `cpp` | `fatal error:`, `#include`, `#if`, `#define` | Missing header, unclosed `#if`, bad macro invocation |
| **Syntax** | `cc1plus` (parser) | `error: expected ... before ... token` | Missing `;`, `)`, `}`, `>` |
| **Semantic / type** | `cc1plus` (semantic analysis) | `error: 'x' was not declared`, `no matching function`, `invalid conversion`, `no member named` | Wrong types, wrong names, wrong arity |
| **Linker** | `ld` | `undefined reference to`, `multiple definition of`, `collect2: error: ld returned 1` | Declared but not defined, or defined twice |

Learn to spot the class in half a second. `fatal error:` → look at your includes. `expected` → look at the line *above* the one named. `undefined reference` → the code compiled, so your types are fine and your *build command* is wrong.

---

## 3. Preprocessor errors

### 3.1 Missing header

```cpp
#include <bits/stdc++.h>   // fine on GCC, fatal on MSVC / macOS clang

int main() { return 0; }
```

```text
main.cpp:1:10: fatal error: bits/stdc++.h: No such file or directory
    1 | #include <bits/stdc++.h>
      |          ^~~~~~~~~~~~~~~
compilation terminated.
```

Reading it: the caret `^~~~~` underlines the *offending token*, and `compilation terminated` means nothing was compiled at all — every later error would be noise, so GCC stops.

Fix: on MSVC or macOS clang, replace with explicit headers (`<iostream>`, `<vector>`, `<algorithm>`, …).

### 3.2 Broken `#if`

```cpp
#include <iostream>
#ifdef LOCAL
int main() {
    std::cout << "debug\n";
// no #endif
```

```text
main.cpp:6: error: unterminated #ifdef
    6 | int main() {
      | 
main.cpp:4: note: #ifdef began here
```

The `note:` points at where the block *started*. Always read the `note:` lines — they carry the actual location.

### 3.3 Macro arity

```cpp
#include <iostream>
#define MAX3(a, b, c) std::max(a, std::max(b, c))

int main() {
    std::cout << MAX3(1, 2) << '\n';   // only two arguments
    return 0;
}
```

```text
main.cpp:5:26: error: macro "MAX3" requires 3 arguments, but only 2 given
    5 |     std::cout << MAX3(1, 2) << '\n';
      |                          ^
```

Note the wording: **macro**, not *function*. That single word tells you the error is at stage 1.

---

## 4. Syntax errors

### 4.1 The missing semicolon

```cpp
#include <iostream>
using namespace std;

int main() {
    int n = 5
    cout << n << '\n';
    return 0;
}
```

```text
main.cpp: In function 'int main()':
main.cpp:6:5: error: expected ';' before 'cout'
    6 |     cout << n << '\n';
      |     ^~~~
```

This is the single most common compile error in the world. Read the message as: **"the statement before this point is not finished"**. The `;` goes at the end of line 5, not line 6. GCC names the token it choked on, which is one token *after* the real problem.

```text
main.cpp:6:5: error: expected ';' before '}' token
```
means the *last statement inside the block* is missing its semicolon.

### 4.2 Unbalanced braces

```cpp
int main() {
    if (true) {
        return 0;
    // missing }
```

```text
main.cpp:6:1: error: expected '}' at end of input
```

`at end of input` → you are missing a closing brace somewhere. Your editor's bracket matcher will find it faster than reading.

### 4.3 Angle-bracket soup (pre-C++11 templates, still caused by typos)

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<vector<int> v;    // forgot the closing >
    return 0;
}
```

```text
main.cpp:4:22: error: '>>' should be '> >' within a nested template argument list
```
This specific message is C++03-era. In C++11 and later `>>` is legal, so the modern message is more like:

```text
main.cpp:4:24: error: template argument 1 is invalid
main.cpp:4:24: error: template argument 2 is invalid
```

A cascade of "template argument is invalid" usually means one missing `>`.

---

## 5. Semantic and type errors

### 5.1 Undeclared name

```cpp
#include <iostream>
using namespace std;

int main() {
    int n = 5;
    cout << m << '\n';
    return 0;
}
```

```text
main.cpp: In function 'int main()':
main.cpp:6:13: error: 'm' was not declared in this scope
    6 |     cout << m << '\n';
      |             ^
```

If a similar name exists, modern GCC appends a suggestion:

```text
main.cpp:6:13: error: 'n' was not declared in this scope; did you mean 'm'?
```

Do not trust the suggestion blindly — it is edit distance, not semantics.

### 5.2 Wrong types in a template call

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << max(1, 2LL) << '\n';    // int vs long long
    return 0;
}
```

```text
main.cpp:5:13: error: no matching function for call to 'max(int, long long int)'
    5 |     cout << max(1, 2LL) << '\n';
      |             ^~~
In file included from ...
/usr/include/c++/13/bits/stl_algobase.h:259:5: note: candidate: 'template<class _Tp> constexpr const _Tp& std::max(const _Tp&, const _Tp&)'
  259 |     max(const _Tp& __a, const _Tp& __b)
      |     ^~~
/usr/include/c++/13/bits/stl_algobase.h:259:5: note:   template argument deduction/substitution failed:
main.cpp:5:13: note:   deduced conflicting types for parameter 'const _Tp' ('int' and 'long long int')
```

The block is long, but only one line matters:

```text
note:   deduced conflicting types for parameter 'const _Tp' ('int' and 'long long int')
```

`std::max` takes **one** template parameter, so both arguments must have the *same* type. This error is CP-relevant far beyond `max`:

```cpp
long long a = 5;
int b = 3;
cout << min(a, b);        // error: conflicting types
```

Fix: make the types match (`min(a, (long long)b)` or `min<long long>(a, b)`), or use `min(a, 1LL*b)`. Same story for `std::swap`, `std::sort` comparators returning inconsistent types, and `accumulate`'s initial value.

### 5.3 Pointer / integer confusion

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a[5] = {1, 2, 3, 4, 5};
    int x = a;          // array decays to int*
    cout << x << '\n';
    return 0;
}
```

```text
main.cpp:6:13: error: invalid conversion from 'int*' to 'int' [-fpermissive]
    6 |     int x = a;
      |             ^
```

`[-fpermissive]` means "this used to be a warning in old GCC; it is now an error". If you see `-fpermissive`, you are doing something C-era sloppy — fix the code, do not add the flag.

### 5.4 Redefinition

```cpp
#include <iostream>
using namespace std;

int x = 1;
int x = 2;      // duplicate definition at namespace scope

int main() { return 0; }
```

```text
main.cpp:5:5: error: redefinition of 'int x'
    5 | int x = 2;
      |     ^
main.cpp:4:5: note: 'int x' previously declared here
    4 | int x = 1;
      |     ^
```

At *function* scope, `int x = 2;` after `int x = 1;` is legal (shadowing). At *namespace* scope it is `redefinition`. This is exactly why pasting two solutions together fails.

### 5.5 A member that does not exist

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> v = {3, 1, 2};
    sort(v.begin(), v.end());
    cout << v.top() << '\n';       // top() is a stack/priority_queue method
    return 0;
}
```

```text
main.cpp:6:15: error: 'class std::vector<int>' has no member named 'top'; did you mean 'pop_back'?
    6 |     cout << v.top() << '\n';
      |              ^~~
```

`has no member named X` → you are holding the wrong container, or you forgot a `#include`. If you forget `<set>` but kept `<bits/stdc++.h>` out, the error becomes `'set' was not declared in this scope` instead.

---

## 6. Linker errors

These come *after* successful compilation. Your syntax and types are fine; the assembler produced object files with unresolved names.

### 6.1 Declared but never defined

```cpp
// sol.cpp
#include <iostream>
int solve(int n);          // declaration only

int main() {
    std::cout << solve(5) << '\n';
    return 0;
}
```

```text
/usr/bin/ld: /tmp/cc8Kk3Zq.o: in function `main':
sol.cpp:(.text+0x2a): undefined reference to `solve(int)'
collect2: error: ld returned 1 exit status
```

Reading it: `undefined reference to 'solve(int)'` = "somebody promised this function exists; nobody provided a body." Note the object file name `/tmp/cc8Kk3Zq.o` — that is the *temporary* object GCC created from `sol.cpp`. The `(.text+0x2a)` is an address inside `main`, i.e. the call site.

### 6.2 Two `main` functions

```text
/usr/bin/ld: /tmp/ccB1.o: in function `main':
b.cpp:(.text+0x0): multiple definition of `main'; /tmp/ccA1.o:a.cpp:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
```

`multiple definition of 'main'` = you compiled two files that each have a `main`. This is the signature of pasting a friend's solution next to yours, or of `g++ *.cpp` in a folder full of old solutions.

### 6.3 Header defined in two translation units

```cpp
// util.h
int add(int a, int b) { return a + b; }   // definition in a header — dangerous
```

Included by `a.cpp` and `b.cpp`, then linked together:

```text
/usr/bin/ld: /tmp/ccB1.o: in function `add(int, int)':
b.cpp:(.text+0x0): multiple definition of `add(int, int)'; /tmp/ccA1.o:a.cpp:(.text+0x0): first defined here
```

Fixes: `inline int add(...)`, or `static`, or move the body into one `.cpp`. In contests you rarely see this — but you *do* see it the moment you use two files.

| Linker message | Meaning | Fix |
|---|---|---|
| `undefined reference to 'f(int)'` | Declared, never defined | Write the body, or add the `.cpp` to the command |
| `undefined reference to 'std::thread::...'` | Missing library | Add `-pthread` |
| `multiple definition of 'main'` | Two entry points | Compile one file, or rename one |
| `multiple definition of 'f(int)'` | Header defined non-`inline` | `inline` / `static` / move to `.cpp` |
| `undefined reference to 'WinMain'` | MinGW console/subsystem mismatch | `-mconsole`, or your `main` signature is wrong |

---

## 7. Reading a compiler message bottom-up

A modern GCC diagnostic has up to five parts:

```text
main.cpp:6:13: error: 'm' was not declared in this scope
    6 |     cout << m << '\n';
      |             ^
```

1. **`file:line:column`** — `main.cpp`, line 6, column 13. Column is 1-based and points at the *first character* of the offending token.
2. **`error:` / `warning:` / `note:`** — severity. `note:` is the most under-read part of the output and usually contains the answer.
3. **The message** — often phrased as "expected X before Y".
4. **The source line, re-printed** — handy when the terminal cannot show the file.
5. **The caret line** — `^` for a point, `^~~~` for a span, `~~~~^` when the problem precedes the token.

### 7.1 Why bottom-up

When GCC recovers from an error it often emits a cascade. Example: you forget `#include <vector>` and reference `vector` 12 times.

```text
main.cpp:7:5: error: 'vector' was not declared in this scope
main.cpp:7:5: note: suggested alternative: 'iterator'
main.cpp:8:5: error: 'vector' was not declared in this scope
...
main.cpp:14:1: error: expected '}' at end of input
```

**Only the first error is trustworthy.** Each later one may be a consequence. So: fix the first error, recompile, repeat. Never fix twelve errors at once.

The exception: when a *linker* error appears after a wall of template notes, read the **last** `note:` in the template instantiation chain — it names your type:

```text
/usr/include/c++/13/bits/stl_algo.h: In instantiation of 'void std::sort(_RAIter, _RAIter, _Compare) [with ...]':
main.cpp:9:14:   required from here
/usr/include/c++/13/bits/predefined_ops.h:98:23: error: no match for 'operator<' (operand types are 'Node' and 'Node')
```

The useful fact is at the very bottom: **you called `sort` on a `vector<Node>` without defining `operator<`**. Everything above is the path that led there.

### 7.2 `-fmax-errors`

```bash
g++ -O2 -std=c++17 -fmax-errors=3 sol.cpp -o sol
```

Caps the cascade at three errors. In a contest, use `-fmax-errors=1` and iterate; you will read less and fix faster.

---

## 8. The eight compile errors that cause 90% of CP rejections

| Message (abridged) | Class | Real cause | Fix |
|---|---|---|---|
| `expected ';' before ...` | Syntax | Missing semicolon on the *previous* line | Add `;` at end of previous statement |
| `'x' was not declared in this scope` | Semantic | Typo, or declared inside another block | Fix name; check brace scope |
| `'vector' was not declared` / `'sort' was not declared` | Semantic | Missing `#include` | `<vector>`, `<algorithm>`, or `<bits/stdc++.h>` |
| `no matching function for call to 'max(int, long long int)'` | Semantic | Mismatched types in a template | Cast both to `ll`, or `max<long long>(a, b)` |
| `invalid conversion from 'int*' to 'int'` | Semantic | Used an array name where an element was meant | `a[i]`, or `*a` |
| `redefinition of 'int x'` | Semantic | Two globals with one name (often a paste) | Rename or delete |
| `expected '}' at end of input` | Syntax | Unclosed brace / `#if` | Match braces |
| `undefined reference to 'f(int)'` | **Linker** | Declared, not defined; wrong files on the command line | Define it, or add the source file |

Bonus, because it is so common in CP:

```cpp
int main() {
    int n; cin >> n;
    int a[n];              // VLA — GCC accepts it, it is not standard C++
    ...
}
```

Use `vector<int> a(n);`. Variable-length arrays are a GCC extension; they work on Codeforces and blow the stack for large `n` (see §10.5).

---

## 9. Warnings that predict wrong answers

A warning is a bug you have not triggered *yet*. These five are worth reading every time.

### 9.1 `-Wuninitialized`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    int n, best;
    cin >> n;
    for (int i = 0; i < n; i++) {
        int x; cin >> x;
        if (x > best) best = x;   // best never initialised
    }
    cout << best << '\n';
    return 0;
}
```

```text
main.cpp: In function 'int main()':
main.cpp:4:12: warning: 'best' is used uninitialized [-Wuninitialized]
    4 |     int n, best;
      |            ^~~~
```

On the judge this is not a warning — it is a random answer. On your machine it may accidentally be 0 because the stack happened to be clean; on test 7 it is `32610`. **The most dangerous bug class in CP**, because it passes the samples.

### 9.2 `-Wsign-compare`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> v = {1, 2, 3};
    for (int i = 0; i < (int)v.size(); i++) cout << v[i] << ' ';
    // without the cast:
    for (int i = 0; i < v.size(); i++) cout << v[i] << ' ';
    return 0;
}
```

```text
main.cpp:7:23: warning: comparison of integer expressions of different signedness: 'int' and 'std::vector<int>::size_type' {aka 'long unsigned int'} [-Wsign-compare]
    7 |     for (int i = 0; i < v.size(); i++) cout << v[i] << ' ';
      |                       ^
```

`size()` is unsigned. `-1 < v.size()` is **false**, because `-1` converts to a huge unsigned value. Where it actually kills you:

```cpp
for (int i = (int)v.size() - 1; i >= 0; i--) { ... }   // signed: fine
for (size_t i = v.size() - 1; i >= 0; i--) { ... }     // unsigned: infinite loop
```

The second loop wraps to `SIZE_MAX` and segfaults. Habit: cast `size()` to `int`, or use `(int)v.size()` in every bound.

### 9.3 `-Wreturn-type`

```cpp
#include <bits/stdc++.h>
using namespace std;

int f(int x) {
    if (x > 0) return 1;
    // no return on the else path
}

int main() { cout << f(-1) << '\n'; return 0; }
```

```text
main.cpp: In function 'int f(int)':
main.cpp:7:1: warning: control reaches end of non-void function [-Wreturn-type]
    7 | }
      | ^
```

Undefined behaviour, usually a garbage value, occasionally a crash. Recursive functions with `if (...) return dfs(...)` and no final `return` are the classic CP instance.

### 9.4 `-Wshadow`

```cpp
#include <bits/stdc++.h>
using namespace std;
int n;
int main() {
    cin >> n;
    if (n > 0) {
        int n = 5;          // shadows the global
        cout << n << '\n';  // prints 5, not the input
    }
    return 0;
}
```

```text
main.cpp:7:13: warning: declaration of 'n' shadows a global declaration [-Wshadow]
    7 |         int n = 5;          // shadows the global
      |             ^
main.cpp:3:5: note: shadowed declaration is here
    3 | int n;
      |     ^
```

Very common when you add `int x` inside a lambda that already captures `x`, or reuse `i` in a nested loop. `-Wshadow` costs nothing and finds real bugs.

### 9.5 `-Winfinite-recursion` (GCC 10+)

```text
main.cpp:8:5: warning: infinite recursion detected [-Winfinite-recursion]
```

Almost always a DFS that forgot its visited check — the kind of bug that MLEs or stack-overflows on the judge.

---

## 10. Runtime errors

Here the compiler was happy. The program died.

| Signal / exception | Typical cause | First thing to check |
|---|---|---|
| `SIGSEGV` (139) | Out-of-bounds index, null pointer, stack overflow | Array sizes, `v[n]` vs `v[n-1]`, recursion depth |
| `SIGFPE` (136) | Integer division by zero, `% 0`, `INT_MIN / -1` | Divisors, `n % i` loops with `i` starting at 0 |
| `SIGABRT` (134) | `assert` failed, `std::bad_alloc` uncaught, `-D_GLIBCXX_DEBUG` check | The assertion's message / `what()` |
| `std::bad_alloc` | Requested more memory than the limit, or a negative size cast to `size_t` | `vector<int> v(-1)` → giant size |
| `std::out_of_range` | `.at()` on a vector/string/map | Index just past the end |

### 10.1 `SIGSEGV` from one bad index

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    int n = 10;
    vector<int> a(n);
    a[n] = 5;        // one past the end — silent corruption, no check
    cout << a[0] << '\n';
    return 0;
}
```

`operator[]` does not check. Sometimes it writes into adjacent heap memory and the program "works"; sometimes it segfaults on `n = 1000000` because you left the allocated region. **This is why `-D_GLIBCXX_DEBUG` exists** (§12).

### 10.2 `SIGFPE` is not always division by zero

```cpp
int x = INT_MIN;
int y = -1;
cout << x / y;     // SIGFPE: the result 2147483648 does not fit in int
```

On x86 this raises `SIGFPE`. Same for `INT_MIN % -1`.

### 10.3 `bad_alloc` from a negative size

```cpp
int n = -1;
vector<int> v(n);                        // length_error / bad_alloc
size_t bad = (size_t)n;                  // 18446744073709551615
vector<int> w(bad);                      // std::bad_alloc, or length_error
```

The `(size_t)` cast of a negative number is a 64-bit monster. It appears when you compute `n - k` with `k > n` and then use it as a size.

### 10.4 Recursion depth

Default stack is 8 MB on Linux. Each `dfs` frame with a few `int`s is ~48–80 bytes, so depth ~100 000 is safe, ~1 000 000 overflows into `SIGSEGV`. A DFS on a path graph with $n = 10^5$ nodes is fine; $n = 10^6$ is not. Either raise the limit (`ulimit -s unlimited`, not available on judges) or write it iteratively.

### 10.5 The VLA stack bomb

```cpp
int n = 1000000;
int a[n];          // ~4 MB on the stack — may survive
int b[n][n];       // instant segfault / stack overflow
```

Static or global arrays live in BSS and can be huge; local ones live on the 8 MB stack. `vector` allocates on the heap. In CP, size-large arrays go **outside** `main`.

---

## 11. Debugging without an IDE

You get a terminal, a compiler, and `stdin`. That is enough.

### 11.1 `assert`: free correctness checks

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    assert(n >= 0 && n <= 100000);      // input contract

    vector<int> a(n);
    for (int &x : a) cin >> x;

    long long sum = 0;
    for (int x : a) { sum += x; assert(sum >= 0); }   // invariant

    cout << sum << '\n';
    return 0;
}
```

An `assert` that fires prints the expression, the file and the line, then aborts:

```text
sol: sol.cpp:14: int main(): Assertion `sum >= 0' failed.
Aborted (core dumped)
```

That is a *located* failure — far better than a WA. Two rules: `assert` is removed when `NDEBUG` is defined (`-DNDEBUG`), so never put side effects inside one (`assert(++i < n)` is a bug); and never submit with asserts that can fire on valid input, because the judge reports it as RE.

### 11.2 `cerr` debug prints

`cerr` is unbuffered and goes to stderr, which judges do not compare against the expected output. So debug prints are *safe* — but slow.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n; cin >> n;
    vector<int> a(n);
    for (int &x : a) cin >> x;

    for (int i = 0; i < n; i++) {
        cerr << "i=" << i << " a[i]=" << a[i] << '\n';   // stderr, not compared
    }
    return 0;
}
```

### 11.3 `#ifdef LOCAL` and `__LINE__`

The professional pattern: debug code that **does not exist** in the submitted binary.

```cpp
#include <bits/stdc++.h>
using namespace std;

#ifdef LOCAL
#define dbg(...) cerr << "[" << #__VA_ARGS__ << "] = "; _dbg(__VA_ARGS__)
template <class T> void _dbg(const T &x) { cerr << x << '\n'; }
template <class T, class... U> void _dbg(const T &x, const U &...y) {
    cerr << x << " | "; _dbg(y...);
}
#else
#define dbg(...) ((void)0)
#endif

int main() {
    int n = 3, m = 4;
    dbg(n);                 // LOCAL: [n] = 3
    dbg(n, m);              // LOCAL: [n, m] = 3 | 4
    dbg(__LINE__, __func__);
    return 0;
}
```

Compile locally with `-DLOCAL`; submit without it and every `dbg(...)` expands to `((void)0)`, which costs zero instructions.

`__LINE__` expands to the current line number, `__func__` to the function name, `__FILE__` to the filename. Together they make a one-line trace:

```cpp
#define trace() cerr << __FILE__ << ":" << __LINE__ << " in " << __func__ << '\n'
```

### 11.4 `gdb` in 60 seconds

```bash
g++ -O0 -g -std=c++17 sol.cpp -o sol
gdb ./sol
```

| Command | Does |
|---|---|
| `run < in.txt` | Start with input redirected |
| `bt` | Backtrace — the call stack at the crash |
| `print i`, `print a[5]` | Inspect a value |
| `frame 2` | Switch to stack frame 2 |
| `break sol.cpp:42` | Breakpoint at a line |
| `next`, `step`, `continue` | Line-by-line, into functions, resume |

`bt` at a `SIGSEGV` names the exact line. That is usually the whole investigation.

---

## 12. `-D_GLIBCXX_DEBUG`

This flag turns libstdc++'s containers into checked ones: out-of-bounds `operator[]`, invalidated iterators, `lower_bound` on an unsorted range, comparing iterators from different containers.

```bash
g++ -O2 -std=c++17 -D_GLIBCXX_DEBUG -g sol.cpp -o sol_dbg
```

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> v = {1, 2, 3};
    cout << v[3] << '\n';     // UB normally; with _GLIBCXX_DEBUG:
    return 0;
}
```

```text
/usr/include/c++/13/debug/vector:524:
In function:
    std::debug::vector<_Tp, _Alloc>::reference std::debug::vector<_Tp,
    _Alloc>::operator[](size_type) [with _Tp = int; _Alloc =
    std::allocator<int>; reference = int&; size_type = long unsigned int]

Error: attempt to subscript container with out-of-bounds index 3, but
container only has size 3
```

It also catches the iterator-invalidation bug:

```cpp
for (auto it = s.begin(); it != s.end(); ) {
    if (*it % 2 == 0) s.erase(it);   // erase invalidates it; then ++it is UB
    else ++it;
}
```

With `_GLIBCXX_DEBUG` you get "attempt to increment an invalid iterator" instead of a mystery WA three submissions later. **Never submit a `_GLIBCXX_DEBUG` build**: it is 5–20× slower and the memory layout differs.

---

## 13. `-fsanitize=address,undefined`

AddressSanitizer finds memory errors; UndefinedBehaviorSanitizer finds signed overflow, bad shifts, misaligned access, and null dereferences — including the ones that "work" on your machine.

```bash
g++ -O1 -g -std=c++17 -fsanitize=address,undefined -fno-omit-frame-pointer sol.cpp -o sol_san
```

Use `-O1` with sanitizers (they need some optimisation to be accurate, but `-O2` can reorder away the thing you are hunting).

### 13.1 Catching signed overflow

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    int a = 2000000000;
    int b = a + a;          // signed overflow: UB
    cout << b << '\n';
    return 0;
}
```

```text
sol.cpp:5:13: runtime error: signed integer overflow: 2000000000 + 2000000000 cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior sol.cpp:5:13
```

That message names the exact line and the exact operation — the single most useful tool for the "why is my answer negative" bug.

### 13.2 Catching stack-buffer overflow

```text
==12345==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ffd...
WRITE of size 4 at 0x7ffd... thread T0
    #0 0x4011a2 in main sol.cpp:7
```

### 13.3 What sanitizers cost

| Build | Relative speed | Use for |
|---|---|---|
| `-O2` | 1.0× | Submitting |
| `-O2 -D_GLIBCXX_DEBUG` | 5–20× slower | Iterator/bounds bugs |
| `-O1 -fsanitize=address,undefined` | 2–3× slower, 3× memory | Memory and overflow bugs |
| `-O2 -fsanitize=address,undefined` | 1.5–2× slower | When `-O1` changes the bug away |

Sanitizers are exactly why the "it works on my machine" submission dies: your machine's UB happens to produce the right answer; the judge's does not.

---

## 14. Stress testing: let the computer find the counterexample

You cannot hand-craft the test that breaks your solution. You *can* write a slow, obviously correct brute force and let a loop search for you.

Three files:

**`brute.cpp`** — correct by construction, no cleverness.

```cpp
#include <bits/stdc++.h>
using namespace std;

// Task: given n and an array, count pairs (i < j) with a[i] + a[j] == 0.
int main() {
    int n;
    cin >> n;
    vector<int> a(n);
    for (int &x : a) cin >> x;
    long long ans = 0;
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if (a[i] + a[j] == 0) ans++;
    cout << ans << '\n';
    return 0;
}
```

**`fast.cpp`** — your real solution.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    vector<int> a(n);
    for (int &x : a) cin >> x;
    unordered_map<int, int> cnt;
    long long ans = 0;
    for (int x : a) {
        ans += cnt[-x];
        cnt[x]++;
    }
    cout << ans << '\n';
    return 0;
}
```

**`gen.cpp`** — random tests, seeded from `argv[1]` so any failure is reproducible.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main(int argc, char** argv) {
    int seed = (argc > 1) ? atoi(argv[1]) : 1;
    mt19937 rng(seed);
    int n = rng() % 8 + 1;              // small n: brute force stays fast
    cout << n << '\n';
    for (int i = 0; i < n; i++) {
        int v = (int)(rng() % 11) - 5;  // values in [-5, 5]: collisions are likely
        cout << v << (i + 1 == n ? '\n' : ' ');
    }
    return 0;
}
```

### 14.1 The compare script (bash / Linux / macOS / WSL / git-bash)

```bash
#!/usr/bin/env bash
# stress.sh — run with: bash stress.sh
set -u
g++ -O2 -std=c++17 -o brute brute.cpp || exit 1
g++ -O2 -std=c++17 -o fast  fast.cpp  || exit 1
g++ -O2 -std=c++17 -o gen   gen.cpp   || exit 1

for i in $(seq 1 2000); do
  ./gen "$i" > in.txt
  ./brute < in.txt > out_brute.txt
  ./fast  < in.txt > out_fast.txt

  if ! cmp -s out_brute.txt out_fast.txt; then
    echo "MISMATCH at seed $i"
    echo "--- input ---";        cat in.txt
    echo "--- expected ---";     cat out_brute.txt
    echo "--- your answer ---";  cat out_fast.txt
    exit 1
  fi

  if (( i % 200 == 0 )); then echo "ok: $i tests"; fi
done
echo "All 2000 tests passed"
```

### 14.2 The compare script (PowerShell / Windows)

```powershell
# stress.ps1 — run with: powershell -File stress.ps1
g++ -O2 -std=c++17 -o brute.exe brute.cpp
g++ -O2 -std=c++17 -o fast.exe  fast.cpp
g++ -O2 -std=c++17 -o gen.exe   gen.cpp

foreach ($i in 1..2000) {
    .\gen.exe $i            | Out-File -Encoding ascii in.txt
    Get-Content in.txt      | .\brute.exe | Out-File -Encoding ascii out_brute.txt
    Get-Content in.txt      | .\fast.exe  | Out-File -Encoding ascii out_fast.txt

    $a = (Get-Content out_brute.txt) -join "`n"
    $b = (Get-Content out_fast.txt)  -join "`n"
    if ($a -ne $b) {
        Write-Host "MISMATCH at seed $i"
        Get-Content in.txt
        Write-Host "expected: $a"
        Write-Host "got     : $b"
        exit 1
    }
    if ($i % 200 -eq 0) { Write-Host "ok: $i tests" }
}
Write-Host "All 2000 tests passed"
```

### 14.3 Designing a generator that actually finds bugs

| Bug you suspect | Generator should produce |
|---|---|
| Off-by-one | `n = 1`, `n = 2` frequently — use `rng() % 8 + 1`, not `% 100000 + 1` |
| Overflow | values near `10^9`, and `11` (for `11 * 10^9`) — mix extremes with small values |
| Ties / duplicates | small value range so collisions happen constantly |
| Wrong on equal elements | arrays with many identical entries |
| Negative-number handling | include negatives |
| Empty/degenerate structure | `n = 0` if the constraints allow |

**The number-one stress-testing mistake is a weak generator.** If the range is $10^6$ and you pick 8 random values, nothing interesting ever happens. Pick tiny ranges so duplicates, zeros and negatives are common.

> *A contest story: an $\mathcal{O}(n \log n)$ solution "correct by inspection" failed stress test seed 4 — `n = 1`, an empty map, and `ans += cnt[-x]` before `cnt[x]++` on a single zero. Ten seconds with a generator beats forty minutes of staring.*

---

## 15. The `#define int long long` debate

The trick: put `#define int long long` at the top and every `int` in your code becomes 64-bit.

```cpp
#include <bits/stdc++.h>
using namespace std;

#define int long long

int32_t main() {          // main must be int32_t — "int" is now long long
    int n = 3;
    int a = 2000000000;
    cout << a + a << '\n';
    return 0;
}
```

Output: `4000000000`. Without the macro you would print `-294967296`.

### 15.1 The pros

- Overflow bugs — the most common WA in CP — mostly disappear.
- You stop writing `1LL *` everywhere.
- `accumulate`, prefix sums, and products of two $10^9$ values become safe by default.

### 15.2 The cons, in order of how much they hurt

1. **`main` must be `int32_t main()`.** `int main()` becomes `long long main()`, which the standard forbids and GCC rejects.
2. **`INT_MAX` is still 32-bit.** `INT_MAX` is the macro `2147483647`; it does not change. So this is now broken:

```cpp
#define int long long

const int INF = INT_MAX;    // 2147483647, not "infinity" for a 64-bit int
```

3. **`printf("%d", x)` is undefined behaviour** — `x` is `long long`, `%d` reads 32 bits. You must write `%lld` everywhere, which is exactly the type-tracking the macro was supposed to save you from.
4. **Memory doubles.** `int dp[5000][5000]` goes from 100 MB to 200 MB — MLE on a 256 MB limit.
5. **It is slower.** 64-bit arithmetic is wider, caches hold half as many elements, and vectorised loops narrow. Typically 10–40% on memory-bound code.
6. **It hides real design errors.** A sum that needs `long long` is a fact about the problem; a macro hides the fact instead of teaching it.
7. **`std::abs`, `std::min`, `std::max` overload resolution changes**, and `unsigned`/`long long` comparisons get more warnings, not fewer.
8. **`memset(dp, 0x3f, sizeof dp)` still "works"** but `0x3f3f3f3f` is now a small number compared to what `int` can hold, so `INF` sentinels silently stop being infinite.

### 15.3 The verdict

| Your level | Recommendation |
|---|---|
| Learning (first 6 months) | **Do not use it.** Learn where `long long` is needed; that skill is examined by every problem |
| Div. 3/4 speed rounds | Optional — it saves a class of WA at the cost of MLE risk |
| Div. 1/2, memory-tight problems | **Do not use it.** You need control of memory and `printf` |
| Your own library (`template.cpp`) | Use `using ll = long long;` and declare `ll` where needed |

The professional compromise: keep `using ll = long long;`, write `ll` in the five places that need it (sums, products, distances, counts, `1LL *`), and never think about the macro again.

---

## 16. Undefined behaviour hall of fame

UB means the standard permits *anything* — including "works on your machine, WA on the judge".

| # | UB | Example | Symptom |
|---|---|---|---|
| 1 | Signed integer overflow | `int x = 2e9; x += x;` | Negative answers, or a compiler that assumes it never happens and deletes your check |
| 2 | Uninitialised read | `int best; if (x > best) ...` | Random WA on test 7 only |
| 3 | Out-of-bounds access | `a[n]`, `v[-1]`, `s[s.size()+1]` | Corruption, segfault, or "correct" output |
| 4 | Modifying a container while iterating | `erase(it)` then `++it` | Skipped elements, crash |
| 5 | Shift by ≥ width | `1 << 32`, `x << -1` | Zero or garbage; use `1LL << 32` |
| 6 | `abs(INT_MIN)` | `abs(INT_MIN)` is UB (no positive counterpart) | Negative "absolute value" |
| 7 | Division by zero / `INT_MIN / -1` | `x / 0` | SIGFPE |
| 8 | Dangling reference | returning a reference to a local, capturing a local by reference in a lambda that outlives it | Garbage after the function returns |
| 9 | `memset` on non-trivial types | `memset(&v, 0, sizeof v)` on a `vector` | Crash on destruction |
| 10 | Reading a `double` as `int` via `printf("%d")` | `printf("%d", 1.5)` | Garbage |
| 11 | `vector<bool>` reference | `auto &b = v[0];` on `vector<bool>` | Compile error, or a proxy that dies when the vector grows |

Two of these deserve a sentence more.

**#4, iterator invalidation**, because it is *the* container bug:

```cpp
// WRONG: erase() invalidates the iterator you then increment
for (auto it = s.begin(); it != s.end(); ++it)
    if (*it % 2 == 0) s.erase(it);

// RIGHT (C++11+): erase returns the next valid iterator
for (auto it = s.begin(); it != s.end(); )
    if (*it % 2 == 0) it = s.erase(it);
    else ++it;

// ALSO RIGHT for vectors: erase-remove
v.erase(remove_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; }), v.end());
```

**#1, signed overflow**, because the optimiser *uses* it. With `-O2` this can print `1`:

```cpp
int f(int x) { return x + 1 > x; }    // "always true" — overflow is UB, so the compiler assumes it cannot happen
```

Wrapping arithmetic on unsigned types is well-defined; on signed types it is not. `-fwrapv` makes it defined, but the correct fix is to use the right type.

---

## Key Takeaways

- Classify the diagnostic first: `fatal error` → preprocessor, `expected` → syntax, `'x' was not declared` / `no matching function` → semantics, `undefined reference` → linker.
- Fix the **first** error only, recompile, repeat; later errors are often cascade artefacts. Use `-fmax-errors=1` to enforce the habit.
- `note:` lines hold the answer — especially `deduced conflicting types` (the `max(int, long long)` bug) and `previously declared here`.
- `expected ';' before X` means the *previous* statement is unfinished; `at end of input` means an unclosed brace or `#if`.
- Warnings are pre-bugs: `-Wuninitialized`, `-Wsign-compare`, `-Wreturn-type`, `-Wshadow` each catch a real WA class for free.
- `v.size()` is `size_t`; a backward loop with `size_t i >= 0` is an infinite loop.
- `assert` + `cerr` + `#ifdef LOCAL` gives located failures without an IDE; `dbg()` compiles to nothing when submitted.
- `-D_GLIBCXX_DEBUG` catches out-of-bounds `[]` and invalidated iterators; `-fsanitize=address,undefined` catches signed overflow and memory abuse. Both are local-only, both cost 2–20× speed.
- Stress testing is mechanical: brute force + seeded generator + compare loop. The generator must use *small ranges* or it finds nothing.
- `#define int long long` trades overflow bugs for `INT_MAX` bugs, `%d` UB, double memory and a 10–40% slowdown. Learn `long long` properly instead.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---------|--------------|-----|
| 1 | Fixing ten errors at once | Nine were cascade artefacts; you break working code | Fix the first error, recompile |
| 2 | Ignoring `note:` lines | The template backtrace names your type in the last `note:` | Read the message bottom-up |
| 3 | `/* */` around code that already has a block comment | The inner `*/` closes early; errors appear far below | `//` each line, or `#if 0 ... #endif` |
| 4 | `assert` with side effects | `-DNDEBUG` deletes the whole expression, including `++i` | Keep asserts pure |
| 5 | Submitting a `-D_GLIBCXX_DEBUG` or `-fsanitize` build | 2–20× slower, different memory layout → TLE/MLE | Separate `dbg` and `fast` build commands |
| 6 | Debugging a WA by staring | You cannot see the failing test | Stress test with a brute force + generator |
| 7 | Generator with huge ranges | Nothing interesting is generated; stress test passes and the judge fails you | Use `rng() % 8 + 1` and small value ranges |
| 8 | `#define int long long` with `int main()` | `long long main()` is ill-formed | `int32_t main()` |
| 9 | `memset(dp, 0x3f, sizeof dp)` with 64-bit ints | `0x3f3f3f3f` is no longer "near infinity" | `fill` with an explicit `INF` |
| 10 | Catching `SIGSEGV` as "the judge is broken" | It is always your index, your pointer, or your stack depth | `-D_GLIBCXX_DEBUG` + `bt` in gdb |
| 11 | Backwards loop with `size_t` | `i >= 0` is always true for unsigned | Cast to `int`, or loop forward |
| 12 | Trusting "it works on my machine" | UB has no required behaviour; the judge's layout differs | Sanitizers locally, every time before submitting |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|--------------|----------------|------------|
| 1 | Deliberately produce a preprocessor, syntax, semantic and linker error; name each | Local experiment | Easy |
| 2 | Break `max(1, 2LL)` and read the full `note:` chain | Local experiment | Easy |
| 3 | Two-file program with a function declared in one and never defined | Local experiment | Easy |
| 4 | Write a bug that overflows `int`; find it with `-fsanitize=undefined` | Local experiment | Easy |
| 5 | Write a solution with an off-by-one; let `-D_GLIBCXX_DEBUG` locate it | Local experiment | Easy–Medium |
| 6 | Build a stress-test harness for any array problem and run 2000 seeds | Local experiment | Medium |
| 7 | Take any solved problem and write an `assert`-based invariant checker | Any judge | Medium |
| 8 | Same problem, two builds: `-O2` and `-O2 -fsanitize=address,undefined`; compare times and verdicts | Any judge | Medium |
| 9 | Find a Div. 2 A/B where `#define int long long` causes MLE | Codeforces Div. 2 | Medium |

---

## Next Up

→ [Topic 03 — Input, Output and Efficient I/O specially for CP](Topic%2003%20-%20Input%2C%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20%28VERY%20IMPORTANT%29.md)