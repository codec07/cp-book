# Topic 04 — Number Systems, Variables and Datatypes

Almost every wrong answer you will ever get falls into one of three buckets: wrong algorithm, wrong complexity, or **wrong type**. The third is the cheapest to fix and the most embarrassing to miss. A solution that is algorithmically perfect still prints `705082704` instead of `5000050000` if one multiplication happens in `int`. This chapter builds the mental model of how C++ stores numbers, exactly where each type breaks, and the reflexes that stop you from breaking it.

> **Prerequisites:** [Topic 01 — CPP Basics and Syntax](Topic%2001%20-%20CPP%20Basics%20and%20Syntax%20Introduction.md), [Topic 02 — Comments, Errors, Warnings](Topic%2002%20-%20Understanding%20Comments%2C%20Errors%2C%20Warnings.md), [Topic 03 — Input, Output and Efficient I/O](Topic%2003%20-%20Input%2C%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20%28VERY%20IMPORTANT%29.md)
> **You will be able to:**
> - Convert between binary, octal, decimal and hexadecimal by hand, in both directions.
> - Read a problem's constraints and immediately name the type that survives them.
> - Predict and prevent signed overflow, including the `n * (n + 1) / 2` disaster at $n = 10^5$.
> - Use `__int128` for modular multiplication when the modulus is near $10^{18}$.
> - Compare floating point numbers correctly, and write a binary search on doubles that terminates.
> - Read and write bitmasks fluently with the standard CP idioms.

---

## 1. Positional number systems

A positional system writes a number as a weighted sum of digits. In base $b$, the string $d_k d_{k-1} \dots d_0$ means

$$N = \sum_{i=0}^{k} d_i \, b^i, \qquad 0 \le d_i < b.$$

The base is just a *packaging choice*. The quantity 156 is the same whether you write `10011100`, `234`, `156` or `9C`. C++ accepts all four in source code and stores exactly one thing: a pattern of bits.

| Base | Name | Digits | C++ literal | Where you meet it |
|---|---|---|---|---|
| 2 | binary | `0`,`1` | `0b1010` (C++14) | Bitmasks, subset DP, `bitset` output |
| 8 | octal | `0`–`7` | `017` (leading zero!) | File permissions, `%o` |
| 10 | decimal | `0`–`9` | `15` | Problem statements, all human input |
| 16 | hexadecimal | `0`–`9`,`A`–`F` | `0xF`, `0xff` | Masks, hashes, memory dumps |

Warning about that leading zero: `int x = 010;` is **8**, not 10, and `int x = 08;` does not compile because `8` is not an octal digit. Never zero-pad an integer literal.

## 2. Hand conversion, both directions

**Decimal → binary: repeated division.** Divide by 2, keep the remainders, read them bottom-up.

```text
156 / 2 = 78  r 0     <- least significant
 78 / 2 = 39  r 0
 39 / 2 = 19  r 1
 19 / 2 =  9  r 1
  9 / 2 =  4  r 1
  4 / 2 =  2  r 0
  2 / 2 =  1  r 0
  1 / 2 =  0  r 1     <- most significant

read upwards: 10011100
```

**Binary → decimal: weighted sum**, or the faster double-and-add (Horner's rule): start at 0, and for each bit double the accumulator and add the bit.

$$10011100_2 = 128 + 16 + 8 + 4 = 156 \qquad (1,2,4,9,19,39,78,156)$$

**Hex ↔ binary: 4-bit grouping.** Each hex digit is one nibble, so conversion is string surgery, not arithmetic. Pad the binary from the **right**:

```text
1011 0110 1101  ->  B 6 D  ->  0xB6D = 11*256 + 6*16 + 13 = 2925
156 = 1001 1100 = 0x9C        (octal uses 3-bit groups: 010 011 100 = 0o234)
```

| Decimal | Binary | Hex | Octal |
|---|---|---|---|
| 0 | `0000` | `0x0` | `0` |
| 7 | `0111` | `0x7` | `7` |
| 10 | `1010` | `0xA` | `12` |
| 15 | `1111` | `0xF` | `17` |
| 16 | `0001 0000` | `0x10` | `20` |
| 255 | `1111 1111` | `0xFF` | `377` |
| 4096 | `0001 0000 0000 0000` | `0x1000` | `10000` |

Two anchors to memorise: `0xFF = 255` and `0x7FFFFFFF = 2147483647 = INT_MAX`. For code, `stoi(s, nullptr, 2)` parses a binary string and `bitset<8>(156).to_string()` prints one.

## 3. Two's complement: why `-1` is `0xFFFFFFFF`

C++ stores no minus sign. Signed integers use **two's complement**: the top bit carries weight $-2^{w-1}$, negation is `~x + 1`, and zero has exactly one representation.

| Bits | Unsigned | Signed | Bits | Unsigned | Signed |
|---|---|---|---|---|---|
| `0000` | 0 | 0 | `1000` | 8 | **-8** |
| `0011` | 3 | 3 | `1100` | 12 | -4 |
| `0111` | 7 | 7 | `1110` | 14 | -2 |
| `1111` | 15 | **-1** | `1001` | 9 | -7 |

`1111` means 15 or $-1$ depending only on the type you read it through. Scaled to 32 bits, that gives the most important identity in low-level C++: `0xFFFFFFFF` is `4294967295u` and `-1` — the same 32 bits.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a = -1;
    cout << (a == (int)0xFFFFFFFFu) << "\n";          // 1  (same bits)
    cout << hex << uppercase << (unsigned)a << "\n";  // FFFFFFFF
    cout << dec << (a + 1) << "\n";                   // 0
    cout << (unsigned)(-1) << "\n";                   // 4294967295
    cout << (a & 0xFF) << "\n";                       // 255
    return 0;
}
```

```text
1
FFFFFFFF
0
4294967295
255
```

Three consequences:

1. **Addition and subtraction "just work" across the sign boundary** — that is the point of the representation, and why `INT_MAX + 1` wraps to `INT_MIN` on real hardware (even though the standard calls it UB).
2. **The signed range is asymmetric**: $-2^{31}$ to $2^{31}-1$. There is no $+2147483648$ in `int`, so `abs(INT_MIN)` is still negative.
3. **Signed overflow is undefined behaviour**, not defined wraparound. Unsigned overflow *is* defined (modulo $2^w$). Optimisers exploit signed-overflow UB, which is why `for (int i = 0; i <= n; ++i)` with `n = INT_MAX` can loop forever.

## 4. The integer types, exactly

| Type | Bits | Min | Max | Max decimal digits | Survives $n^2$ up to |
|---|---|---|---|---|---|
| `char` (signed) | 8 | -128 | 127 | 3 | — |
| `unsigned char` | 8 | 0 | 255 | 3 | — |
| `short` | 16 | -32,768 | 32,767 | 5 | 181 |
| `unsigned short` | 16 | 0 | 65,535 | 5 | 255 |
| `int` | 32 | -2,147,483,648 | **2,147,483,647** | 10 | 46,340 |
| `unsigned int` | 32 | 0 | 4,294,967,295 | 10 | 65,535 |
| `long` (Windows) | 32 | -2.1e9 | 2.1e9 | 10 | 46,340 |
| `long` (Linux/GCC) | 64 | -9.2e18 | 9.2e18 | 19 | 3.0e9 |
| `long long` | 64 | -9,223,372,036,854,775,808 | **9,223,372,036,854,775,807** | 19 | 3.0e9 |
| `unsigned long long` | 64 | 0 | 18,446,744,073,709,551,615 | 20 | 4.2e9 |
| `__int128` (GCC) | 128 | $-2^{127}$ | $2^{127}-1$ | **39** | 1.8e19 |

The "survives $n^2$" column is the one to internalise. If $n \le 10^5$ and you sum pairwise products, $n^2 = 10^{10}$ does **not** fit in `int`; `long long` eats it comfortably.

**Never use `long` in competitive code.** On Codeforces (GCC, Linux, LP64) it is 64 bits; on Windows (LLP64) it is 32. Same source, different answers. Always write `long long`.

## 5. `sizeof` — the ground truth

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << "bool " << sizeof(bool) << " char " << sizeof(char)
         << " short " << sizeof(short) << " int " << sizeof(int) << "\n";
    cout << "long " << sizeof(long) << " longlong " << sizeof(long long)
         << " size_t " << sizeof(size_t) << " ptr " << sizeof(void*) << "\n";
    cout << "float " << sizeof(float) << " double " << sizeof(double)
         << " longdouble " << sizeof(long double) << "\n";
    return 0;
}
```

On a typical Codeforces box (x86-64 Linux, GCC):

```text
bool 1 char 1 short 2 int 4
long 8 longlong 8 size_t 8 ptr 8
float 4 double 8 longdouble 16
```

| Fact | Value | Why you care |
|---|---|---|
| 1 byte | 8 bits | `char` is the only type guaranteed to be 1 byte |
| `double` storage | 8 bytes | 64 bits total, only 53 for the mantissa |
| `long double` on Linux | 16 bytes (80-bit extended, padded) | ~18–19 significant digits |
| `long double` on MSVC | 8 bytes | **identical to `double`** — no extra precision |
| `bool` | 1 byte | `bool[10^7]` is 10 MB; `int[10^7]` is 40 MB |
| `size_t` | 8 bytes | Unsigned — source of the `-1` bug (§13) |

*Contest story: with a 256 MB limit, `bool vis[2000][2000]` costs 4 MB and `int vis[2000][2000]` costs 16 MB. Narrowing the type is a real optimisation when you allocate millions of cells.*

## 6. Integer overflow, and the death of `int` at $10^5$

Sum of $1..n$ is $\frac{n(n+1)}{2}$. For $n = 10^5$ that is $5{,}000{,}050{,}000$ — bigger than `INT_MAX`. But the *type of the expression* is `int`, because `n` is `int`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n = 100000;
    long long wrong = n * (n + 1) / 2;        // int * int happens first
    long long right = 1LL * n * (n + 1) / 2;  // promoted before multiplying
    cout << wrong << "\n" << right << "\n";
    return 0;
}
```

```text
705082704
5000050000
```

Why `705082704`? $10^5 \cdot 100001 = 10{,}000{,}100{,}000$; modulo $2^{32}$ that is $1{,}410{,}165{,}408$; halved, $705{,}082{,}704$. Every bit of the error is explained by the wrap, and none of it by your algorithm.

**The type of an arithmetic expression is decided by its operands, not by the variable you assign to.** Assigning to `long long` afterwards is too late; the damage happened inside the expression.

| Expression | Result type | Safe for |
|---|---|---|
| `int * int` | `int` | products below $2.1 \times 10^9$ |
| `1LL * a * b` | `long long` | products below $9.2 \times 10^{18}$ |
| `1LL * a * b % m` | `long long` | moduli up to $\approx 3 \times 10^9$ |
| `(__int128)a * b % m` | `__int128` | moduli up to $1.7 \times 10^{38}$ |

Unsigned arithmetic is defined to wrap modulo $2^w$, so `0u - 1u` is reliably `4294967295u`. Signed overflow is not defined at all: `-fsanitize=undefined` reports it, and `-O2` may assume it never happens. Since contest builds use `-O2` without sanitizers, the only safe policy is never to overflow.

## 7. `long long` discipline and the `1LL *` trick

Four habits, applied mechanically, remove most overflow bugs:

1. **Read big values as `long long`.** If values reach $10^9$ and you sum $10^5$ of them, the *sum* needs 64 bits.
2. **Cast before the first multiplication**, not after: `(long long)a * b` or the shorter `1LL * a * b`.
3. **Watch comparisons too.** `a * b > 1e9` is already broken before the comparison runs.
4. **Print with `%lld`** or `cout`. `printf("%d", x)` on a `long long` is UB and prints garbage.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int a = 100000, b = 100000;
    long long bad  = a * b;                    // 10^10 does not fit in int
    long long good = 1LL * a * b;              // correct
    long long pairs = 1LL * a * (a - 1) / 2;   // ~5e9, also needs 64 bits
    cout << bad << " " << good << " " << pairs << "\n";
    return 0;
}
```

```text
1410065408 10000000000 4999950000
```

Order matters for division: `a * b / 2` overflows before dividing, while `a / 2 * b` is exact only when `a` is even. Compute in the widest type and divide last.

## 8. `__int128`: when 64 bits is not enough

`__int128` (GCC/Clang, available on every major judge) holds up to $1.7 \times 10^{38}$. The canonical use is **modular multiplication with a modulus near $10^{18}$**, where $a \cdot b$ reaches $10^{36}$ and `long long` silently wraps.

```cpp
#include <bits/stdc++.h>
using namespace std;

using i128 = __int128_t;

long long mod_mul(long long a, long long b, long long m) {
    return (long long)((i128)a * b % m);
}

string to_string_i128(i128 x) {
    if (x == 0) return "0";
    bool neg = x < 0;
    if (neg) x = -x;
    string s;
    while (x > 0) { s.push_back(char('0' + (int)(x % 10))); x /= 10; }
    if (neg) s.push_back('-');
    reverse(s.begin(), s.end());
    return s;
}

int main() {
    long long m = 1000000000000000003LL;     // ~1e18, prime
    long long a = 999999999999999999LL;
    cout << mod_mul(a, a, m) << "\n";
    cout << to_string_i128((i128)1e18 * 1000000000LL) << "\n";   // 1e27
    return 0;
}
```

| Property | Reality |
|---|---|
| `cin >> x` | **Does not work.** Read into `long long` or a string, then cast |
| `cout << x` | **Does not compile.** Use the helper above |
| `std::to_string` | No overload |
| Arithmetic | `+ - * / %`, comparisons and bitwise all work |
| Speed | ~2–4× slower than 64-bit — fine for $O(\log m)$, not for $10^7$ inner-loop steps |
| Portability | GCC/Clang only; MSVC has none (irrelevant on Codeforces) |

Do not reach for it casually: if the modulus is $10^9+7$, `long long` is enough and twice as fast.

## 9. Floating point: `float`, `double`, `long double`

An IEEE-754 value is $(-1)^s \times 1.m \times 2^{e}$: sign bit, mantissa bits (precision), exponent bits (range).

| Type | Bits | Mantissa bits | Significant digits | Range | Verdict |
|---|---|---|---|---|---|
| `float` | 32 | 24 | ~7 | $\pm 3.4 \times 10^{38}$ | Almost never in CP |
| `double` | 64 | 53 | **15–17** | $\pm 1.8 \times 10^{308}$ | The default |
| `long double` (x86) | 80 used / 16 stored | 64 | ~18–19 | $\pm 1.1 \times 10^{4932}$ | Only if $10^{-12}$ is demanded |
| `long double` (MSVC) | 64 | 53 | 15–17 | same as `double` | Misleading — no gain |

Two hard numbers: **`double` represents every integer exactly up to $2^{53} \approx 9.0 \times 10^{15}$** (beyond that `x + 1 == x` is possible), and it carries about **15 significant decimal digits, not 15 decimal places**.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    double a = 0.1, b = 0.2;
    cout << (a + b == 0.3) << "\n";               // 0 -> false
    cout << setprecision(17) << (a + b) << "\n";  // the full truth
    cout << setprecision(6)  << (a + b) << "\n";  // what cout shows by default
    return 0;
}
```

```text
0
0.30000000000000004
0.3
```

`0.1` in binary is $0.000110011001100\dots_2$ — a repeating fraction that cannot be stored exactly, just as $1/3$ cannot be written in decimal. The two nearest `double`s to 0.1 and 0.2 do not add to the nearest `double` to 0.3. Default 6-digit output hides it; `==` does not. Judges compare reals with a tolerance, and so must you.

## 10. Epsilon comparison, NaN and infinity

```cpp
#include <bits/stdc++.h>
using namespace std;

const double EPS = 1e-9;

bool nearly_equal(double a, double b) {         // relative + absolute
    return fabs(a - b) <= EPS * max(1.0, max(fabs(a), fabs(b)));
}

int main() {
    cout << (0.1 + 0.2 == 0.3) << " " << nearly_equal(0.1 + 0.2, 0.3) << "\n";

    double z = 0.0;
    cout << (z / z) << " " << (1.0 / z) << " " << (-1.0 / z) << "\n";
    cout << (z / z == z / z) << " " << isnan(z / z) << " " << isinf(1.0 / z) << "\n";
    return 0;
}
```

```text
0 1
nan inf -inf
0 1 1
```

| Check | Correct way | Wrong way |
|---|---|---|
| Equality | `fabs(a - b) < 1e-9` | `a == b` |
| Zero | `fabs(a) < 1e-12` | `a == 0` |
| NaN | `isnan(x)` or `x != x` | `x == NAN` (always false) |
| Infinity | `isinf(x)` | `x > 1e308` |

A *relative* epsilon is safer than a fixed absolute one when answers reach $10^9$: at that magnitude a `double` resolves only about $10^{-7}$, so `1e-9` absolute is unachievable. Floating-point division by zero yields `inf`/`nan`; **integer** division by zero crashes.

### 10.1 Binary search on doubles needs fixed iterations

```cpp
double lo = 0, hi = 1e9;
while (hi - lo > 1e-12) {            // BAD: may never terminate
    double mid = (lo + hi) / 2;
    if (feasible(mid)) hi = mid; else lo = mid;
}

for (int it = 0; it < 100; ++it) {   // GOOD: always terminates
    double mid = (lo + hi) / 2;
    if (feasible(mid)) hi = mid; else lo = mid;
}
```

Once `hi - lo` falls below the spacing of `double` at magnitude $10^9$ (about $10^{-7}$), `(lo + hi) / 2` rounds to `lo` or `hi`, the interval stops shrinking, and the loop spins forever — a guaranteed TLE. A hundred halvings shrink the interval by $2^{100} \approx 10^{30}$, far below the type's precision. **Use 60–100 fixed iterations**; use `long double` and 200 if the statement demands $10^{-12}$.

*Contest story: a geometry problem demanding $10^{-9}$ absolute error. The team that wrote `while (hi - lo > 1e-11)` got TLE on the last test; the fix was `for (int it = 0; it < 100; ++it)`. Two lines of diff, one accepted submission.*

## 11. `char`, ASCII and character arithmetic

A `char` is a 1-byte integer carrying a character code. The standard guarantees digits `'0'..'9'` are contiguous and letters are contiguous within each case; every judge uses ASCII.

| Char | Dec | Char | Dec | Char | Dec |
|---|---|---|---|---|---|
| `'\0'` | 0 | `'A'` | 65 | `'a'` | 97 |
| `'\n'` | 10 | `'Z'` | 90 | `'z'` | 122 |
| `' '` | 32 | `'0'` | 48 | `'9'` | 57 |

Derived constants: `'a' - 'A' == 32`, `'0' + 9 == '9'`, `'a' > 'Z'` (97 > 90).

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    char c = '7';
    cout << (c - '0') << " " << (int)c << "\n";        // 7 55

    for (int d = 0; d < 3; ++d) cout << char('0' + d);
    cout << "\n";                                       // 012

    string s = "HeLLo-42";
    for (char &ch : s) if (isalpha((unsigned char)ch)) ch ^= 32;
    cout << s << "\n";                                  // hEllO-42
    return 0;
}
```

| Operation | Idiom | Note |
|---|---|---|
| digit char → int | `c - '0'` | Never `atoi(&c)` — no null terminator |
| int → digit char | `char('0' + d)` | `d` must be 0–9 |
| letter → index | `c - 'a'` / `c - 'A'` | 0–25, perfect for `int[26]` |
| upper → lower | `c + 32`, `tolower(c)` | `tolower` is safe on non-letters |
| toggle case | `c ^ 32` | Only if `isalpha(c)` |
| is letter | `'a' <= c && c <= 'z' \|\| 'A' <= c && c <= 'Z'` | Faster than `isalpha` in hot loops |

Portability trap: **`char` may be signed or unsigned.** On x86 GCC it is signed, so a byte of value 200 becomes `-56`. When you treat `char` as a byte, use `unsigned char`; and always cast to `(unsigned char)` before calling `isalpha`/`toupper`, which is UB on negative inputs.

## 12. `bool` and implicit conversions

`bool` is a 1-byte integer with two values. The rule is blunt: **zero becomes `false`, everything else becomes `true`** — including `-1` and including `0.5`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    bool a = 5, b = -3, c = 0;
    cout << a << b << c << "\n";                  // 110
    cout << (true + true) << "\n";                 // 2   (bool promotes to int)
    cout << (5 > 2) + (3 > 4) << "\n";             // 1   count satisfied conditions
    cout << boolalpha << a << c << "\n";           // truefalse
    return 0;
}
```

```text
110
2
1
truefalse
```

`cout` prints `0`/`1` unless `boolalpha` is set. If the expected output is `YES`/`NO`, print the words yourself: `cout << (ok ? "YES" : "NO")`. Because a comparison yields `0` or `1`, you can *count* satisfied conditions by adding them — a legitimate CP idiom (§17 of Topic 06).

## 13. Type promotion and the `unsigned` bug

The rules, in order:

1. `bool`, `char`, `short` are **promoted to `int`** in arithmetic.
2. Between different types, the smaller converts to the larger.
3. If one is signed and the other unsigned of the *same or larger rank*, **the signed one becomes unsigned**.

Rule 3 is the killer, because `size()` returns `size_t`, i.e. `unsigned long long`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v;                        // empty
    cout << (int)v.size() - 1 << "\n";    // -1
    // cout << v.size() - 1;              // 18446744073709551615  (underflow)

    int i = -1;
    if (i < (int)v.size()) cout << "safe\n";     // false: 0 < 0 is false
    if (i < v.size())      cout << "unsafe\n";   // true: -1 becomes 2^64-1
    return 0;
}
```

```text
-1
```

Cast `size()` to a signed type of the same width, as a reflex:

```cpp
for (int i = 0; i + 1 < (int)v.size(); ++i) { /* compare v[i], v[i+1] */ }
```

That prevents both classic bugs: `i < v.size()` with `int i = -1`, and `v.size() - 1` on an empty container.

| Expression | Type | Value when `v` is empty |
|---|---|---|
| `v.size()` | `size_t` | 0 |
| `v.size() - 1` | `size_t` | 18446744073709551615 |
| `(int)v.size() - 1` | `int` | -1 |
| `v.empty()` | `bool` | `true` |

## 14. `const`, `static`, `auto`

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MOD = 1'000'000'007;   // C++14 digit separators, cosmetic only
const int MAXN = 200005;

int counter() {
    static int calls = 0;        // initialised once, survives across calls
    return ++calls;
}

int main() {
    static int table[MAXN];      // zero-initialised, in BSS, not on the stack
    cout << counter() << " " << counter() << " " << counter() << "\n";  // 1 2 3

    auto x = 5;      // int
    auto y = 5LL;    // long long
    auto z = 2.5;    // double
    cout << sizeof(x) << sizeof(y) << sizeof(z) << "\n";   // 488
    return 0;
}
```

| Keyword | Meaning | CP use |
|---|---|---|
| `const` | Cannot be modified after initialisation | `const int MOD = 1e9+7`, `const double EPS = 1e-9` |
| `static` (local) | One instance for the whole program | Memo tables, call counters |
| `static` (global) | Zero-initialised, internal linkage | Large arrays, avoids stack overflow |
| `auto` | Deduce from the initialiser | Iterators, lambdas, `auto p = make_pair(1, 2)` |

`auto` strips `const` and references, so `auto c = s[0]` copies a `char` and writing through `c` changes nothing — the same bug as the range-for trap in Topic 05. `auto` gets full treatment in Topic 07.

## 15. Bitwise operators: the CP cheat table

Bitwise operators work on individual bits: one CPU instruction, no branch.

| Operator | Name | Example ($a=12$, $b=10$) | Result | CP use |
|---|---|---|---|---|
| `a & b` | AND | `1100 & 1010` | `1000` = 8 | Test/mask flags |
| `a \| b` | OR | `1100 \| 1010` | `1110` = 14 | Combine/set flags |
| `a ^ b` | XOR | `1100 ^ 1010` | `0110` = 6 | Toggle; find the unique element |
| `~a` | NOT | `~1100` | `...0011` = -13 | Clear a bit with `a & ~mask` |
| `a << k` | left shift | `1100 << 1` | `11000` = 24 | Multiply by $2^k$ |
| `a >> k` | right shift | `1100 >> 1` | `0110` = 6 | Divide by $2^k$, floor |

| Idiom | Meaning | Note |
|---|---|---|
| `x & 1` | is `x` odd? | Faster than `x % 2`, correct for negatives |
| `x >> 1` | `x / 2` rounded down | For negatives, `/` truncates toward zero instead |
| `x & -x` | lowest set bit | `12 & -12 == 4`; Fenwick trees |
| `x \| (1 << i)` | set bit `i` | Use `1LL << i` when `i` can reach 31 |
| `x & ~(1 << i)` | clear bit `i` | |
| `x ^ (1 << i)` | toggle bit `i` | |
| `(x >> i) & 1` | read bit `i` | |
| `x & (x - 1)` | clear lowest set bit | |
| `x && !(x & (x-1))` | is `x` a power of two? | |
| `__builtin_popcountll(x)` | set-bit count | Compiles to `POPCNT` |
| `__builtin_ctzll(x)` | index of lowest set bit | UB when `x == 0` |
| `__builtin_clzll(x)` | leading zeros | `63 - clz` is `floor(log2 x)` |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int x = 12;                                                       // 1100
    cout << (x & 1) << " " << (x >> 1) << " " << (x & -x) << "\n";      // 0 6 4
    cout << (x | (1 << 1)) << " " << (x & ~(1 << 3)) << "\n";           // 14 4
    cout << (x ^ (1 << 0)) << " " << (x & (x - 1)) << "\n";             // 13 8

    unsigned long long m = 0xF0F0F0F0ULL;
    cout << __builtin_popcountll(m) << " " << __builtin_ctzll(m) << "\n";  // 16 4
    return 0;
}
```

Two traps worth memorising:

```cpp
int a = 1 << 31;         // UB: 1 is int and 2^31 does not fit. Write 1LL << 31.
int b = 1 << 32;         // UB, often 1, because the shift count wraps.
long long c = 1LL << 62; // largest safe single-bit mask for long long.
```

Also: `~0` is `-1` in `int` but `4294967295` in `unsigned`, and left-shifting a negative `x` is UB before C++20 — shift unsigned values when in doubt.

## 16. `std::bitset`: fixed-width bit work

`std::bitset<N>` is a compile-time-sized bit array with $O(N/64)$ bulk operations. For $N \le 64$ it is essentially a free register.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    bitset<10> b;
    b[3] = 1; b[7] = 1;
    cout << b << "\n";              // 0010001000  (index 9 printed first)
    cout << b.count() << " " << b.to_ullong() << " " << b.test(3) << "\n";  // 2 136 1
    b.flip();
    cout << b.count() << "\n";      // 8

    bitset<8> c(0b1010'1010);
    cout << c << " " << c.to_ulong() << "\n";   // 10101010 170
    return 0;
}
```

| Use case | How |
|---|---|
| Sieve of Eratosthenes | `bitset<100000001> is_prime;` — 12.5 MB, 8× smaller than `bool[]` |
| Subset DP for $n \le 20$ | `for (int mask = 0; mask < (1 << n); ++mask)` |
| Set of small integers | `b.set(x); b.test(x); b.count();` |
| Bulk shift | `b <<= 1` moves every bit in one pass |
| Printing binary | `cout << b` |

The size must be a compile-time constant. For runtime sizes use `vector<bool>` (packed but slow, proxy references) or `vector<uint64_t>` with manual masks.

## 17. `numeric_limits` vs `INT_MAX`, and literal suffixes

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << INT_MAX << " " << numeric_limits<int>::max() << "\n";        // same
    cout << LLONG_MAX << " " << numeric_limits<long long>::max() << "\n";
    return 0;
}
```

| Expression | Value | Header | Notes |
|---|---|---|---|
| `INT_MAX` | 2147483647 | `<climits>` | Macro, type `int` |
| `INT_MIN` | -2147483648 | `<climits>` | `-2147483648` typed literally is unary minus on an out-of-range value |
| `LLONG_MAX` | 9223372036854775807 | `<climits>` | |
| `numeric_limits<int>::max()` | 2147483647 | `<limits>` | Typed, `constexpr`, template-friendly |
| `0x3f3f3f3f` | 1061109567 | — | "safe infinity": adding two stays in `int` |

Prefer `numeric_limits<T>::max()` in templates, `INT_MAX`/`LLONG_MAX` in contest code. What you must not do is mix `int` infinity with `long long` distances — that is how a Dijkstra reports `2139062143` as a distance.

| Literal | Type | Use |
|---|---|---|
| `42` | `int` | default |
| `42LL` | `long long` | **the CP workhorse** |
| `42ULL` | `unsigned long long` | hashing |
| `42.0` | `double` | default for reals |
| `42.0L` | `long double` | 18 digits |
| `1e9` | **`double`** | trap: `int x = 1e9;` works, `x % 1e9` does not compile |
| `1'000'000'007` | `int` | C++14 separators, readability only |
| `0x1F`, `0b1010`, `017` | int, int, octal 15 | bases |

`1e9` is a *floating* literal. It happens to be exact, but `1e9 * x` promotes the whole expression to `double` and can lose precision for large `x`. Write integer literals for integer quantities; `1e9` is acceptable as a bound, never as an operand.

---

## Key Takeaways

- The type of an expression comes from its **operands**, not from the variable you assign it to.
- `int` holds 10 decimal digits and dies at $2^{31}-1$; anything with $n^2$ for $n > 46{,}340$ needs `long long`.
- `1LL *` at the front of a multiplication is the cheapest insurance in competitive programming.
- Signed overflow is **UB**; unsigned overflow is defined wraparound. Never rely on either.
- `__int128` exists for products near $10^{36}$; it cannot be read or printed directly.
- `double` has ~15–17 significant digits and is exact for integers only up to $2^{53} \approx 9 \times 10^{15}$.
- Never compare floats with `==`; use a relative epsilon, `fabs(a-b) <= EPS * max(1.0, max(fabs(a), fabs(b)))`.
- Binary search on doubles uses a **fixed iteration count** (60–100), never `while (hi - lo > eps)`.
- `char` is an integer: `c - '0'` converts digits, `c ^ 32` toggles case for letters, `'a' - 'A' == 32`.
- `size()` is unsigned; cast it to `int` before subtracting or comparing with signed values.
- `1 << 31` is UB — write `1LL << i` when `i` can reach 31.
- `1e9` is a `double`; use `1000000000LL` when you mean an integer.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `long long s = n * (n + 1) / 2;` with `int n` | Product computed in `int` first | `1LL * n * (n + 1) / 2` |
| 2 | Using `long` for 64-bit values | 32-bit on Windows, 64-bit on Linux | Always `long long` |
| 3 | `abs(INT_MIN)`, or negating `INT_MIN` | No positive counterpart exists | Work in `long long` |
| 4 | `if (a + b > c)` near `INT_MAX` | Overflow before comparison | `if (1LL*a + b > c)` |
| 5 | `v.size() - 1` on an empty vector | Unsigned underflow to $2^{64}-1$ | `(int)v.size() - 1`, or check `empty()` |
| 6 | `int i = -1; i < v.size()` | `-1` converts to a huge unsigned | Cast size to `int` |
| 7 | `while (hi - lo > 1e-12)` in double binary search | Interval stops shrinking, infinite loop | Fixed `for` loop, 100 iterations |
| 8 | `if (0.1 + 0.2 == 0.3)` | Binary fractions are inexact | Epsilon comparison |
| 9 | Expecting `(double)1e18 + 1` to change | Only 53 mantissa bits | Keep integers in integer types |
| 10 | `int x = 1 << 31;` | UB, often `INT_MIN` or `0` | `1LL << 31` |
| 11 | `printf("%d", someLongLong)` | Format mismatches type | `%lld` or `cout` |
| 12 | `bitset<64>(x).count()` in a hot loop | Much slower than the intrinsic | `__builtin_popcountll(x)` |
| 13 | `int a[1000005]` inside `main` | 4 MB on the stack, plus recursion | Make it global/`static` |
| 14 | Hashing bytes as `char` | Signed `char` goes negative past 127 | Cast to `unsigned char` |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Print a number in binary/hex without library conversion | Any judge, warm-up | Easy |
| 2 | Sum of $1..n$ for $n \le 10^9$ (answer needs 64 bits) | CSES introductory | Easy |
| 3 | Count pairs $\binom{n}{2}$ for $n \le 2 \times 10^5$ | Codeforces Div. 2 A/B | Easy |
| 4 | Detect overflow in `a * b` using a division check | Interview classic | Medium |
| 5 | Modular exponentiation with $m$ up to $10^{18}$ | CSES "Exponentiation II" variant | Medium |
| 6 | Binary search on a real-valued answer to $10^{-9}$ | Codeforces Div. 2 C | Medium |
| 7 | Sum of distances on a line with coordinates up to $10^9$ | Codeforces Div. 2 B | Medium |
| 8 | Subset enumeration over $n \le 20$ with popcount filtering | AtCoder ABC (bit DP intro) | Medium |
| 9 | Count set bits for every $x \in [0, n]$, $n \le 10^6$ | Classic bit DP | Medium |
| 10 | Lowest-set-bit / Fenwick tree construction | CSES "Dynamic Range Sum" | Hard |

## Next Up

→ [Topic 05 — String Manipulation Methods & Statements](Topic%2005%20-%20String%20Manipulation%20Methods%20%26%20Statements.md)