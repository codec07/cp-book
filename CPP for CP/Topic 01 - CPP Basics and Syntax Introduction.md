# C++ Basics and Syntax — The Contestant's Starting Line

Competitive programming is a sport with a compiler. You get a problem, a time limit measured in seconds, and a program that must be *correct on the first or second try*. C++ is the language of that sport because it lets you control the constant factor — the thing that decides whether $\mathcal{O}(n \log n)$ finishes in 0.4 s or 2.1 s. This chapter builds the machinery you will use in every single round: the file skeleton, the headers, the compile command, and the template you paste before you have even read problem A.

> **Prerequisites:** None — this is the starting point. You need a working `g++` and a text editor, nothing else.
> **You will be able to:**
> - Explain what a judge actually does with your file, and what AC / WA / TLE / MLE / RE mean in terms of your code.
> - Write, compile and run a C++17 program from the command line with the exact flags a CP coder uses.
> - Justify every line of the starter template you will paste into every contest for the next two years.
> - Compile several files into one program and recognise when an error is the compiler's fault versus the linker's.

---

## 1. What competitive programming actually is

A competitive programming problem is a *function specification*. It gives you:

1. **Input format** — exactly what bytes arrive on `stdin`.
2. **Output format** — exactly what bytes must leave on `stdout`.
3. **Constraints** — $1 \le n \le 2\cdot 10^5$, $1 \le a_i \le 10^9$, and so on. The constraints are the single most important line of the statement: they tell you which complexity class is allowed.
4. **Time and memory limit** — typically 1 s or 2 s and 256 MB.

Your program is run on a set of hidden test files. If it produces the right bytes on all of them within the limits, you get **AC**. Otherwise you get a verdict that is a diagnosis: wrong answer, too slow, too much memory, or crashed.

### 1.1 The landscape

| Platform | Format | Time limit culture | What it is good for |
|---|---|---|---|
| **ICPC** (regionals, then World Finals) | 3-person team, 5 hours, 8–13 problems, one shared computer | 1–5 s, generous | Learning to communicate and to prioritise; the "real" team sport |
| **Codeforces** | Individual, 2–2.25 h, 5–7 problems, rated Div. 1–4 | 1–2 s, tight | Rating grind, editorial culture, hacking phase |
| **CSES Problem Set** | 300 fixed problems, no timer, no rating | 1 s, strict | Deliberate practice; each problem teaches one technique |
| **AtCoder** (ABC / ARC / AGC) | Individual, 100 min, ABC has A–G | 2 s, very consistent | Clean statements, strong tests, good for beginners |
| **CodeChef** | Individual, long + short contests | 1–2 s | Indian ecosystem, ICC tie-ins |
| **LeetCode** | Function-completion, no stdin/stdout | 1–2 s per case | Interview prep — *not* the same skill as CP |

The first thing to notice: **CSES and AtCoder have 1-second limits and strong tests**. That is deliberate. A solution with a sloppy constant factor passes on Codeforces and dies on CSES. Practise on the strict ones.

### 1.2 The 1-second reality

A modern judge machine executes roughly $10^8$ *simple* operations per second in C++ with `-O2`. "Simple" means an integer add, a compare, an array index. A `std::map` lookup is not simple — it is 20–40 of them. This gives you the single most useful table in competitive programming:

| Constraint | Allowed complexity | Typical technique |
|---|---|---|
| $n \le 10$ | $\mathcal{O}(n!)$, $\mathcal{O}(2^n n)$ | Brute force, permutations, bitmask DP |
| $n \le 20$ | $\mathcal{O}(2^n n)$, $\mathcal{O}(3^n)$ | Subset enumeration, meet-in-the-middle |
| $n \le 500$ | $\mathcal{O}(n^3)$ | Floyd–Warshall, interval DP |
| $n \le 5000$ | $\mathcal{O}(n^2)$ | Simple DP, all pairs |
| $n \le 2\cdot 10^5$ | $\mathcal{O}(n \log n)$ | Sort, binary search, segment tree, Dijkstra |
| $n \le 10^6$ | $\mathcal{O}(n)$ or $\mathcal{O}(n \log n)$ *with care* | Sieve, two pointers, prefix sums |
| $n \le 10^9$ | $\mathcal{O}(\log n)$, $\mathcal{O}(\sqrt n)$ | Binary search on answer, matrix power, digit DP |
| $n$ up to $10^{18}$ | $\mathcal{O}(1)$ | Formula / closed form |

Read the constraints *before* you read the rest of the statement. The constraints usually name the intended solution.

> *A contest story: in a Codeforces Div. 2 round, an $\mathcal{O}(n^2)$ solution with $n \le 2\cdot 10^5$ needs about $4\cdot 10^{10}$ operations. On a 2-second limit that is roughly 200 seconds of CPU — except you will not see "200 seconds". You will see **TLE on test 3**, and you will spend twenty minutes thinking your logic is wrong instead of your complexity.*

### 1.3 The verdicts, decoded

| Verdict | Meaning | What it usually points to in your code |
|---|---|---|
| **AC** | Accepted | Correct and fast enough. Check the time — 1.9 s / 2.0 s means a fragile solution |
| **WA** | Wrong answer | Wrong logic, uninitialised variable, off-by-one, overflow, missing `long long` |
| **TLE** | Time limit exceeded | Wrong complexity, or right complexity with a huge constant (`cin` without sync-off, `map` where an array would do) |
| **MLE** | Memory limit exceeded | `int dp[10000][10000][10]`, storing all $10^6$ strings, recursion depth of $10^6$ |
| **RE** | Runtime error | Out-of-bounds index, division by zero, null dereference, stack overflow from deep recursion, `assert` firing |
| **CE** | Compilation error | Syntax, typo, wrong header, `main` missing, two `main`s |
| **PE** | Presentation error | Trailing space or missing newline — *rare now, most judges treat it as WA* |
| **ILE** | Idleness limit exceeded | Interactive problem: your program printed nothing when the judge expected a query |
| **Skipped** | Not run | You failed a pretest earlier, or you submitted after the round ended |

Note the asymmetry: **WA costs you a penalty but teaches you nothing specific**, while **RE and CE give you the line number**. When you are stuck, deliberately re-run your code locally with sanitizers (Chapter 02) to convert a silent WA into a loud RE with a line number.

---

## 2. The anatomy of a C++ program

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, judge\n";
    return 0;
}
```

```text
Hello, judge
```

Four things are happening, and each one has a CP-specific consequence:

| Piece | What it does | Why a CP coder cares |
|---|---|---|
| `#include <iostream>` | Textually pastes a header into your file at *preprocessing* time | Wrong/missing header = CE. `<bits/stdc++.h>` pastes ~all of them |
| `int main()` | The entry point the C runtime calls | Must return `int`. `void main()` is rejected by every real judge |
| `std::cout << ...` | Writes into a buffer, not to the screen | Buffering is why `\n` vs `endl` matters (Chapter 03) |
| `return 0;` | Exit status | Non-zero exit status is often reported as RE even if output is correct |

Compile and run it the way you always will:

```bash
g++ -O2 -std=c++17 -Wall -o sol sol.cpp
./sol
```

On Windows with MinGW the last line is `sol.exe`, and the build line is the same.

---

## 3. `#include`, headers, and `<bits/stdc++.h>`

`#include` is not a smart import. It is a **literal text insertion** performed by the preprocessor before the compiler sees anything. `#include <iostream>` copies the entire contents of the file `iostream` into your source at that point. That is why include order occasionally matters, and why a 3-line program can become 800 000 lines after preprocessing.

### 3.1 The headers that actually matter

| Header | Gives you | You need it for |
|---|---|---|
| `<iostream>` | `cin`, `cout`, `cerr`, `clog` | Any I/O |
| `<cstdio>` | `scanf`, `printf`, `fopen`, `getchar` | Fast I/O, `freopen` |
| `<algorithm>` | `sort`, `lower_bound`, `max`, `min`, `next_permutation` | 90% of contests |
| `<vector>`, `<array>`, `<deque>` | Sequence containers | Everything |
| `<set>`, `<map>`, `<unordered_map>` | Ordered / hashed containers | Counting, ordering |
| `<string>` | `std::string` | Any text problem |
| `<numeric>` | `accumulate`, `gcd`, `iota` | Sums, gcd |
| `<climits>` | `INT_MAX`, `LLONG_MAX` | Sentinel values |
| `<cstring>` | `memset`, `strlen` | Bulk initialisation |
| `<cmath>` | `sqrt`, `pow`, `abs` | Geometry |
| `<iomanip>` | `setprecision`, `fixed` | Printing doubles |
| `<bits/stdc++.h>` | *all of the above* | Speed of typing |

### 3.2 The `bits/stdc++.h` debate

```cpp
#include <bits/stdc++.h>
using namespace std;
```

This is a GCC implementation detail — a header that pulls in essentially the whole standard library. Facts:

- **It is not standard C++.** It exists in GCC (and therefore on Codeforces, AtCoder, CSES, CodeChef, most judges). It does **not** exist in MSVC, and on macOS's clang you must install GCC's libstdc++ to get it.
- **It costs compile time.** Roughly 0.8–1.5 s versus 0.2 s for `<iostream>` alone. Irrelevant in a contest, annoying in a 500-file project.
- **It hides missing includes.** You will write `sort` for months and never learn it lives in `<algorithm>`. Then you paste code into a LeetCode-style environment with explicit headers and it does not compile.

**CP rule:** use `<bits/stdc++.h>` in contests, use explicit headers in anything you ship or share. When you write your own library file that other files `#include`, always use explicit headers — otherwise you leak the whole standard library into every translation unit.

---

## 4. `main`, return codes, and what the judge does with them

```cpp
int main() {
    // ...
    return 0;
}
```

Three rules, each with a bite:

1. **`main` must return `int`.** `void main()` is a Microsoft extension; GCC rejects it (or warns and does undefined things). If a tutorial shows `void main`, close it.
2. **Reaching the end of `main` returns 0 implicitly** — this is the *only* function in C++ where falling off the end is legal. Falling off the end of any other non-`void` function is undefined behaviour, and `-Wall` will call it out (`-Wreturn-type`).
3. **The exit status is checked.** A judge that sees exit code 1 (or a signal) reports **RE**, even if your output was perfect. `assert` failures abort with `SIGABRT` — that is exactly why `assert` is a debugging tool and not a submission tool.

`argc`/`argv` exist but you will essentially never use them in a contest: input arrives on `stdin`, not as command-line parameters.

```cpp
int main(int argc, char** argv) {
    // argv[1] is the first argument — useful for your own stress-test scripts,
    // almost never for a judge.
    return 0;
}
```

---

## 5. Namespaces: `std::` and the `using namespace std;` question

Everything in the standard library lives in namespace `std`. Without help, you must qualify: `std::cout`, `std::vector<int>`, `std::sort`. Three ways to avoid the typing:

```cpp
// 1. import one name — surgical, safe
using std::cout;
using std::vector;

// 2. import one namespace — what CP does
using namespace std;

// 3. fully qualify — verbose, unambiguous
std::cout << std::vector<int>{1, 2, 3}.size() << '\n';
```

### 5.1 Why CP uses `using namespace std;`

Because you type `vector` 200 times per contest and you have 40 minutes. There is no downside in a single-file, 200-line program: you control every name in the file, so a collision is your own doing and you notice it immediately at compile time.

### 5.2 Why production code does not

A header with `using namespace std;` at global scope poisons every file that includes it. The classic disaster: a project defines its own `count`, `distance`, `left`, `right`, `rank`, `data`, or `swap`, and after an include the call becomes ambiguous or silently resolves to `std::count`. `std::distance` and `std::count` are real names that have broken real codebases.

**The CP-specific version of this bug** is worth knowing because you *will* hit it:

```cpp
#include <bits/stdc++.h>
using namespace std;

int count = 0;          // a global called count

int main() {
    vector<int> v = {1, 2, 3};
    // cout << count(v.begin(), v.end(), 1);  // error: 'count' is not a function
    return 0;
}
```

The global `int count` hides `std::count`. The fix is either `std::count(...)` explicitly or, better, name your variable `cnt` — which is why every competitive programmer on earth writes `cnt`.

### 5.3 The `std::` names that collide most often

| Your variable | Collides with | Symptom |
|---|---|---|
| `count` | `std::count` | "is not a function" |
| `left`, `right` | `std::left`, `std::right` (stream manipulators) | confusing overload errors |
| `data` | `std::data` (C++17) | ambiguity |
| `swap` | `std::swap` | ambiguity / ADL chaos |
| `rank` | `std::rank` (type trait) | ambiguity |
| `distance` | `std::distance` | ambiguity |
| `plus`, `minus`, `equal_to` | function objects in `<functional>` | ambiguity |

**Habit:** prefix your globals. `cnt`, `n`, `m`, `res`, `ans`, `ok`, `lo`, `hi`. Two-to-four letter names, no collisions, and they read fine.

---

## 6. The compilation pipeline

One `g++` command runs four programs in sequence. Knowing them turns a wall of red text into a located bug.

| Stage | Tool | Input | Output | Command to stop here |
|---|---|---|---|---|
| 1. Preprocess | `cpp` | `.cpp` | `.ii` (pure C++, no `#` lines) | `g++ -E sol.cpp -o sol.ii` |
| 2. Compile | `cc1plus` | `.ii` | `.s` (assembly) | `g++ -S sol.cpp -o sol.s` |
| 3. Assemble | `as` | `.s` | `.o` (machine code, unresolved symbols) | `g++ -c sol.cpp -o sol.o` |
| 4. Link | `ld` (via `collect2`) | `.o` + libraries | executable | `g++ sol.o -o sol` |
| 5. Load & run | OS loader | executable | output | `./sol` |

### 6.1 Why you should care: three different kinds of failure

```bash
g++ -E sol.cpp | tail -5      # what the preprocessor produced
g++ -c sol.cpp                # compile only — catches syntax/type errors
g++ sol.o -o sol              # link only — catches missing/duplicate definitions
```

- An error at **stage 1** is a preprocessor error: bad `#include`, unterminated `#if`, macro abuse.
- An error at **stage 2** is a compile error: syntax, types, undeclared names.
- An error at **stage 4** is a **linker** error: `undefined reference to 'f(int)'` (you declared but never defined it) or `multiple definition of 'main'` (two files define it).

The judge reports stages 1, 2 and 4 all as **CE**, but the *fix* is completely different, so you must be able to tell them apart. Chapter 02 dissects real messages from each.

### 6.2 Multi-file programs

```bash
g++ -O2 -std=c++17 -Wall main.cpp solver.cpp -o sol
```

Each `.cpp` compiles independently to a `.o`. A function declared in a header and defined in exactly one `.cpp` links fine; define it in two and the linker says `multiple definition`. This is why the header guard / `inline` / `static` rules exist — and why pasting two contestants' code together (the infamous "Frankenstein submission") usually produces a link error about `main`.

---

## 7. The `g++` flags a CP coder actually uses

```bash
# local development — warnings on, sanitizers on, optimised like the judge
g++ -O2 -std=c++17 -Wall -Wextra -Wshadow \
    -fsanitize=address,undefined -D_GLIBCXX_DEBUG -g \
    sol.cpp -o sol
```

| Flag | Effect | Use it when |
|---|---|---|
| `-O2` | Standard optimisation. What nearly every judge uses | **Always** |
| `-O3` | More aggressive: vectorisation, more inlining | Sometimes 1.2–1.5× faster; can expose UB. Match the judge — if it uses `-O2`, test with `-O2` |
| `-std=c++17` | C++17 language + library | Always (Codeforces compiles with `gnu++17`) |
| `-std=c++20` | C++20: `ranges`, concepts, `<bit>` (`std::popcount`) | If the judge supports it; check the compiler version in the problem's language list |
| `-Wall` | The useful warnings | Always |
| `-Wextra` | More warnings, some noisy | Local only |
| `-Wshadow` | Warn when an inner variable hides an outer one | Local — catches the classic `int i` inside a loop that shadows a global |
| `-g` | Keep debug symbols (needed by sanitizers and `gdb`) | Local only |
| `-fsanitize=address,undefined` | Runtime checks for OOB, use-after-free, signed overflow, UB | **Local only** — 2–10× slowdown, and it changes memory layout |
| `-D_GLIBCXX_DEBUG` | Checks STL iterator validity and `operator[]` bounds | Local only — catches "iterator invalidated by `erase`" |
| `-DONLINE_JUDGE` | Defines a macro; convention for disabling debug output | Optional |
| `-static` | Statically link libstdc++ | Rare; some judges need it |
| `-Ofast` | `-O3` + fast-math: **breaks IEEE floating point** | Almost never — it makes `double` comparisons lie |
| `-pthread` | Enable threads | Only for `std::thread` experiments |

Two traps worth memorising:

- **Never judge speed with `-O0`.** `-O0` can be 3–10× slower than `-O2`. A solution that "TLEs locally" may be fine on the judge, and vice versa. Compile with `-O2` when you time anything.
- **Sanitizers change behaviour.** `-D_GLIBCXX_DEBUG` makes `vector::operator[]` bounds-checked, so an out-of-bounds read that silently returns garbage under the judge becomes a clean abort locally. That is the point — but never submit the sanitizer build.

> *A contest story: the local `-Wall -Wshadow` habit catches roughly one bug per round for free. The warning `declaration of 'i' shadows a previous local` has saved more submissions than any editorial.*

---

## 8. The template: what you paste before reading problem A

Every serious contestant has a file called `template.cpp` on the desktop. The point is not cleverness — it is **zero decisions at minute zero**. You open the file, save-as the problem name, and start thinking about the problem instead of about `ios_base::sync_with_stdio`.

Here is the modern template, small enough to remember, complete enough to never touch again:

```cpp
#include <bits/stdc++.h>
using namespace std;

using ll = long long;
using pii = pair<int, int>;
using vi = vector<int>;

#define all(x) begin(x), end(x)
#define sz(x) (int)(x).size()
#define pb push_back

const int INF = 1e9;
const ll LINF = 4e18;

void solve() {
    int n;
    cin >> n;
    vi a(n);
    for (int &x : a) cin >> x;

    ll sum = accumulate(a.begin(), a.end(), 0LL);
    cout << sum << '\n';
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int t = 1;
    cin >> t;                 // remove this line if the problem has one test case
    while (t--) solve();

    return 0;
}
```

Why each piece exists, in the order you will need it:

| Line | Reason |
|---|---|
| `#include <bits/stdc++.h>` | One line replaces twelve headers; saves ~15 s per problem |
| `using namespace std;` | `vector` not `std::vector`; see §5 for the caveat |
| `using ll = long long;` | You will type `long long` 50 times; `ll` is 3 characters and cannot overflow differently |
| `using pii = pair<int,int>;` | `priority_queue<pii, vector<pii>, greater<pii>>` becomes readable |
| `#define all(x)` | `sort(all(v))` instead of `sort(v.begin(), v.end())` |
| `#define sz(x)` | `(int)` cast avoids the signed/unsigned warning `-Wsign-compare` |
| `#define pb push_back` | Pure typing speed |
| `const int INF = 1e9;` | A sentinel larger than any input but small enough that `INF + INF` does not overflow `int` |
| `const ll LINF = 4e18;` | For `long long` Dijkstra distances; `4e18 < 9.22e18 = LLONG_MAX` |
| `void solve()` | Keeps `main` clean; makes the multi-test loop one line |
| `sync_with_stdio(false); cin.tie(nullptr);` | 2–5× faster input (Chapter 03) |
| `int t; cin >> t;` | Reads the test count; **the number-one template bug is forgetting to delete it** |
| `return 0;` | Exit status 0 = success |

### 8.1 The multi-test-case trap in detail

Half of all Codeforces problems have a first line `t` = number of test cases. Half do not. If you leave `cin >> t` in and the input is a single test case, `cin >> t` consumes the value of $n$, `t` becomes $n$, and your program runs `solve()` $n$ times on garbage. You get WA on test 1 with an obviously correct solution.

If you delete `cin >> t` and the input *does* have `t`, the first test case reads $t$ as $n$. Same symptom.

**Discipline:** read the input format section, say out loud "there is / is not a $t$", then comment or uncomment the one line. Every submission. No exceptions.

### 8.2 A variant for interactive problems

Interactive problems should not use `sync_with_stdio(false)` — output must be flushed after every query (Chapter 03 explains why in detail). Keep a second template:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    for (int i = 0; i < n; i++) {
        cout << "? " << i << endl;   // endl flushes — required here
        int r;
        cin >> r;
    }
    cout << "! done" << endl;
    return 0;
}
```

---

## 9. Macros, `typedef`, and `using`

### 9.1 `typedef` vs `using`

```cpp
typedef long long ll;          // C style — still valid
typedef vector<int> vi;
using ll2 = long long;         // C++11 style — preferred
using vi2 = vector<int>;
```

They are identical for simple types. `using` wins for two reasons: it reads left-to-right (name = type), and it is the only form that works for alias templates:

```cpp
template <class T>
using vv = vector<vector<T>>;      // vv<int> is vector<vector<int>>

// typedef cannot do this:
// template <class T> typedef vector<vector<T>> vv;   // error
```

In a contest you will use `using ll = long long;` and maybe `using pii = pair<int,int>;`. That is the whole job.

### 9.2 Object-like macros

```cpp
#define MOD 1000000007
#define MAXN 200005
```

Textual substitution. `MOD` is not a variable: it has no address, no type, and the debugger cannot show it. Prefer:

```cpp
const int MOD = 1'000'000'007;
const int MAXN = 200005;
```

`const int` is type-checked, scoped, visible in the debugger, and optimises to the same code. The one legitimate macro use is `#define int long long` (Chapter 02) — and even that is a debate.

### 9.3 Function-like macros and the parenthesisation disaster

```cpp
#define sq(x) x * x
int y = sq(1 + 2);        // expands to 1 + 2 * 1 + 2  =  5, not 9
```

The "fix" people write is `#define sq(x) ((x) * (x))`, which still breaks on `sq(i++)` (double increment) and on types with side effects. The correct answer in C++ is a function or a lambda:

```cpp
inline int sq(int x) { return x * x; }
// or, generic and zero-cost:
auto sq = [](auto x) { return x * x; };
```

A real one from contest folklore:

```cpp
#define left(x) (2 * (x))
// ... and later:  auto it = s.lower_bound(v);  // lower_bound internally calls left()?
```

More realistically, `#define left` / `#define right` / `#define next` / `#define prev` collide with member functions on `std::map` and `std::set` and with `std::next`:

```cpp
#include <bits/stdc++.h>
using namespace std;
#define next(x) ((x) + 1)

int main() {
    auto it = next(1);      // "works"
    // but:  next(it) on an iterator is now a syntax error
    // and:  std::next(it) is a preprocessor disaster
    return 0;
}
```

### 9.4 Why you should not over-macro

| Macro habit | What actually happens |
|---|---|
| `#define int long long` | `INT_MAX` becomes wrong; `main` must be `int32_t main()`; memory doubles; sometimes 30% slower |
| `#define left` / `right` / `next` | Silently breaks `std::map` members and `std::next` |
| `#define rep(i,a,b) for(int i=a;i<b;i++)` | Debugger cannot step into the loop condition sensibly; shadowing warnings multiply |
| `#define endl '\n'` | Breaks interactive problems (no flush) — the bug is invisible until you get ILE |
| `#define scanf(...)` wrappers | Argument-evaluation-order surprises |
| `#define pb push_back` | Harmless — this one is fine |

The rule: **macros that only shorten typing are fine; macros that change semantics are not.** `#define pb push_back` changes nothing. `#define int long long` changes types, overflow behaviour, `printf` format specifiers, and the signature of `main`.

---

## 10. The contest loop: read, template, submit

A 2-hour Codeforces round looks like this:

| Minute | Action |
|---|---|
| 0–3 | Open all problems. Read A and B fully. Note which has the larger solve count |
| 3–5 | Paste template into `a.cpp`. Confirm "is there a $t$?" |
| 5–15 | Solve A. Submit. **Do not polish.** |
| 15–20 | Solve B. Submit |
| 20–60 | C. If stuck at 40 min, switch to D for 10 minutes to see if it is easier than it looks |
| 60–90 | The hard problem, or re-check A/B for overflow and edge cases |
| 90–120 | Hacking phase (Div. 2): read others' A/B for overflow, `int` vs `long long`, `n = 1` |

The literal loop for a single problem:

1. **Read the constraints first.** They select the complexity, which selects the algorithm.
2. **Write the brute force in your head** for small $n$. It is your oracle for stress testing (Chapter 02).
3. **Write the solution.** Compile with `-Wall`.
4. **Test the samples**, then the three edge cases: $n = 1$, all-equal input, maximum values (`10^9`, `10^{18}`).
5. **Submit.** Read the verdict and the *test number*. Test 3 of 40 usually means "small random case" — logic. Test 39 of 40 usually means "maximum size" — complexity or overflow.

### 10.1 The pre-submit checklist

Run this in your head in ten seconds; it catches most of your WA:

- Is there a $t$ in the input? Did I read it exactly once?
- Does anything multiply two values near $10^9$? → `long long` (and `1LL *`).
- Does the answer fit? `int` holds $2.1\cdot 10^9$; sums of $10^6$ values of $10^9$ do not.
- `n = 1`? Empty input? All elements equal?
- Did I clear global arrays between test cases? (Multi-test + global `vector` = the classic WA on test 2.)
- Output format: newline at the end? No trailing space? Correct number of lines?

---

## 11. How judging works

A judge is a script. For each test file $i$ it: runs your binary with stdin = `input_i`, stdout captured, a wall-clock timer, a memory cap; then compares your output to `output_i` **byte for byte, ignoring trailing whitespace on each line** (most checkers do; some are strict).

| Concept | Meaning |
|---|---|
| **Test file** | One input/output pair; a problem has 20–200 of them |
| **Pretests** | The first few tests, run immediately when you submit on Codeforces. Fast feedback, deliberately weak |
| **System tests** | The full set, run after the round. This is where "pretests passed" turns into "WA on test 47" |
| **Subtasks** | Groups of tests worth partial credit (IOI, CSES-style, some CodeChef). A wrong answer in a subtask may zero the whole group |
| **Checker** | Program that compares output. Default is token-by-token equality; a *special judge* accepts any valid answer (e.g. "output any shortest path") |
| **Hacking** | Codeforces Div. 2/Educational: after pretests pass, others may submit a test that breaks your code |
| **Interactor** | For interactive problems, the judge *talks* to your program; your output is a query, not an answer |

Consequences you must internalise:

- **Pretests passing means nothing.** They are designed to let you continue, not to validate you. A solution accepted at minute 20 can fail system tests at midnight.
- **Subtasks reward partial solutions.** A brute force over $n \le 20$ that earns 30 points is worth more than an unfinished $\mathcal{O}(n \log n)$. Always submit the brute force first if the full solution is not ready — it costs 30 seconds and locks in points.
- **Trailing whitespace is usually forgiven, missing newlines usually are not.** Print `'\n'` after every line including the last.
- **The judge compiles with its own flags.** Codeforces uses something like `g++ -O2 -std=gnu++17`. Your `-O3 -Ofast` local build may behave differently, especially on floating point.

---

## 12. The fully annotated starter template

This is the version to keep. Every line is load-bearing; every comment tells you *when to delete it*.

```cpp
// ---------------------------------------------------------------
//  Competitive programming template — C++17
//  Save as: <problem>.cpp   Compile: g++ -O2 -std=c++17 -Wall
// ---------------------------------------------------------------
#include <bits/stdc++.h>          // 1. every standard header, one line
using namespace std;              // 2. drop the std:: prefix (safe in a single-file contest program)

using ll  = long long;            // 3. 64-bit: use it for anything that can exceed 2e9
using ull = unsigned long long;   // 4. rare; hashing and bit tricks
using pii = pair<int, int>;       // 5. for sorting pairs, priority queues
using vi  = vector<int>;          // 6. the workhorse container

#define all(x)  begin(x), end(x)  // 7. sort(all(v)), v.erase(unique(all(v)), v.end())
#define sz(x)   (int)(x).size()   // 8. the (int) cast kills -Wsign-compare warnings
#define pb      push_back         // 9. pure typing speed, no semantic change

const int  INF  = 1e9;            // 10. bigger than any input, small enough that INF+INF fits in int
const ll   LINF = 4e18;           // 11. for Dijkstra/floyd distances; 4e18 < LLONG_MAX
const int  MOD  = 1'000'000'007;  // 12. digit separators are C++14, use them

// ---- optional debug block: compiles to nothing on the judge ----
#ifdef LOCAL
#define dbg(x) cerr << #x << " = " << (x) << '\n'
#else
#define dbg(x) ((void)0)
#endif

void solve() {                    // 13. one function per test case
    int n;
    cin >> n;                     // 14. read n
    vi a(n);                      // 15. size-known vector: no reallocation
    for (int &x : a) cin >> x;    // 16. reference binding: writes into the vector, no copies

    ll sum = 0;
    for (int x : a) sum += x;     // 17. accumulate in ll, not int
    cout << sum << '\n';          // 18. '\n', never endl (see Chapter 03)
}

int main() {
    ios_base::sync_with_stdio(false);  // 19. decouple C and C++ streams: 2-5x faster cin
    cin.tie(nullptr);                  // 20. stop flushing cout before every cin

    int t = 1;
    cin >> t;                     // 21. DELETE if the problem has a single test case
    while (t--) solve();          // 22. the multi-test loop

    return 0;                     // 23. exit status 0
}
```

The three lines you will change most often:

| Line | Change when |
|---|---|
| 21 | The problem has no `t` → delete it (keep `int t = 1;`) |
| 12 | The problem gives a different modulus → change `MOD` |
| 19–20 | The problem is interactive → delete both, use `endl` |

And the minimal version, for when you are typing into a judge's built-in editor with no template:

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);
    int n; cin >> n;
    vector<long long> a(n);
    for (auto &x : a) cin >> x;
    long long s = 0;
    for (auto x : a) s += x;
    cout << s << '\n';
    return 0;
}
```

---

## Key Takeaways

- The **constraints select the complexity**; read them before the story of the problem. $n \le 2\cdot10^5$ almost always means $\mathcal{O}(n\log n)$.
- A judge compiles with its own flags (typically `-O2 -std=gnu++17`) and compares bytes. Your local `-O0` timings are meaningless — always time with `-O2`.
- `#include` is textual pasting performed by the preprocessor; `<bits/stdc++.h>` is a GCC convenience, not standard C++, and costs ~1 s of compile time.
- `main` must return `int`; a non-zero exit status or a signal is reported as **RE** even when the output is right.
- `using namespace std;` is correct CP style and bad library style — it hides `std::count`, `std::distance`, `std::left`, `std::right`. Name your variables `cnt`, not `count`.
- Errors at stage 1 (preprocess), 2 (compile) and 4 (link) are all reported as **CE** but need completely different fixes.
- `-fsanitize=address,undefined` and `-D_GLIBCXX_DEBUG` are local-only tools that convert silent WA into a located crash; never submit those builds.
- The template exists to remove decisions, not to show off. The most common template bug is leaving `cin >> t` in a single-test problem.
- `typedef` and `using` are equivalent for simple types; `using` is required for alias templates.
- Over-macroing (`#define int long long`, `#define left`, `#define endl '\n'`) changes semantics, not just typing speed, and its bugs are invisible until the judge finds them.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---------|--------------|-----|
| 1 | Reading the statement before the constraints | You design an $\mathcal{O}(n^2)$ solution for $n = 2\cdot10^5$ and TLE at test 3 | Read constraints first, pick the complexity, then read the story |
| 2 | Leaving `cin >> t` in a single-test problem | `t` swallows $n$; `solve()` runs $n$ times on garbage → WA on test 1 | Check the input format; comment the line out explicitly |
| 3 | `void main()` or missing `return 0` | Rejected or reported as RE by strict judges | Always `int main() { ... return 0; }` |
| 4 | Timing with `-O0` | 3–10× slower than the judge; false TLE panic or false confidence | Compile with the same `-O2` the judge uses |
| 5 | Naming a global `count`, `left`, `right`, `data` | Collides with `std::` names under `using namespace std;` | Prefix or abbreviate: `cnt`, `lo`, `hi`, `val` |
| 6 | Submitting the sanitizer build | `-fsanitize` is 2–10× slower and aborts where the judge would not | Keep two build commands: `dbg` and `fast` |
| 7 | `#define sq(x) x*x` | `sq(1+2)` expands to `5`; `sq(i++)` increments twice | Use an `inline` function or a lambda |
| 8 | Trusting pretests | System tests and hacks find the overflow you skipped | Stress-test locally against a brute force before the round ends |
| 9 | Forgetting to clear globals between test cases | Test 1 passes, test 2 inherits stale data → WA | Declare inside `solve()`, or `fill`/`assign` at the top |
| 10 | Printing with `endl` in a tight loop | Flush per line: seconds of wasted time on $10^6$ lines | Use `'\n'`; reserve `endl` for interactive problems |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|--------------|----------------|------------|
| 1 | Print "Hello World" / basic I/O round-trip | CSES *Introductory Problems* (first problem) | Easy |
| 2 | Read $n$ then $n$ integers, print their sum | AtCoder ABC A/B | Easy |
| 3 | Multi-test-case sum with $t$ on the first line | Codeforces Div. 3 A | Easy |
| 4 | Sum of $10^6$ integers — deliberately test `cin` vs `scanf` | CSES (any $\mathcal{O}(n)$ problem with $n = 10^6$) | Easy |
| 5 | Compile the same file with `-O0`, `-O2`, `-O3` and compare runtimes | Local experiment, no judge | Easy |
| 6 | Deliberately cause CE / RE / TLE / MLE on a local judge and identify each verdict | Local experiment | Easy–Medium |
| 7 | Two-file program (`main.cpp` + `solver.cpp`) with a shared header | Local experiment | Medium |
| 8 | Solve any Div. 2 A in under 8 minutes including template paste | Codeforces Div. 2 A | Medium |

---

## Next Up

→ [Topic 02 — Understanding Comments, Errors, Warnings](Topic%2002%20-%20Understanding%20Comments%2C%20Errors%2C%20Warnings.md)