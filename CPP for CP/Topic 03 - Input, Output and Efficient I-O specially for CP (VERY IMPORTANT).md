# Input, Output, and Efficient I/O — The Chapter That Saves Your Rating

Two contestants write the same $\mathcal{O}(n \log n)$ algorithm. One gets AC in 0.6 s. The other gets **TLE on test 12** and spends the next hour convinced his segment tree is wrong. The only difference is eleven characters: `ios_base::sync_with_stdio(false);`. I/O is not a beginner topic you graduate from — it is the constant factor sitting in front of every algorithm you will ever write, and it is the cheapest rating you will ever buy.

> **Prerequisites:** [Topic 01 — C++ Basics and Syntax](Topic%2001%20-%20CPP%20Basics%20and%20Syntax%20Introduction.md), [Topic 02 — Errors, Warnings, Debugging](Topic%2002%20-%20Understanding%20Comments%2C%20Errors%2C%20Warnings.md).
> **You will be able to:**
> - Explain exactly what `cin`, `cout`, `sync_with_stdio` and `tie` do, and what breaks when you disable them.
> - Measure and predict the cost of every I/O style, from `cin` to a hand-written `fread` scanner.
> - Read any input format: until EOF, unknown token counts, lines with spaces, character grids.
> - Write and use a production-quality fast reader and fast writer.
> - Handle interactive problems, file-based judges, `getline` traps, and `double`/`__int128` formatting correctly.

---

## 1. Why I/O decides contests

Here is the arithmetic that matters. A 1-second limit gives you about $10^8$ simple operations. Now count what `cin >> x` costs: it constructs a sentry, checks stream state, locks the buffer, parses digits through a locale-aware facet, and handles whitespace. Realistically **50–150 ns per integer** without optimisation. A million integers is then 0.05–0.15 s of pure input — often 20% of your budget before a single line of your algorithm runs.

Now add output. `cout << x << '\n'` for $10^6$ values, written with `endl` instead of `'\n'`, forces $10^6$ flushes: **several seconds**, and an automatic TLE on a problem whose algorithm takes 0.1 s.

> *A contest story: in a Codeforces Div. 2 C with $n \le 10^6$ and a 1-second limit, an accepted solution differs from a TLE'd one only by `sync_with_stdio(false)`. The algorithm was right both times. The rating went down anyway.*

The rest of this chapter is about never being that second contestant.

---

## 2. How `cin` and `cout` actually work

`cin` and `cout` are objects of type `std::istream` / `std::ostream`, each owning a `streambuf` — a fixed-size buffer (typically 8 KB for `stdio`-backed buffers) that sits between your program and the operating system.

- **Reading**: `cin >> x` fills the buffer with a `read()` from the OS (one syscall per 8 KB), then parses digits out of the buffer in user space. The syscall count is fine; the *per-call parsing overhead* is what costs.
- **Writing**: `cout << x` formats into the buffer and returns. The buffer is written out when it fills, when the stream is flushed (`endl`, `flush`, `unitbuf`), or at program exit.

Two consequences you must internalise:

1. **Output is delayed.** If your program crashes, whatever is still in the `cout` buffer never appears. A `SIGSEGV` can therefore look like "no output at all".
2. **`cout` is tied to `cin`.** By default, before every read from `cin`, `cout` is flushed. That is what makes interactive programs work at all — and what silently makes your non-interactive program slower.

### 2.1 What `sync_with_stdio(false)` really does

By default, C++ streams are **synchronised** with C stdio: `printf` and `cout` share one buffer, so mixing them produces correctly ordered output, and `scanf` and `cin` share the same underlying file position. To guarantee that, every `cin` operation goes through the shared `stdio` layer with extra bookkeeping.

`ios_base::sync_with_stdio(false)` switches `cin`/`cout` to their **own private buffers**, bypassing the C layer. Input becomes dramatically faster; the price is that `printf`/`scanf` and `cin`/`cout` are no longer coordinated.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);   // must be called before any I/O
    cin.tie(nullptr);                   // must be called before any I/O

    int n;
    cin >> n;
    cout << n * 2 << '\n';
    return 0;
}
```

Rules:

- Call both **at the top of `main`, before the first read or write**. Calling them later is undefined in practice.
- After `sync_with_stdio(false)`, **never mix** `printf`/`puts`/`scanf`/`getchar` with `cin`/`cout`. Output order is not guaranteed: your `printf` debug line can appear before text that logically preceded it, and interleaved writes can corrupt each other.
- If you need `scanf`-speed *and* `cout`, pick one family and stay in it.

### 2.2 What `cin.tie(nullptr)` really does

`cin` has a *tied* stream: `cout`. Before each formatted extraction, the tied stream is flushed. For a program that does $10^6$ reads and prints at the end, that flush is wasted work — and for a program that prints inside a loop, it is catastrophic.

```cpp
cin.tie(nullptr);    // "do not flush cout before reading"
```

Only matters when you interleave reads and writes heavily — which is exactly the interactive case, which is exactly why you must *not* do this in interactive problems (§12).

---

## 3. `endl` vs `'\n'`

```cpp
cout << x << endl;    // writes '\n' AND flushes the buffer  (std::endl is a function)
cout << x << '\n';    // writes one newline character, no flush
```

`endl` is a manipulator: `ostream& endl(ostream&)` writes `'\n'` and calls `flush()`. A flush on a file-backed stream is a `write()` syscall. In a loop of $10^6$ iterations, that is $10^6$ syscalls instead of ~100.

Measured on the same machine, printing $10^6$ integers one per line:

| Statement | Time | Notes |
|---|---|---|
| `cout << i << endl;` | 8 700 ms | One flush per line |
| `cout << i << '\n';` | 585 ms | Buffered, ~100 syscalls |
| `cout << i << '\n';` with `sync_with_stdio(false)` | 830 ms | Same order; noise dominates |
| `printf("%d\n", i);` | 4 100 ms | *MinGW's* `printf` is unusually slow — see caveat |

**15×** from one keyword. The absolute numbers are from GCC 6.3 on Windows, where `printf` goes through the MSVCRT; on a Linux judge with glibc, `printf` and unsynced `cout` land within ~1.5× of each other and both sit near the `'\n'` row. The universally portable fact is the first two rows: **`endl` in a loop is 5–20× slower than `'\n'` on every platform ever measured.**

Use `endl` only when you *need* a flush (interactive problems). Everywhere else: `'\n'`.

---

## 4. `scanf` and `printf`: the C way

```cpp
#include <cstdio>

int main() {
    int n;
    long long x;
    double d;
    char s[64];

    scanf("%d %lld %lf %63s", &n, &x, &d, s);      // & for everything except arrays
    printf("%d %lld %.2f %s\n", n, x, d, s);
    return 0;
}
```

### 4.1 Format specifiers you actually need

| Specifier | `scanf` expects | `printf` expects | Use for |
|---|---|---|---|
| `%d` | `int*` | `int` | 32-bit integers |
| `%u` | `unsigned*` | `unsigned` | Unsigned |
| `%ld` | `long*` | `long` | `long` (64-bit on Linux, 32-bit on Windows!) |
| `%lld` | `long long*` | `long long` | **The one you want for 64-bit** |
| `%llu` | `unsigned long long*` | `unsigned long long` | 64-bit unsigned |
| `%zu` | `size_t*` | `size_t` | `v.size()`, `sizeof` |
| `%c` | `char*` | `int` (char promoted) | Single character, **does not skip whitespace** |
| `%s` | `char*` | `char*` | Whitespace-delimited word |
| `%f` | `float*` | `double` | **Asymmetric** — see below |
| `%lf` | `double*` | `double` | Doubles in `scanf` |
| `%Lf` | `long double*` | `long double` | Long doubles |
| `%x` / `%o` | `unsigned*` | `unsigned` | Hex / octal |
| `%p` | `void**` | `void*` | Pointers |
| `%%` | — | — | A literal `%` |

The `%f` / `%lf` asymmetry is a classic interview question and a real CP bug: in `printf`, `float` arguments are promoted to `double`, so `%f` prints a `double`. In `scanf` there is no promotion (it takes pointers), so `%f` writes a **4-byte float** into what you passed. Pass a `double*` to `%f` and you get garbage.

### 4.2 Width and padding

```cpp
printf("%5d|", 42);        // "   42|"
printf("%-5d|", 42);       // "42   |"
printf("%05d\n", 42);      // "00042"
printf("%.3f\n", 3.14159); // "3.142"
printf("%.10f\n", 1.0/3);  // "0.3333333333"
```

Useful when a problem requires zero-padded output (times, IDs).

### 4.3 The `%d` + `long long` bug

```cpp
long long x = 5000000000LL;
printf("%d\n", x);        // undefined behaviour — prints garbage (often 705032704)
printf("%lld\n", x);      // correct
```

`%d` reads 4 bytes of an 8-byte argument. It compiles with no warning unless you add `-Wformat` (included in `-Wall`), which is one more reason to compile with `-Wall`. The same class of bug: `printf("%d", v.size())` on 64-bit — use `%zu` or cast to `int`.

---

## 5. Measured: four input styles on $10^6$ integers

Test: read $10^6$ integers (3.9 MB file), sum them, print the sum. Built with `g++ -O2`. Input redirected from a file.

| Style | Time | Relative | Verdict |
|---|---|---|---|
| `cin >> x` (default) | 1 580 ms | 47× | Never use for $n > 10^5$ |
| `cin >> x` + `sync_with_stdio(false)` + `tie(nullptr)` | 910 ms | 27× | The default choice |
| `scanf("%d", &x)` | 500 ms | 15× | Use if you already write C-style code |
| Custom `fread` scanner | 34 ms | 1× | Use when the limit is brutal or $n \ge 10^6$ |

Environment: GCC 6.3 (MinGW), Windows 10, warm file cache. Absolute numbers differ on your judge — the *ratios* are what transfer, and they transfer well: unsynced `cin` is typically 2–5× faster than synced `cin`, `scanf` sits between, and a `fread` scanner is 5–15× faster than `scanf` because it makes one syscall instead of one per format spec.

### 5.1 When is the custom reader actually worth it?

| Situation | Use |
|---|---|
| $n \le 10^5$ | Unsynced `cin` is plenty. Spend your time on the algorithm |
| $n \le 10^6$, comfortable time limit | Unsynced `cin` usually passes; measure if nervous |
| $n \ge 10^6$ with a tight limit, or $10^7$+ tokens | Custom reader |
| Input is a huge string / grid of characters | `fread` the whole file, or `cin` unsynced (both fine) |
| Interactive problem | Plain `cin`/`cout` + `endl` (§12) |

The custom reader is a 30-line insurance policy. Keep it in your template file; paste it when the constraints say $10^6$ or more.

---

## 6. Reading until EOF

When the problem says "input contains several test cases until end of file", there is no `t` and no `n` count.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int x;
    long long sum = 0;
    while (cin >> x) {        // false when extraction fails (EOF or bad input)
        sum += x;
    }
    cout << sum << '\n';
    return 0;
}
```

With `scanf`:

```cpp
int x;
while (scanf("%d", &x) == 1) {   // == 1 means "one item converted"
    /* ... */
}
```

Both idioms are correct because the stream's `operator bool` is false after a failed extraction. Two traps:

- `while (!cin.eof())` is **wrong**: `eof()` only becomes true *after* a failed read, so you process the last value twice.
- `while (cin >> x, !cin.eof())` is also wrong, for the same reason.

### 6.1 Interleaved pairs

A common shape: "each line contains two integers, until EOF".

```cpp
int a, b;
while (cin >> a >> b) {
    cout << a + b << '\n';
}
```

Works because the chained extraction fails as soon as either read fails.

---

## 7. `getline` and the trailing-newline trap

`cin >> x` stops *before* the whitespace that terminated the number. That whitespace — usually `'\n'` — is still in the stream. The next `getline` reads it and returns an **empty string**.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;
    string line;
    getline(cin, line);            // reads "" — the newline after n
    cout << "[" << line << "]\n";
    getline(cin, line);            // reads the real first line
    cout << "[" << line << "]\n";
    return 0;
}
```

Input:

```text
3
hello world
```

Output:

```text
[]
[hello world]
```

The fix is one line:

```cpp
cin.ignore(numeric_limits<streamsize>::max(), '\n');   // eat everything up to and including '\n'
```

or the simpler `cin.ignore();` when you know exactly one character is pending (fine in practice for `cin >> n` followed by `getline`, but fragile if there is trailing whitespace on the line).

### 7.1 The bulletproof line-reading pattern

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    if (!(cin >> n)) return 0;
    string s;
    getline(cin, s);                        // discard the rest of the "n" line

    vector<string> lines(n);
    for (int i = 0; i < n; i++) getline(cin, lines[i]);

    for (auto &l : lines) cout << l.size() << '\n';
    return 0;
}
```

Using `getline` for the discard (instead of `ignore`) is *slightly* more robust: it handles the case where the first line has extra junk after `n`.

### 7.2 Mixing `>>` and `getline` safely

| Need | Code |
|---|---|
| Read `n`, then `n` whole lines | `cin >> n; cin.ignore(LLONG_MAX,'\n');` then `getline` × n |
| Read `n`, then `n` words | `cin >> n;` then `cin >> s` × n — no `ignore` needed |
| Read `n`, then `n` lines each starting with a word | Read the word with `>>`, then `getline` the rest of the line |
| Read a line that may contain trailing spaces | `getline` and `stringstream` |

---

## 8. Lines, tokens, spaces, and grids

### 8.1 A line with an unknown number of integers

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string line;
    getline(cin, line);
    stringstream ss(line);
    vector<int> a;
    int x;
    while (ss >> x) a.push_back(x);          // reads until the line is exhausted

    long long sum = 0;
    for (int v : a) sum += v;
    cout << a.size() << ' ' << sum << '\n';
    return 0;
}
```

`stringstream` is simple and slow-ish (~3–5× `scanf`), which is fine for one line and *not* fine inside a loop over $10^5$ lines. For that, parse manually or use the fast reader.

### 8.2 Strings with spaces vs words

```cpp
string word;
cin >> word;                 // stops at whitespace: "hello"

string line;
getline(cin, line);          // keeps spaces: "hello world"
```

Problems state which they want. "A string $s$ consisting of lowercase Latin letters" → `cin >> s`. "A line of text" → `getline`.

### 8.3 Character grids

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;
    vector<string> g(n);
    for (int i = 0; i < n; i++) cin >> g[i];   // each row: no spaces, e.g. "..#.."

    int dots = 0;
    for (int i = 0; i < n; i++)
        for (int j = 0; j < m; j++)
            dots += (g[i][j] == '.');
    cout << dots << '\n';
    return 0;
}
```

If the grid is given **with spaces** (`# . # . .`), reading into `string` gives you one character per string. Either read `char` in a loop, or read the string and check its length. This inconsistency between judges is a classic 10-minute loss:

```cpp
vector<string> g(n, string(m, '?'));
for (int i = 0; i < n; i++)
    for (int j = 0; j < m; j++)
        cin >> g[i][j];       // works for BOTH "..#.." and "# . # . ."
```

`cin >> char` skips whitespace and takes the next non-space character — the portable way to read a grid.

---

## 9. The custom fast reader

The idea: do **one** `fread` of a large block, then parse digits directly out of memory. No streams, no locales, no per-call function overhead.

```cpp
#include <bits/stdc++.h>
using namespace std;

class FastScanner {
    static const int BUFSZ = 1 << 16;      // 64 KB
    char buf[BUFSZ];
    int idx = 0, len = 0;

    int readByte() {
        if (idx >= len) {
            len = (int)fread(buf, 1, BUFSZ, stdin);
            idx = 0;
            if (len <= 0) return -1;       // EOF
        }
        return (unsigned char)buf[idx++];
    }

public:
    // Reads the next integer (handles an optional '-' and any whitespace).
    // Returns false at EOF.
    template <class T>
    bool nextInt(T &out) {
        int c = readByte();
        while (c != -1 && (c < '0' || c > '9') && c != '-') c = readByte();
        if (c == -1) return false;

        bool neg = false;
        if (c == '-') { neg = true; c = readByte(); }

        T val = 0;
        while (c >= '0' && c <= '9') {
            val = val * 10 + (c - '0');
            c = readByte();
        }
        out = neg ? -val : val;
        return true;
    }

    // Reads the next whitespace-delimited token.
    bool nextString(string &s) {
        int c = readByte();
        while (c != -1 && c <= ' ') c = readByte();
        if (c == -1) return false;
        s.clear();
        while (c > ' ') { s.push_back((char)c); c = readByte(); }
        return true;
    }

    // Reads one non-whitespace character (for grids).
    bool nextChar(char &out) {
        int c = readByte();
        while (c != -1 && c <= ' ') c = readByte();
        if (c == -1) return false;
        out = (char)c;
        return true;
    }
};

int main() {
    FastScanner fs;
    int n;
    if (!fs.nextInt(n)) return 0;

    long long sum = 0;
    for (int i = 0; i < n; i++) {
        long long x;
        fs.nextInt(x);
        sum += x;
    }
    printf("%lld\n", sum);
    return 0;
}
```

Notes on this implementation, because you will be asked in an interview and because the bugs are subtle:

- `readByte` returns `-1` at EOF. `int` is used so `-1` is distinguishable from any byte value.
- The `(unsigned char)` cast matters: if `char` is signed (the default on x86), bytes above 127 become negative and the digit tests break on non-ASCII input.
- `while (c != -1 && (c < '0' || c > '9') && c != '-')` skips whitespace *and* any other junk, and stops at a minus sign. If EOF arrives mid-scan you return `false`.
- `T` is a template parameter so the same function reads `int`, `long long`, and `unsigned`.

### 9.1 The `getchar_unlocked` variant (shorter, POSIX only)

```cpp
#include <bits/stdc++.h>
using namespace std;

static inline int readInt() {
    int x = 0, c = getchar_unlocked();
    while (c < '0' || c > '9') c = getchar_unlocked();
    while (c >= '0' && c <= '9') { x = x * 10 + (c - '0'); c = getchar_unlocked(); }
    return x;                          // no negatives, no EOF handling
}
```

`getchar_unlocked` skips the `stdio` lock. It exists on Linux/macOS (`<cstdio>`), not on Windows. It is roughly 2–3× slower than the `fread` block version but 2–3× faster than `scanf`.

### 9.2 Reading the entire file at once

For maximum speed and simplicity on inputs up to a few hundred MB:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    // slurp all of stdin
    string data;
    char tmp[1 << 16];
    size_t got;
    while ((got = fread(tmp, 1, sizeof tmp, stdin)) > 0) data.append(tmp, got);

    size_t i = 0, N = data.size();
    long long sum = 0;
    while (i < N) {
        while (i < N && (data[i] < '0' || data[i] > '9') && data[i] != '-') i++;
        if (i >= N) break;
        bool neg = (data[i] == '-');
        if (neg) i++;
        long long x = 0;
        while (i < N && data[i] >= '0' && data[i] <= '9') { x = x * 10 + (data[i] - '0'); i++; }
        sum += neg ? -x : x;
    }
    printf("%lld\n", sum);
    return 0;
}
```

This is the version I use in practice: it is 20 lines, obviously correct, and needs no `BUFSZ` reasoning. Cost: the whole input in memory (usually a few MB — check the input size, not the output size).

---

## 10. Fast output

The mirror image. `cout` is usually good enough once unsynced; the pathological case is $10^6$ lines with many numbers each.

### 10.1 The simple version: build a string, write once

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 1000000;
    string out;
    out.reserve(8 * n);                       // avoid reallocation; ~8 bytes per line
    for (int i = 0; i < n; i++) {
        out += to_string(i);
        out += '\n';
    }
    fwrite(out.data(), 1, out.size(), stdout);
    return 0;
}
```

`reserve` is the whole trick. Without it, `string` doubles its capacity ~20 times and copies ~16 MB total.

### 10.2 A proper buffered writer

```cpp
#include <bits/stdc++.h>
using namespace std;

class FastOutput {
    static const int BUFSZ = 1 << 16;
    char buf[BUFSZ];
    int idx = 0;

    void flushBuf() {
        if (idx > 0) { fwrite(buf, 1, idx, stdout); idx = 0; }
    }

public:
    ~FastOutput() { flushBuf(); }

    void put(char c) {
        if (idx == BUFSZ) flushBuf();
        buf[idx++] = c;
    }

    void writeInt(long long x) {
        if (x == 0) { put('0'); return; }
        unsigned long long mag;
        if (x < 0) { put('-'); mag = 0ULL - (unsigned long long)x; }  // safe for LLONG_MIN
        else       { mag = (unsigned long long)x; }
        char tmp[24];
        int n = 0;
        while (mag > 0) { tmp[n++] = char('0' + (int)(mag % 10)); mag /= 10; }
        while (n > 0) put(tmp[--n]);
    }

    void writeString(const string &s) {
        for (char c : s) put(c);
    }

    void writeChar(char c) { put(c); }

    void newline() { put('\n'); }
};

int main() {
    FastOutput out;
    int n = 1000000;
    for (int i = 0; i < n; i++) {
        out.writeInt(i);
        out.newline();
    }
    return 0;                    // destructor flushes the tail
}
```

Details that matter:

- The destructor flush is what makes this safe with `return 0;` — but **not** with `exit(0)` from inside a function or an `abort()`. If you call `exit()`, flush manually first.
- `0ULL - (unsigned long long)x` gives the magnitude of `LLONG_MIN` without signed overflow. The naive `x = -x` is UB for `LLONG_MIN`, and UBSan will tell you so.
- Never mix `FastOutput` with `cout` — two independent buffers, arbitrarily interleaved output.

### 10.3 Do you need any of this?

| Output size | Recommended |
|---|---|
| $< 10^5$ numbers | `cout` with `'\n'` |
| $10^5$–$10^6$ numbers | `cout` with `'\n'`, unsynced — usually fine |
| $> 10^6$ numbers, or many per line | Build a `string` with `reserve`, or `FastOutput` |
| Output must be flushed to an interactor | `endl` / `flush` — never a custom buffer |

---

## 11. Formatting numbers

### 11.1 Doubles

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    double x = 1.0 / 3.0;
    cout << x << '\n';                                  // 0.333333

    cout << fixed << setprecision(9) << x << '\n';      // 0.333333333
    cout << setprecision(12) << x << '\n';              // 0.333333333333 (still fixed)
    cout << defaultfloat << setprecision(6) << x << '\n'; // 0.333333

    double big = 1234567.891;
    cout << fixed << setprecision(2) << big << '\n';    // 1234567.89
    cout << scientific << setprecision(3) << big << '\n'; // 1.235e+06
    return 0;
}
```

```text
0.333333
0.333333333
0.333333333333
0.333333
1234567.89
1.235e+06
```

**`fixed` is sticky.** Once you write `cout << fixed`, every later double in that stream is fixed too. Problems that say "print the answer with absolute error at most $10^{-6}$" are satisfied by `fixed << setprecision(9)`; problems that say "print exactly 6 digits after the decimal" *require* it.

With `printf`:

```cpp
printf("%.9f\n", x);      // 9 digits after the point, doubles
printf("%.2Lf\n", ld);    // long double
```

### 11.2 `__int128`

`__int128` holds up to $1.7\cdot 10^{38}$ — enough for products of two $10^{18}$ values. It has no `cin`/`cout` overload, so you print it yourself:

> Note: `__int128` requires a 64-bit GCC target. It works on every Linux judge (Codeforces, AtCoder, CSES, CodeChef) and on 64-bit WSL, but **not** on 32-bit MinGW or MSVC — there, the type simply does not exist.

```cpp
#include <bits/stdc++.h>
using namespace std;

string toString128(__int128 x) {
    if (x == 0) return "0";
    bool neg = x < 0;
    unsigned __int128 mag = neg ? (unsigned __int128)(-x) : (unsigned __int128)x;
    string s;
    while (mag > 0) { s.push_back(char('0' + (int)(mag % 10))); mag /= 10; }
    if (neg) s.push_back('-');
    reverse(s.begin(), s.end());
    return s;
}

int main() {
    long long a = 1000000000000000000LL;
    __int128 p = (__int128)a * a;
    cout << toString128(p) << '\n';       // 1000000000000000000000000000000000000
    return 0;
}
```

Reading one is symmetric: read a `long long` and cast, or read two `long long`s and multiply in `__int128`.

---

## 12. Interactive problems: the opposite rules

In an interactive problem your program *talks* to a judge program. You print a query, the interactor answers on your `stdin`. If you do not flush after a query, the interactor waits forever and you get **Idleness limit exceeded**.

The rules, in order of how often they are broken:

1. **Flush after every query.** Use `endl` (which flushes) or `cout.flush()`. Never `'\n'` alone on a query line.
2. **Do not use `sync_with_stdio(false)`.** It is not fatal if you flush correctly, but it removes the safety net: with synchronised streams, `'\n'` on a terminal-like stream is line-buffered and often appears anyway. Unsynced `cout` is *fully* buffered, so a forgotten flush silently deadlocks. Interactive contests also often have an interactor written in C that mixes `printf`, which sync-off decouples. The convention everyone follows: **no sync-off in interactive problems.**
3. **Do not use a custom output buffer.** Same reason, worse.
4. **Debug to `stderr` only.** `cerr` is unbuffered and never compared; `cout` debug lines will be read as queries.
5. **Read exactly what the interactor sends.** Extra reads block.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;                       // 1 <= hidden number <= n, say

    int lo = 1, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        cout << "? " << mid << endl;    // endl flushes — mandatory
        string resp;
        cin >> resp;                    // interactor answers "yes" / "no"
        if (resp == "yes") hi = mid;
        else lo = mid + 1;
    }
    cout << "! " << lo << endl;
    return 0;
}
```

Cost: `endl` here is *correct* and you should not "optimise" it away. Interactive problems have small outputs — the flush costs nothing.

---

## 13. File I/O with `freopen`

A few judges (older Olympiad-style systems, some university judges, USACO's training mode) require named files: read `input.txt`, write `output.txt`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    freopen("input.txt", "r", stdin);
    freopen("output.txt", "w", stdout);

    int a, b;
    scanf("%d %d", &a, &b);
    printf("%d\n", a + b);
    return 0;
}
```

Alternatively with streams:

```cpp
ifstream fin("input.txt");
ofstream fout("output.txt");
int a, b;
fin >> a >> b;
fout << a + b << '\n';
```

| Situation | What to do |
|---|---|
| Problem statement mentions no files | Use stdin/stdout. **Most judges** |
| Statement says "input file: input.txt" | `freopen` both, and keep them as the first lines of `main` |
| You are testing locally | `freopen` is convenient; or just redirect: `./sol < in.txt > out.txt` |
| You use `freopen` and `sync_with_stdio(false)` | Fine — call `freopen` first, then the two ios lines |
| Interactive problem | **Never** `freopen`; the interactor uses the real pipes |

Forgetting `freopen` when the judge requires it gives **WA on every test** with correct-looking output on your machine — or RE, depending on whether the judge's sandbox has the file.

### 13.1 Local testing without touching your code

```bash
g++ -O2 -std=c++17 sol.cpp -o sol
./sol < in.txt > out.txt
diff out.txt expected.txt
```

On Windows PowerShell:

```powershell
Get-Content in.txt | .\sol.exe > out.txt
Compare-Object (Get-Content out.txt) (Get-Content expected.txt)
```

Keep `freopen` out of your submitted file unless the statement demands it.

---

## 14. Common I/O traps

| # | Trap | Symptom | Fix |
|---|---|---|---|
| 1 | `%d` with a `long long` | Garbage numbers, or correct on small values only | `%lld`; compile with `-Wall` |
| 2 | `scanf("%f", &d)` for a `double` | Garbage / zeros | `%lf` in `scanf` (but `%f` in `printf`) |
| 3 | `getline` right after `cin >> n` | First line comes back empty | `cin.ignore(...)` or a throwaway `getline` |
| 4 | Trailing space before `'\n'` | WA / PE on strict judges | Print separators *before* each element except the first, or check the statement |
| 5 | No newline at the very end | WA on some checkers | `cout << ans << '\n';` |
| 6 | Mixing `printf` and `cout` after sync-off | Scrambled or missing output | Pick one family |
| 7 | `ios_base::sync_with_stdio(false)` after the first read | Undefined; speedup does not apply | Move it to line 1 of `main` |
| 8 | `endl` in a $10^6$-iteration loop | TLE | `'\n'` |
| 9 | Reading `n` then `n` lines, but the lines were space-separated tokens | `getline` returns partial lines | Read with `>>` if tokens, `getline` if lines |
| 10 | `while (!cin.eof())` | Last value processed twice | `while (cin >> x)` |
| 11 | Output buffered, program crashes | "No output" although logic ran | Flush before crash tests, or use `cerr` for debug |
| 12 | Missing flush in an interactive problem | Idleness limit exceeded | `endl` / `flush` per query |
| 13 | `cout << setprecision(2) << x` without `fixed` | `1.2e+06` instead of `1200000.00` | `fixed << setprecision(2)` |
| 14 | `printf("%d\n", v.size())` | Garbage on 64-bit | `%zu` or `(int)v.size()` |
| 15 | Reading a grid with `cin >> g[i]` when cells are space-separated | Wrong dimensions / wrong characters | Read `char` by `char` |

### 14.1 Trailing spaces, precisely

The safe pattern for "print $n$ numbers separated by spaces":

```cpp
for (int i = 0; i < n; i++) {
    if (i) cout << ' ';
    cout << a[i];
}
cout << '\n';
```

`cout << a[i] << ' '` inside the loop leaves a trailing space. Most checkers ignore it; some do not. The `if (i)` version costs one branch per element and removes the question entirely.

---

## 15. When to use what

| Situation | Input | Output |
|---|---|---|
| Standard CF/AtCoder/CSES problem, $n \le 2\cdot10^5$ | `cin` unsynced | `cout` with `'\n'` |
| $n \le 10^6$, generous limit | `cin` unsynced (measure) | `cout` with `'\n'` |
| $n \ge 10^6$, tight limit | `FastScanner` or slurp-all | `FastOutput` or one big `string` |
| Already writing C-style code | `scanf` | `printf` |
| Interactive | `cin` **with** sync | `cout` + `endl` per query |
| Fixed decimal precision required | — | `fixed << setprecision(k)` |
| Requires `input.txt` | `freopen` | `freopen` |
| Reading whole lines with spaces | `getline` (+ `ignore`) | any |
| Grid of characters | `cin >> char` loop, or `vector<string>` | any |
| Answer can exceed $9.2\cdot10^{18}$ | — | `__int128` + `toString128` |

**The default, and it is right 95% of the time:**

```cpp
ios_base::sync_with_stdio(false);
cin.tie(nullptr);
```

...plus `'\n'` everywhere, and `endl` only when an interactor is listening.

---

## Key Takeaways

- Unsynced `cin` is the correct default. Synced `cin` is 2–5× slower and is the most common cause of "my $\mathcal{O}(n\log n)$ TLEs".
- `endl` writes a newline **and flushes**; in a $10^6$-iteration loop it is 5–20× slower than `'\n'`.
- `sync_with_stdio(false)` gives `cin` its own buffer; afterwards, mixing `scanf`/`printf` with `cin`/`cout` is undefined. `cin.tie(nullptr)` stops `cout` being flushed before every read.
- `scanf`/`printf` need exact specifiers: `%lld` for `long long`, `%lf` (in `scanf`) for `double`, `%zu` for `size_t`, `%Lf` for `long double`. `printf("%d", someLongLong)` is UB.
- A `fread`-based scanner is 5–15× faster than `scanf` because it makes one syscall per 64 KB instead of one parse per value. Worth it at $n \ge 10^6$.
- `getline` after `cin >> n` reads the leftover newline. Fix with `cin.ignore(numeric_limits<streamsize>::max(), '\n')`.
- `while (cin >> x)` is the correct EOF loop; `while (!cin.eof())` processes the last item twice.
- Build output in a `string` with `reserve` (or a fixed buffer) when you print millions of values; flush in a destructor, never rely on `exit()` to do it.
- Interactive problems invert every rule: `endl` per query, no sync-off, no custom buffer, debug only to `cerr`.
- `freopen("input.txt","r",stdin)` only when the statement names the file. Otherwise it turns every test into WA.
- `fixed` is sticky; `setprecision` alone switches to significant digits, not decimals.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---------|--------------|-----|
| 1 | Leaving `cin` synchronised | Hidden 2–5× slowdown; TLE with a correct algorithm | `sync_with_stdio(false); cin.tie(nullptr);` as the first lines of `main` |
| 2 | `endl` in a hot loop | One flush per line → seconds of syscalls | `'\n'` |
| 3 | Mixing `printf` and `cout` after sync-off | Output ordering breaks; text disappears | One family only |
| 4 | Forgetting `cin.ignore()` after `cin >> n` | First `getline` returns `""` | `ignore(max(), '\n')` |
| 5 | `printf("%d", ll)` | 4 bytes read from 8 → garbage | `%lld`, and compile with `-Wall` |
| 6 | `scanf("%f", &dbl)` | Writes 4 bytes into an 8-byte object | `%lf` in `scanf` |
| 7 | `while (!cin.eof())` | Last element read twice | `while (cin >> x)` |
| 8 | Trailing space in output | PE/WA on strict checkers | Print the separator before, not after |
| 9 | No final newline | WA on some judges | Always end with `'\n'` |
| 10 | Custom fast reader with `char` (signed) | Non-ASCII bytes look negative; parsing breaks | Cast to `unsigned char` |
| 11 | `-x` for `LLONG_MIN` in a writer | Signed overflow → UB, wrong digit on INT64_MIN | Convert to `unsigned long long` magnitude |
| 12 | `freopen` without being asked | WA on every test | Only when the statement names the file |
| 13 | No flush in an interactive program | Idleness limit exceeded | `endl` per query |
| 14 | `setprecision(2)` without `fixed` | Scientific notation or wrong digit count | `fixed << setprecision(2)` |
| 15 | `reserve` forgotten when building a big string | ~20 reallocations, 2× memory peak | `out.reserve(estimated_bytes)` |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|--------------|----------------|------------|
| 1 | Read $n$, then $n$ integers; print the sum — run all four I/O styles and time them | CSES *Introductory Problems*, or local | Easy |
| 2 | Multi-test-case sum with `t` on the first line | Codeforces Div. 3 A | Easy |
| 3 | Read integers until EOF and print their running sum | Any judge's "sum until EOF" task | Easy |
| 4 | Read `n`, then `n` lines with spaces; print each line reversed | Local / any judge | Easy |
| 5 | Read a grid of `.` and `#` and count connected components (needs grid I/O) | CSES *Graph* section / AtCoder ABC | Medium |
| 6 | Sum of $10^6$ integers under a 0.5 s limit | CSES (strict limits) | Medium |
| 7 | Print $10^6$ values with `fixed << setprecision(6)` | Local benchmark | Medium |
| 8 | Any interactive binary-search problem (guess the number) | Codeforces interactive tag | Medium |
| 9 | Multiply two $10^{18}$ values and print the exact product | Any judge, needs `__int128` | Medium |
| 10 | Rewrite a slow solution with `FastScanner` + `FastOutput` and compare runtimes | Local benchmark | Medium |

---

## Next Up

→ [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems%2C%20Variables%20and%20Datatypes.md)