# Topic 08 — Useful Math Functions & Operations on Data

C++ gives you a maths library that is designed for scientists, not for contestants: it works in `double`, it rounds when you are not looking, and its integer helpers have overloads that quietly pick the wrong type. A contestant needs three things from it — exactness at the boundaries, overflow safety, and a rough idea of cost. This chapter covers the functions you will call in every second problem, the traps each one carries, and the modular-arithmetic toolkit that every counting problem depends on.

> **Prerequisites:** [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems,%20Variables%20and%20Datatypes.md), [Topic 07 — Loops & 'Auto'](Topic%2007%20-%20Loops%20&%20'Auto'.md)
> **You will be able to:**
> - Call `abs`, `sqrt`, `pow`, `min`/`max` without silent truncation or float error.
> - Predict and fix C++'s modulo behaviour on negative numbers.
> - Compute `gcd`/`lcm` without overflow, and use the bit builtins in $\mathcal{O}(1)$.
> - Do modular addition, subtraction and multiplication without overflow, including the `__int128` escape hatch.
> - Reach for the nine standard CP recipes (digit sum, prime test, divisors, factorial mod, nCr mod, …) from memory.

---

## 1. The mental model: `<cmath>` speaks `double`

Almost every function in `<cmath>` takes and returns `double` (or `float`/`long double` overloads). `double` has 53 bits of mantissa, so it represents every integer exactly only up to $2^{53} \approx 9 \times 10^{15}$. Above that, integers get rounded to the nearest representable value — silently.

| You want | Exact type | `<cmath>` version | Risk |
|---|---|---|---|
| absolute value of `int` | `int` | `abs(int)` | fine |
| absolute value of `long long` | `long long` | `llabs` / `std::abs` | wrong overload → truncation |
| $\sqrt{n}$ as an integer | `long long` | `sqrt(double)` | off-by-one from rounding |
| $a^b$ for integers | `long long` | `pow(double,double)` | 1 ulp error → wrong integer |
| $\lfloor a/b \rfloor$ | `long long` | `floor(a/b)` | `a/b` already truncated first |
| digit count | `int` | `log10(n)` | exact powers of 10 |

*Contest story: a Div. 2 C asked for $\lfloor \sqrt{n} \rfloor$ for $n \le 10^{18}$. `(long long)sqrt(n)` passed every sample and failed system test 7, because `sqrt(10^18 - 1)` rounds up to exactly $10^9$. One line of adjustment code would have saved the submission.*

---

## 2. `abs` — the overload that can truncate your `long long`

`abs` is not one function; it is a family. In C++17 with the right headers you get `abs(int)`, `abs(long)`, `abs(long long)`, `abs(float)`, `abs(double)`, `abs(long double)`. The danger is the C legacy: `<cstdlib>` also declares `int abs(int)`, and if the `long long` overload is not visible (missing header, older toolchain, or an unlucky order of includes), a `long long` argument is **converted to `int`** and the answer is garbage for anything above $2^{31}-1$.

```cpp
int main() {
    long long x = -3000000000LL;   // |x| = 3e9, does not fit in int
    cout << llabs(x) << '\n';      // guaranteed: long long overload
    cout << std::abs(x) << '\n';   // qualified: picks abs(long long)
    return 0;
}
```

```text
3000000000
3000000000
```

Rules that keep you safe:

1. **Include `<cstdlib>` (or `<bits/stdc++.h>`) and prefer `std::abs`,** or use `llabs` for `long long`. Never rely on unqualified `abs` finding the 64-bit overload.
2. **`abs(INT_MIN)` is undefined behaviour** — the true value $2^{31}$ is not representable in `int`. If a problem can hand you `INT_MIN`, widen first: `llabs((long long)x)`.
3. **`abs` on `unsigned` is meaningless.** It compiles (via conversion) and returns the same huge value. If you see `abs` near an unsigned variable, you have a bug.
4. **For `double`, use `fabs`** (or `std::abs`, which is overloaded). `abs(3.7)` may resolve to the integer version and return `3`.

| Call | Argument type | Correct? | Use instead |
|---|---|---|---|
| `abs(x)` | `int` | yes | — |
| `abs(x)` | `long long`, small value | accidentally yes | `llabs(x)` |
| `abs(x)` | `long long`, value $> 2^{31}$ | **no** — truncates | `llabs(x)`, `std::abs(x)` |
| `abs(x)` | `double` | maybe (int overload) | `fabs(x)` |
| `abs(INT_MIN)` | `int` | **UB** | `llabs((long long)x)` |
| `abs(x)` | `unsigned` | no-op nonsense | rethink the code |

---

## 3. `min` and `max`: exactly two arguments, same type

```cpp
int a = 3, b = 7;
cout << min(a, b) << ' ' << max(a, b) << '\n';      // 3 7
cout << min({5, 2, 9, 1}) << '\n';                  // 1 — initializer-list form (C++11)
cout << max({5, 2, 9, 1}) << '\n';                  // 9
```

Three facts cause most of the pain:

- **The two-argument form takes exactly two arguments.** `min(a, b, c)` does not compile. Use `min({a, b, c})`, or nest: `min(a, min(b, c))`.
- **Both arguments must have the same type.** `min(1, 2LL)` fails to compile — deliberately, because the compiler does not know which type you want back. Fix by making the types match: `min<long long>(1, 2LL)` or `min(1LL, 2LL)`.
- **The result is a reference.** `min` returns `const T&`, so binding it to a reference that outlives the arguments is a dangling reference:

```cpp
const int &r = min({a, b, c});   // DANGLING: refers into a temporary array
int m = min({a, b, c});          // correct: a copy
```

`min`/`max` compile to a compare and a conditional move — $\mathcal{O}(1)$, no cost worth worrying about. The place where they *do* cost is inside a loop over a container of heavy objects: `max(a, b)` on `std::string` compares contents, which is $\mathcal{O}(\text{length})$.

---

## 4. `swap`

```cpp
int a = 1, b = 2;
swap(a, b);                       // a=2, b=1 — <utility>, O(1)
vector<int> x(1000000), y(1000000);
swap(x, y);                       // O(1): swaps three pointers, not a million ints
int arr1[3] = {1,2,3}, arr2[3] = {4,5,6};
swap(arr1, arr2);                 // O(n): element-wise for C arrays
```

`std::swap` is a three-move exchange for scalars and a pointer exchange for containers — that is why `swap(v[i], v[j])` inside a reverse loop is free. For C arrays there is no pointer to swap, so it degrades to element-wise copying; if you need to swap large buffers, hold them in a `vector` or swap pointers.

---

## 5. `sqrt`, and how to get an exact integer square root

`sqrt(double)` is correctly rounded for the *double* result, which is not the same as giving you the exact floor of the integer square root. Two failure modes:

1. **Rounding up.** `sqrt(10^18 - 1)` may return exactly `1e9`, and `(long long)` of that is `1000000000` instead of `999999999`.
2. **Large integers.** Above $2^{53}$, the argument itself is already rounded, so no cast can recover the exact answer.

The fix is always the same: take the float guess, then **adjust with integer arithmetic** until it is right.

```cpp
#include <bits/stdc++.h>
using namespace std;

long long isqrt(long long n) {              // floor(sqrt(n)) exactly, n >= 0
    if (n < 0) return -1;                   // caller's bug
    long long r = (long long)sqrtl((long double)n);
    while ((r + 1) * (r + 1) <= n) ++r;     // may be one too small
    while (r * r > n) --r;                  // may be one too big
    return r;
}

int main() {
    cout << isqrt(0) << '\n';                       // 0
    cout << isqrt(999999999999999999LL) << '\n';    // 999999999
    cout << isqrt(1000000000000000000LL) << '\n';   // 1000000000
    return 0;
}
```

```text
0
999999999
1000000000
```

Why the loops are safe: after the float guess, the error is at most 1 for any `long long` input, so each `while` runs at most once or twice. `sqrtl` (long double, 64-bit mantissa on x86) is more accurate than `sqrt` and costs nothing measurable.

Use the same pattern for cube roots and for any "is $n$ a perfect square" test: `r = isqrt(n); return r*r == n;`.

### 5.1 `cbrt`, `hypot`, `exp`

```cpp
cout << cbrt(27.0) << '\n';                 // 3
cout << hypot(3.0, 4.0) << '\n';            // 5  — sqrt(x²+y²) without intermediate overflow
cout << exp(1.0) << '\n';                   // 2.71828...
```

`hypot` is the safe way to compute a distance: `sqrt(x*x + y*y)` overflows to `inf` when `x` or `y` exceeds about $10^{154}$, while `hypot` scales internally and does not. `cbrt` handles negative arguments correctly (`cbrt(-8) == -2`), unlike `pow(-8, 1.0/3)`, which returns `NaN` because `1.0/3` is not exactly one third.

---

## 6. `pow`: why `pow(10, 9)` can be `999999999`

`pow(x, y)` computes $x^y$ in `double`. For integer exponents and integer bases, the result is *usually* exact for small values and *sometimes* one ulp low, which truncates to the wrong integer.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << fixed << setprecision(3);

    double p = pow(10.0, 9.0);
    cout << p << '\n';                      // 1000000000.000 (usually)
    cout << (long long)p << '\n';           // 1000000000

    // The dangerous shape: a result computed in double, truncated to long long
    cout << (long long)pow(9.0, 9.0) << '\n';   // 387420489 — fine here
    cout << (long long)pow(2.0, 53.0) << '\n';  // 9007199254740992 — last exact power of 2

    // Negative base, fractional exponent → NaN
    cout << pow(-8.0, 1.0 / 3.0) << '\n';       // nan
    return 0;
}
```

```text
1000000000.000
1000000000
387420489
9007199254740992
nan
```

The rule: **never use `pow` to compute an integer power that must be exact.** Write a loop for small exponents, or fast exponentiation for large ones, or precompute a table.

```cpp
long long ipow(long long b, int e) {        // exact for e >= 0
    long long r = 1;
    while (e--) {
        if (__builtin_mul_overflow(r, b, &r)) return -1;   // overflow guard
    }
    return r;
}
```

`__builtin_mul_overflow` (GCC/Clang) sets `r` and returns `true` on overflow — a clean way to detect the $b^e > 9.2 \times 10^{18}$ case instead of silently wrapping. Note the four places where `pow` is genuinely fine: computing a non-integer power for geometry, `pow(x, 0.5)` when you already intend `sqrt`, and constants you will not truncate.

---

## 7. `log`, `log2`, `log10`, and the digit-counting trap

The textbook formula for the number of decimal digits of $n$ is

$$d(n) = \lfloor \log_{10} n \rfloor + 1.$$

It is correct in mathematics and wrong in floating point whenever `log10` rounds across an integer boundary. The reproducible case is a long run of nines, where the true log is `14.999999…` and `double` stores `15.0`:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    for (long long n : {999999999999999LL, 1000000000000000LL, 9999999999999999LL}) {
        int bad  = (int)floor(log10((double)n)) + 1;
        int good = (int)to_string(n).size();
        cout << n << "  log10-based=" << bad << "  correct=" << good << '\n';
    }
    return 0;
}
```

```text
999999999999999  log10-based=16  correct=15
1000000000000000  log10-based=16  correct=16
9999999999999999  log10-based=17  correct=16
```

Exact powers of ten fail the same way on many libm builds: `log10(100.0)` can return `1.9999999999999998`, so `floor` gives 1 and you report 2 digits for a 3-digit number. Two safe alternatives:

```cpp
int digits = (int)to_string(n).size();                    // simple, allocates a string
int digits2 = (int)(log10((double)n) + 1e-9) + 1;         // epsilon fix, safe up to 2^53
```

The epsilon must be smaller than the smallest possible gap to the next integer, which is why `1e-9` works for $n \le 10^{15}$ but not for `double` arguments near $10^{17}$.

For binary, `log2(n)` is *exact* for powers of two up to $2^{53}$ because the result is an integer representable in `double` — but `__builtin_clz` is faster and exact anyway:

```cpp
int bits = 64 - __builtin_clzll(n);        // number of bits, n > 0
int floorLog2 = 63 - __builtin_clzll(n);   // floor(log2(n)), n > 0
```

Both are undefined for `n == 0`; guard it.

---

## 8. `floor`, `ceil`, `round`, `trunc` — and the integer division you already did

| Call | `2.5` | `-2.5` | `2.4` | `-2.4` | Returns |
|---|---|---|---|---|---|
| `floor(x)` | 2 | **-3** | 2 | -3 | largest integer $\le x$ |
| `ceil(x)` | 3 | -2 | 3 | -2 | smallest integer $\ge x$ |
| `trunc(x)` | 2 | -2 | 2 | -2 | drop the fraction (toward zero) |
| `round(x)` | 3 | -3 | 2 | -2 | nearest, halfway **away from zero** |
| `(int)x` | 2 | -2 | 2 | -2 | same as `trunc` for in-range values |

Two CP consequences:

- **`round` is not "round half to even".** `round(2.5)` is 3 and `round(-2.5)` is -3, so it is not symmetric in the way banker's rounding would be. `rint`/`nearbyint` use the current rounding mode (usually half-to-even) — avoid them in solutions.
- **`floor(a / b)` is a no-op if `a` and `b` are integers.** `a / b` already truncated toward zero *before* `floor` saw it. For a true floor division you must correct for negative operands:

```cpp
long long floor_div(long long a, long long b) {   // b != 0
    long long q = a / b, r = a % b;
    if (r != 0 && ((r < 0) != (b < 0))) --q;
    return q;
}
```

Ceiling division, used constantly in CP (`ceil(a/b)` for positive integers):

```cpp
long long ceil_div(long long a, long long b) { return (a + b - 1) / b; }   // a, b > 0
```

Careful with `a + b - 1`: it overflows when `a` is near `LLONG_MAX`. The safe form is `a / b + (a % b != 0)`.

---

## 9. Integer division and modulo with negatives

Since C++11 the rule is fixed: **division truncates toward zero, and the remainder takes the sign of the dividend.**

$$(-7) / 3 = -2, \qquad (-7) \bmod 3 = -1.$$

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << (-7) / 3 << ' ' << (-7) % 3 << '\n';    // -2 -1
    cout << 7 / (-3) << ' ' << 7 % (-3) << '\n';    // -2 1
    cout << (-7) / (-3) << ' ' << (-7) % (-3) << '\n';  // 2 -1

    // the fix that makes a remainder non-negative
    int m = 3;
    cout << (((-7) % m) + m) % m << '\n';           // 2
    return 0;
}
```

```text
-2 -1
-2 1
2 -1
2
```

This matters the moment you index an array with a modulo: `a[(i - k) % n]` with negative `i - k` gives a negative index, and a negative index is undefined behaviour, not a wrap-around. Always write the normalising form:

```cpp
int mod(long long x, int m) { return (int)((x % m + m) % m); }
```

Why `(x % m + m) % m` and not `x % m + m`: if `x % m` is 0, the second form returns `m`, which is out of range. The double modulo is not redundant.

Python and mathematics use *floored* modulo (`-7 % 3 == 2`); C++ uses *truncated* modulo. Any formula you port from Python, or any closed-form you derive on paper, needs this correction. This single difference is responsible for a large share of "my formula works on paper" WAs.

---

## 10. `gcd` and `lcm`

Since C++17, `<numeric>` gives you both:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << gcd(12, 18) << '\n';            // 6
    cout << gcd(0, 5) << '\n';              // 5   (gcd(0, n) = n)
    cout << gcd(0, 0) << '\n';              // 0   (defined, not UB)
    cout << lcm(4, 6) << '\n';              // 12
    cout << __gcd(12, 18) << '\n';          // 6   (GCC builtin, non-standard)
    return 0;
}
```

The manual Euclidean algorithm, which you should be able to write from memory because it generalises to `__int128`, polynomials, and extended-gcd problems:

```cpp
long long gcd_ll(long long a, long long b) {
    a = llabs(a); b = llabs(b);
    while (b) { long long t = a % b; a = b; b = t; }
    return a;
}
```

$\mathcal{O}(\log \min(a,b))$ — at most about 90 iterations for 64-bit inputs (Fibonacci numbers are the worst case).

**`lcm` overflow is the classic trap.** `lcm(a, b) = a * b / gcd(a, b)`, and `a * b` overflows long before the answer does: for $a = b = 10^9$, the product is $10^{18}$ (fits), but for $a = b = 10^{10}$ it does not. Divide **before** multiplying:

```cpp
long long lcm_ll(long long a, long long b) {
    if (a == 0 || b == 0) return 0;
    return a / gcd_ll(a, b) * b;        // divide first: exact, no intermediate overflow
}
```

`std::lcm` is specified to have undefined behaviour if the result is not representable — it will not protect you. In a "lcm of an array modulo $m$" problem, compute the lcm modulo $m$ with the same divide-first order, or the answer overflows at the third element.

---

## 11. Bit builtins: `popcount`, `clz`, `ctz`, `parity`

These compile to single instructions on modern CPUs and are $\mathcal{O}(1)$. All of them are GCC/Clang extensions, available on every judge that uses GCC.

| Builtin | Meaning | Undefined for |
|---|---|---|
| `__builtin_popcount(x)` | number of 1 bits in `unsigned int` | — (0 → 0) |
| `__builtin_popcountll(x)` | same, `unsigned long long` | — |
| `__builtin_clz(x)` | leading zeros in 32-bit | `x == 0` |
| `__builtin_clzll(x)` | leading zeros in 64-bit | `x == 0` |
| `__builtin_ctz(x)` | trailing zeros = index of lowest set bit | `x == 0` |
| `__builtin_parity(x)` | popcount mod 2 | — |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    unsigned x = 0b10110100u;              // 180
    cout << __builtin_popcount(x) << '\n'; // 4
    cout << __builtin_ctz(x) << '\n';      // 2  (lowest set bit is bit 2)
    cout << __builtin_clz(x) << '\n';      // 24 (32 - 8 significant bits)

    cout << __builtin_popcountll(1ULL << 40) << '\n';   // 1
    return 0;
}
```

```text
4
2
24
1
```

The idiom that appears in every bitmask DP: `for (int s = mask; s; s = (s - 1) & mask)` enumerates submasks, and `x & -x` isolates the lowest set bit (equal to `1 << ctz(x)`). Both are worth knowing cold — [Topic 12](Topic%2012%20-%20STL%20Part%201%20%28Vectors,%20DeQue,%20Lists%20and%20Forward%20Lists%29.md) covers `bitset` if you need a container of bits rather than integer tricks.

---

## 12. Integer limits, and the `1e18` rule

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << INT_MAX << '\n';        // 2147483647
    cout << INT_MIN << '\n';        // -2147483648
    cout << LLONG_MAX << '\n';      // 9223372036854775807
    cout << LLONG_MIN << '\n';      // -9223372036854775808
    cout << (int)numeric_limits<int>::max() << '\n';       // same as INT_MAX
    cout << numeric_limits<long long>::lowest() << '\n';   // most negative
    cout << numeric_limits<double>::min() << '\n';         // 2.2e-308 — smallest POSITIVE
    return 0;
}
```

Three things to take away:

1. **`INT_MIN` is not `-INT_MAX`.** It is one smaller, which is why `-INT_MIN` overflows. Never write the literal `-2147483648` in code: the compiler parses it as unary minus applied to `2147483648`, which does not fit in `int`.
2. **`numeric_limits<double>::min()` is the smallest positive normal, not the most negative.** Use `lowest()` for the minimum. Getting this wrong in a "set the answer to the smallest possible" line yields a *positive* initial value and a wrong answer whenever all candidates are negative.
3. **Use `1e18`, not `LLONG_MAX`, as infinity.** The whole point of an infinity constant is that `INF + INF` and `INF + x` do not overflow. `LLONG_MAX + 1` wraps to a huge negative number and turns `min(ans, dist + w)` into nonsense.

```cpp
const long long INF = (long long)1e18;    // safe to add twice: 2e18 < 9.2e18
// const long long INF = LLONG_MAX;       // INF + 1 overflows → negative
```

If you truly need "bigger than any possible answer" for sums of up to $10^5$ terms each $10^9$, then $10^{18}$ is enough ($10^{14}$ max) and doubling it is still safe.

---

## 13. Modular arithmetic: the four operations

Most counting problems ask for the answer modulo $10^9+7$ (or $998244353$). The operators are easy; the *overflows* are not.

```cpp
const int MOD = 1000000007;

int addmod(int a, int b) { return (a + b) % MOD; }        // a,b < MOD → sum < 2^31, safe
int submod(int a, int b) { return (a - b + MOD) % MOD; }  // +MOD keeps it non-negative
int mulmod(long long a, long long b) { return (int)(a * b % MOD); }  // product < 1e18, safe
```

Why these are safe: with `a, b < 10^9+7`, `a + b < 2^{31}` fits in `int`, and `a * b < 10^{18}` fits in `long long`. Two rules follow:

- **Always widen before multiplying.** `int * int` overflows at $2^{31}$; cast one operand to `long long` (or declare the parameter as `long long` as above).
- **`(a - b) % MOD` can be negative** when `a < b`. Add `MOD` before the modulo, as in `submod`. For a chain of subtractions, add `MOD` after each step or accumulate in `long long` and normalise once at the end.

When the modulus itself is up to $10^{18}$, `a * b` overflows `long long` even though each factor fits. That is what `__int128` is for:

```cpp
long long mulmod_big(long long a, long long b, long long m) {
    return (long long)((__int128)a * b % m);
}
```

`__int128` is a GCC/Clang extension with 128-bit range — enough for the product of any two 64-bit values. It is not standard C++, so MSVC does not have it, but every competitive judge does.

### 13.1 Binary exponentiation — a teaser

Computing $b^e \bmod m$ by repeated multiplication is $\mathcal{O}(e)$ and dies at $e = 10^9$. Squaring instead of multiplying gives $\mathcal{O}(\log e)$ in four lines, and every modular-inverse, matrix-power, and hashing routine in later chapters is built on it:

```cpp
long long power(long long b, long long e, long long m) {
    long long r = 1;
    b %= m;
    while (e > 0) {
        if (e & 1) r = r * b % m;
        b = b * b % m;
        e >>= 1;
    }
    return r;
}
```

The full treatment — including why `power(a, m-2, m)` is the modular inverse when `m` is prime — belongs to the number-theory chapter. Here you only need to recognise the shape and trust the complexity: 30 iterations for $e = 10^9$.

---

## 14. Comparing floating point, and printing it

Equality on `double` is almost never what you want:

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    double a = 0.1 + 0.2;
    double b = 0.3;
    cout << (a == b) << '\n';            // 0 — they differ by ~5.5e-17
    cout << (fabs(a - b) < 1e-9) << '\n';  // 1

    cout << fixed << setprecision(6) << a << '\n';   // 0.300000
    cout << setprecision(3) << 1234.5678 << '\n';    // 1234.568  (fixed still active)
    cout << defaultfloat << setprecision(3) << 0.000123456789 << '\n';  // 0.000123
    return 0;
}
```

```text
0
1
0.300000
1234.568
0.000123
```

- **Relative epsilon for large numbers.** `fabs(a - b) < 1e-9` is wrong when `a` and `b` are around $10^{12}$: their difference is dominated by representation error of order $10^{-4}$. Use `fabs(a - b) <= 1e-9 * max(1.0, max(fabs(a), fabs(b)))`, or compare with the judge's tolerance in mind.
- **`setprecision(n)` without `fixed` means *significant* digits** — `0.000123456789` printed as `0.000123`, three significant digits, not three decimals. With `fixed`, it means digits after the decimal point, which is what geometry problems expect. Once you have set `fixed`, it stays set for the whole stream: use `defaultfloat` to go back.
- **Never print more digits than asked.** "Absolute or relative error $10^{-6}$" means six decimals is enough; printing 15 is not a problem for the judge but printing 3 is.
- **`long double` buys you ~64 bits of mantissa** on x86. It is worth using for `sqrt` and for sums of $10^6$ doubles (accumulating in `long double` reduces error), but it does not make `==` safe.

---

## 15. Angles and trigonometry, briefly

`M_PI` is **not** part of standard C++ — it is a POSIX/GNU extension that happens to exist on judges. To be portable:

```cpp
const double PI = acos(-1.0);        // 3.141592653589793
const double DEG = PI / 180.0;

double rad = 60.0 * DEG;             // degrees → radians
double deg = rad / DEG;              // radians → degrees
```

Every `<cmath>` trig function takes radians. Feeding degrees to `sin(30)` gives `-0.988` instead of `0.5` — a bug that produces plausible-looking wrong answers everywhere. `atan2(y, x)` is the correct way to get the angle of a vector (it handles all four quadrants and `x == 0`); `atan(y/x)` does not.

---

## 16. The math cheat table

| Function | Header | Returns | Cost | Main gotcha |
|---|---|---|---|---|
| `abs` | `<cstdlib>` / `<cmath>` | same integer type | $\mathcal{O}(1)$ | truncates if only `abs(int)` visible; `abs(INT_MIN)` is UB |
| `llabs` | `<cstdlib>` | `long long` | $\mathcal{O}(1)$ | — |
| `fabs` | `<cmath>` | `double` | $\mathcal{O}(1)$ | returns double, not int |
| `min`/`max` | `<algorithm>` | `const T&` | $\mathcal{O}(1)$ | two args only; same type; reference dangles on temporaries |
| `swap` | `<utility>` | void | $\mathcal{O}(1)$ / $\mathcal{O}(n)$ | element-wise for C arrays |
| `sqrt` | `<cmath>` | `double` | ~20–50 cycles | not exact for large integers; adjust with a loop |
| `cbrt` | `<cmath>` | `double` | ~50 cycles | exact for small perfect cubes only |
| `pow` | `<cmath>` | `double` | ~50–200 cycles | can be 1 ulp low → wrong integer after cast |
| `log`, `log2`, `log10` | `<cmath>` | `double` | ~50 cycles | exact powers of 10 give `x.9999999` |
| `exp` | `<cmath>` | `double` | ~50 cycles | overflows to `inf` near 710 |
| `hypot` | `<cmath>` | `double` | ~2× sqrt | slower than `sqrt(x*x+y*y)`, but overflow-safe |
| `floor`/`ceil`/`round`/`trunc` | `<cmath>` | `double` | ~5 cycles | `round` is half-away-from-zero; `floor(a/b)` does nothing for ints |
| `gcd`/`lcm` | `<numeric>` (C++17) | integer | $\mathcal{O}(\log \min)$ | `lcm` overflows without divide-first |
| `__gcd` | GCC builtin | integer | $\mathcal{O}(\log \min)$ | non-standard; `__gcd(0,0) == 0` |
| `__builtin_popcount` | GCC | `int` | 1 instruction | use `ll` version for 64-bit |
| `__builtin_clz`/`ctz` | GCC | `int` | 1 instruction | **UB for 0** |
| `accumulate` | `<numeric>` | `T` | $\mathcal{O}(n)$ | initial value's type decides the sum type — see [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md) |

---

## 17. CP recipes

Each recipe below is complete and compiles as shown. They are the ones you will retype most often.

### 17.1 Digit sum, and the digital-root shortcut

```cpp
long long digitSum(long long n) {
    n = llabs(n);
    long long s = 0;
    while (n > 0) { s += n % 10; n /= 10; }
    return s;
}
```

Repeated digit sums collapse to the digital root: for $n > 0$,

$$\text{dr}(n) = 1 + (n-1) \bmod 9.$$

```cpp
int digitalRoot(long long n) { return n == 0 ? 0 : (int)(1 + (n - 1) % 9); }
```

$\mathcal{O}(1)$ instead of $\mathcal{O}(\log n)$ iterations, and it is the intended solution in problems with $n$ up to $10^{18}$ and $10^5$ queries.

### 17.2 Count digits without a loop

```cpp
int countDigits(long long n) {
    if (n == 0) return 1;
    n = llabs(n);
    return (int)to_string(n).size();          // exact, no float error
}
```

The `log10` version is faster but needs the epsilon correction from §7. `to_string` allocates, so in a loop over $10^6$ numbers prefer `(int)(log10((double)n) + 1e-9) + 1`.

### 17.3 Reverse a number

```cpp
long long reverseNumber(long long n) {
    long long sign = (n < 0) ? -1 : 1;
    n = llabs(n);
    long long r = 0;
    while (n > 0) { r = r * 10 + n % 10; n /= 10; }
    return sign * r;
}
```

Overflow: reversing `1234567890123456789` exceeds `long long`. Problems that care ask you to return 0 on overflow (the LeetCode 32-bit variant); check `r > (LLONG_MAX - n % 10) / 10` before multiplying if the statement mentions it.

### 17.4 Primality by trial division

```cpp
bool isPrime(long long n) {
    if (n < 2) return false;
    if (n % 2 == 0) return n == 2;
    if (n % 3 == 0) return n == 3;
    for (long long i = 5; i * i <= n; i += 6) {     // 6k ± 1 candidates only
        if (n % i == 0 || n % (i + 2) == 0) return false;
    }
    return true;
}
```

$\mathcal{O}(\sqrt n)$: about $3 \times 10^4$ iterations at $n = 10^{10}$, instant. The `i * i <= n` condition must not be written `i <= sqrt(n)` (float error) and must not overflow: for `n` near `LLONG_MAX`, `i * i` overflows. Use `i <= n / i` when that is a risk. Above $10^{12}$ you need Miller–Rabin, which is a later chapter.

### 17.5 Count divisors

```cpp
long long countDivisors(long long n) {
    long long cnt = 0;
    for (long long i = 1; i * i <= n; i++) {
        if (n % i == 0) cnt += (i * i == n) ? 1 : 2;   // i and n/i
    }
    return cnt;
}
```

$\mathcal{O}(\sqrt n)$. The `i * i == n` case is the perfect square, which must be counted once, not twice — forgetting it gives every square number one divisor too many, and it is the single most common bug in this recipe.

### 17.6 Factorial modulo $m$

```cpp
long long factMod(int n, long long m) {
    long long f = 1;
    for (int i = 2; i <= n; i++) f = f * i % m;
    return f;
}
```

Use it when $n$ is small enough to iterate ($n \le 10^7$). For many queries with the same $m$, precompute factorials into an array once: `fact[i] = fact[i-1] * i % m`. Note `0! = 1`, so the loop starting at 2 is correct, and `factMod` with `n = 0` returns 1.

### 17.7 nCr modulo a prime (small $n$)

```cpp
const long long MOD = 1000000007;

long long power(long long b, long long e, long long m) {
    long long r = 1; b %= m;
    while (e > 0) { if (e & 1) r = r * b % m; b = b * b % m; e >>= 1; }
    return r;
}

long long nCr(int n, int r) {
    if (r < 0 || r > n) return 0;
    long long num = 1, den = 1;
    for (int i = 1; i <= r; i++) {
        num = num * (n - r + i) % MOD;
        den = den * i % MOD;
    }
    return num * power(den, MOD - 2, MOD) % MOD;   // Fermat inverse
}
```

$\mathcal{O}(r + \log MOD)$. This is the "small $n$, few queries" version; for $10^5$ queries you precompute factorials and inverse factorials once and answer each in $\mathcal{O}(1)$. Keep `n < MOD`, otherwise the inverse argument breaks.

### 17.8 Parity of a big product

You are given $n$ numbers and asked whether their product is even. Multiplying is hopeless (hundreds of digits). The product is even iff at least one factor is even:

```cpp
bool productIsEven(const vector<long long> &a) {
    for (long long x : a) if (x % 2 == 0) return true;
    return false;
}
```

Generalising: the exponent of 2 in $n!$ is $\lfloor n/2 \rfloor + \lfloor n/4 \rfloor + \lfloor n/8 \rfloor + \cdots$ (Legendre's formula), computable in $\mathcal{O}(\log n)$ without ever forming the factorial. "Parity of a product" and "trailing zeros of $n!$" are the same idea.

### 17.9 Sum $1..n$ without overflow

```cpp
long long sumTo(long long n) {          // n >= 0
    if (n % 2 == 0) return (n / 2) * (n + 1);
    return n * ((n + 1) / 2);
}
```

$\mathcal{O}(1)$ instead of $\mathcal{O}(n)$. The two branches divide an even factor first so that the intermediate product never exceeds the true answer. `n * (n + 1) / 2` overflows for $n \approx 4 \times 10^9$ even though the answer fits in `long long`; this version is safe up to $n \approx 4.3 \times 10^9$ and returns the exact answer for all of them.

---

## Key Takeaways

- `<cmath>` works in `double`: exact integers only to $2^{53}$. For exact integer results, compute in integers and use floats only as a guess.
- `abs` on a `long long` can silently truncate through the `int` overload — use `llabs` or `std::abs`.
- `min`/`max` take two arguments of the *same* type; `min({a,b,c})` works but returns a reference into a temporary, so copy the result.
- `sqrt` is not an exact integer square root: take the float guess and adjust with `while` loops using integer multiplication.
- Never use `pow` for exact integer powers; use a loop or fast exponentiation. `pow(10,9)` can truncate to `999999999`.
- `(int)floor(log10(n)) + 1` miscounts exact powers of 10; use `to_string(n).size()` or add an epsilon.
- C++ division truncates toward zero, so `(-7) % 3 == -1`. Normalise with `(x % m + m) % m`.
- `lcm` must be computed as `a / gcd(a,b) * b`; the naive `a * b` overflows long before the answer does.
- Use `1e18` for infinity, never `LLONG_MAX`, so `INF + INF` stays finite.
- Modular multiplication needs a `long long` (or `__int128` when the modulus exceeds $10^9$); `(a*b) % m` in `int` is wrong above $\sqrt{2^{31}}$.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `abs(x)` where `x` is a big `long long` | Resolves to `abs(int)`; truncates | `llabs(x)` / `std::abs(x)` |
| 2 | `abs(INT_MIN)` | Result not representable → UB | Widen first: `llabs((long long)x)` |
| 3 | `min(a, b, c)` | No such overload | `min({a,b,c})` or nested `min` |
| 4 | `min(1, 2LL)` | Deduction conflict → compile error | `min(1LL, 2LL)` |
| 5 | `const auto& m = min({a,b,c});` | Reference into a destroyed temporary | `int m = min({a,b,c});` |
| 6 | `(long long)sqrt(n)` for $n \le 10^{18}$ | Rounds up near perfect squares | `sqrtl` + adjustment loops |
| 7 | `(long long)pow(10, 9)` | Can be `999999999.999…` | Integer loop or fast pow |
| 8 | `pow(-8, 1.0/3)` | `NaN` (fractional exponent) | Use `cbrt` |
| 9 | Digit count via `floor(log10(n)) + 1` | Off by one at exact powers of 10 | `to_string(n).size()` or `+1e-9` |
| 10 | `a[(i - k) % n]` | Negative index → UB | `((i - k) % n + n) % n` |
| 11 | `a * b / gcd(a,b)` for lcm | `a*b` overflows | `a / gcd * b` |
| 12 | `int` multiplication under a modulus | Overflows at $2^{31}$ | Cast to `long long` before multiplying |
| 13 | `MOD` subtraction without `+MOD` | Negative result | `(a - b + MOD) % MOD` |
| 14 | `const long long INF = LLONG_MAX;` | `INF + w` wraps negative | `1e18` |
| 15 | `__builtin_clz(0)` | Undefined behaviour | Guard `if (x == 0)` |
| 16 | `fabs(a-b) < 1e-9` for $10^{12}$ values | Absolute epsilon too small at scale | Scale the epsilon |
| 17 | `setprecision(6)` without `fixed` | Prints 6 significant digits, not decimals | `fixed << setprecision(6)` |
| 18 | `sin(30)` expecting 0.5 | Trig takes radians | Multiply by `PI/180` |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Digital root / repeated digit sum, $10^{18}$ input | Codeforces Div. 2 A, AtCoder ABC | Easy |
| 2 | Count digits, reverse a number, check palindrome number | AtCoder ABC A/B | Easy |
| 3 | Primality test with $n \le 10^{12}$ | CSES "Prime Check" style tasks | Easy–Medium |
| 4 | Count divisors of $n \le 10^{12}$ | CSES "Counting Divisors" | Easy–Medium |
| 5 | Sum of divisors in $\mathcal{O}(\sqrt n)$ | Codeforces Div. 2 B | Medium |
| 6 | $n! \bmod p$, and trailing zeros of $n!$ | CSES "Mathematics" set | Medium |
| 7 | nCr mod $10^9+7$ with $n \le 10^6$, many queries | Codeforces Div. 2 C / AtCoder ABC | Medium |
| 8 | Compute $a^b \bmod m$ with $b \le 10^{18}$ | CSES "Exponentiation" | Medium |
| 9 | GCD and LCM of an array, answer mod $10^9+7$ | Codeforces Div. 2 B/C | Medium |
| 10 | Geometry: distance/angle with `hypot`, `atan2`, epsilon comparison | Codeforces Div. 2 B (geometry) | Medium |
| 11 | Overflow drill: sum $1..n$ for $n=10^{18}$, exact | Any judge, self-imposed | Easy but instructive |

---

## Next Up

→ [Topic 09 — Arrays (1D, 2D and 3D)](Topic%2009%20-%20Arrays%20%281D,%202D%20and%203D%29.md)