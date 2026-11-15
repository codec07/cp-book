# Topic 02 — Math for CP Part 2: Modular Arithmetic, Exponentiation and Inverses

Every large answer in competitive programming is now printed modulo something. Sometimes the statement says so in the first line, "print the answer modulo $10^9+7$", and sometimes it is buried three paragraphs down in a sentence beginning "since the number can be enormous". Either way, the moment a problem asks for a count, a product, or a power of something huge, you are in the modular world - and the modular world has one rule that trips up everybody who arrives from ordinary arithmetic: **you cannot divide**.

This topic builds the arithmetic that *is* available, the exponentiation that makes huge powers cheap, and the two independent ways to divide anyway when the problem's modulus allows it. The payoff is immediate: fast powers, binomial coefficients, hashing, and most of the counting problems in the rest of the book.

> **Prerequisites:** Topic 01, in particular the Euclidean algorithm and Bézout's identity - the second half of this topic is that identity wearing different clothes.
>
> **You will be able to:** compute $a^b \bmod m$ for $b$ up to $10^{18}$ in about sixty multiplications; decide whether an inverse exists and produce it two different ways; precompute all inverses up to $n$ in linear time; and avoid the three overflow and normalisation bugs that cause most modular wrong answers.

---

## 1. The modular world

Fix an integer $m > 1$, called the **modulus**. Two integers are congruent modulo $m$, written $a \equiv b \pmod m$, when $m \mid (a-b)$. Each integer belongs to exactly one **residue class**, and the class is represented by its remainder in $\{0, 1, \dots, m-1\}$.

The crucial point is that congruence is compatible with addition, subtraction and multiplication:

$$a \equiv b \pmod m \ \text{and}\ c \equiv d \pmod m \ \Longrightarrow \ a + c \equiv b + d,\quad a - c \equiv b - d,\quad ac \equiv bd \pmod m.$$

So you may reduce at every step, keep everything below $m$, and never compute a large number at all. That is why the technique scales: instead of reasoning about $2^{10^{18}}$, a number with three hundred million million digits, you compute sixty residues.

What does **not** survive is division, and the reason is simple: $2 \cdot 3 = 6 \equiv 0 \pmod 6$, so $2 \cdot 3 \equiv 0 \cdot 3 \pmod 6$, yet $2 \not\equiv 0 \pmod 6$. Cancelling a common factor is not allowed in general. Recovering division is the subject of the second half of this topic.

### Where it shows up

- **Counts** that grow factorially or exponentially: "print the number of ways modulo $10^9+7$".
- **Huge powers**: "compute $a^b$ where $b \le 10^{18}$".
- **Hashing**: polynomial hashes are arithmetic modulo a large prime, and $2^{64}$ for `unsigned long long`.
- **Cycles**: "the counter has $m$ positions and rotates $k$ steps" is arithmetic modulo $m$.
- **Geometry and combinatorics** on a circular board, where positions wrap.

The statement will tell you $m$; what it will not tell you is whether $m$ is prime. That single fact decides which inverse technique you may use, so read the constraint carefully: "$10^9+7$" is prime, "$998244353$" is prime, "the number of boxes" is usually not.

---

## 2. Addition, subtraction, multiplication: what survives

With residues in $[0, m)$, the three operations are

```cpp
(a + b) % m          // may overflow if a, b are near m and m > 9e18 / 2 - see below
(a - b + m) % m      // the + m keeps the intermediate non-negative
(a * b) % m          // may overflow if a * b > 9.22e18
```

Each has a failure mode tied to the size of $m$:

| $m$ up to | `a + b` | `a * b` |
|---|---|---|
| $10^9$ | safe in `long long` | safe in `long long` ($10^{18}$) |
| $10^{12}$ | safe ($2 \cdot 10^{12}$) | **overflows** |
| $10^{18}$ | safe ($2 \cdot 10^{18}$) | **overflows** |

So the "safe modulus" for naive multiplication is about $3 \times 10^{9}$; for $10^9+7$ you are fine, for $10^{18}$ you are not. Section 7 fixes that.

Two smaller habits worth forming now:

- Reduce the inputs before the operation (`a %= m; b %= m;`), not after. It costs nothing and keeps products small for longer.
- Never write `x = (x + y) % m` inside a tight loop over $10^{8}$ iterations if you can avoid it; a conditional subtraction is measurably faster. Contests rarely care, but the same function often appears in a loop with a ten-second limit.

### Subtraction is where the bugs live

In C++, `(a - b) % m` can be negative. The mathematical residue is not. The universal fix is to add $m$ before taking the remainder:

```cpp
long long sub(long long a, long long b, long long m) {
    return (a - b % m + m) % m;
}
```

That version is safe because $a \ge 0$ and $0 \le b \bmod m < m$, so the intermediate $a - (b \bmod m) + m$ lies in $(0, 2m)$: it is never negative, and one final reduction lands in $[0, m)$. Compare the two tempting shortcuts: `(a - b) % m + m` can reach $2m$ and leaves an unreduced value, while `((a - b) % m + m) % m` is also correct but pays for two divisions. The `% m` on $b$ first, then one addition of $m$, then one division, is the cheapest correct form.

---

## 3. Normalising: the only safe way to do `%` with negatives

C++'s `%` returns a remainder whose sign follows the dividend. That is fine for a divisibility test and wrong for everything else.

```cpp
long long norm(long long a, long long m) {   // result in [0, m)
    a %= m;
    if (a < 0) a += m;
    return a;
}
```

Use `norm` whenever a value may be negative: a difference of two residues, a coefficient from Bézout, a prefix sum of signed values, or a `long long` read straight from input. It is cheaper than the double-modulo `((a % m) + m) % m` and cannot be written wrongly.

**Contest phrasing to watch for.** "The answer may be negative, print it modulo $m$" means the true answer is an integer that may be negative and you must print its normalised residue. "Print the remainder" means the same thing but is often tested with negative inputs, because setters know it is a trap.

---

## 4. Solved Problem 1 — The Clockmaker's Ledger

> **Statement.** A clockmaker records three quantities about a rotating dial with $m$ positions: the sum, the difference, and the product of two ledger entries $a$ and $b$. The dial counts positions $0$ through $m-1$ and has no negative stickers, so every recorded value must land on the dial: the ledger entry is the position reached, not the arithmetic result. The entries may be negative - the maker uses negative numbers for counter-clockwise turns. Entries are at most $10^9$ in absolute value, and so is $m$.

<details>
<summary>Solution</summary>

### Why this approach

Each of the three quantities is a ring operation reduced modulo $m$, and each may be negative before reduction. The only care needed is to normalise into $[0, m)$ and to keep the product inside `long long`: $|a|, |b| \le 10^{9}$ gives $|ab| \le 10^{18}$, which fits.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long norm(long long a, long long m) {
    a %= m;
    if (a < 0) a += m;
    return a;
}

int main() {
    long long a, b, m;
    cin >> a >> b >> m;

    cout << norm(a + b, m) << ' '
         << norm(a - b, m) << ' '
         << norm(a * b, m) << '\n';
    return 0;
}
```

**Input**

```text
-7 3 5
```

**Output**

```text
1 0 4
```

Check each: $a + b = -4 \equiv 1$, $a - b = -10 \equiv 0$, and $ab = -21 \equiv 4 \pmod 5$.

**Complexity.** $O(1)$.

### The trap

The statement's "every recorded value must land on the dial" is the part that matters, and it is easy to skim past: without it, printing `-1` for the product would look acceptable. In a real problem this sentence is often phrased as "positions are numbered $0$ to $m-1$" and nothing more.

The second trap is `a * b` with $a, b$ near $10^{9}$ and `int`: the product overflows a 32-bit type and the reduction then produces a plausible-looking but wrong residue. Multiply in `long long` even when the answer is small.

</details>

---

## 5. Exponentiation by squaring

Computing $a^{b}$ by repeated multiplication costs $b$ steps, which is hopeless at $b = 10^{18}$. But squaring doubles the exponent cheaply:

$$a^{2k} = (a^k)^2, \qquad a^{2k+1} = (a^k)^2 \cdot a.$$

Read the binary expansion of $b$ from the least significant bit: each bit says "multiply the answer by the current square" (if the bit is one), and each step squares the base. The number of multiplications is at most $2 \lfloor \log_2 b \rfloor + 1$, so $b = 10^{18}$ takes about $120$.

```cpp
// a^e mod m, with e >= 0 and m >= 1. O(log e) multiplications.
long long power(long long a, long long e, long long m) {
    long long result = 1 % m;          // 1 mod 1 == 0
    a %= m;
    while (e > 0) {
        if (e & 1LL) result = result * a % m;
        a = a * a % m;
        e >>= 1;
    }
    return result;
}
```

Three details that are not cosmetic:

- `1 % m` rather than `1`, so that $m = 1$ works. It is a favourite edge case: every power is $0$ modulo $1$.
- `a %= m` before the loop, so a huge base reduces immediately.
- `e >>= 1` on a `long long`, and testing `e & 1LL`; with `int e` the loop is simply wrong for exponents above $2^{31}$.

The recursive version is the same algorithm written the other way:

```cpp
long long powerRec(long long a, long long e, long long m) {
    if (e == 0) return 1 % m;
    long long half = powerRec(a, e / 2, m);
    long long sq = half * half % m;
    return (e & 1LL) ? sq * (a % m) % m : sq;
}
```

It uses $O(\log e)$ stack frames - about sixty, which is fine - but the iterative version is what you should carry, because when you later need to exponentiate a matrix or a linear map you will want the same loop with a different `*`.

### What "the same loop with a different `*`" means

Binary exponentiation needs only an associative operation and an identity. Replace `long long` by a $2 \times 2$ matrix, `*` by matrix multiplication, and `1` by the identity matrix, and you can compute the $n$-th Fibonacci number modulo $m$ in $O(\log n)$. The structure is identical; that is why this technique keeps returning in later topics.

---

## 6. Solved Problem 2 — The Paper Folding

> **Statement.** A sheet of paper is folded so that each fold doubles the number of layers. Starting from a single layer, the stack is folded $b$ times, and after each fold one layer is removed for the archive, so the number of layers in the working stack becomes $a$ times what it was. The archivist wants to know the working stack's size modulo a registry number $m$, because the full number would not fit in any ledger. The starting multiplier is $a$; the number of folds can be as large as $10^{18}$, and $a$ and $m$ are up to $10^{9}$.

<details>
<summary>Solution</summary>

### Why this approach

Each fold multiplies the stack by $a$, so after $b$ folds the size is $a^b$. The statement asks for its residue modulo $m$; with $b$ up to $10^{18}$, repeated multiplication is out of the question, and binary exponentiation is exactly the tool.

Note the second half of the statement - "after each fold one layer is removed" - is decoy prose: it does not change the recurrence the archivist describes. Real statements contain a sentence like this to see whether you are modelling the process or the number.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long power(long long a, long long e, long long m) {
    long long result = 1 % m;
    a %= m;
    while (e > 0) {
        if (e & 1LL) result = result * a % m;
        a = a * a % m;
        e >>= 1;
    }
    return result;
}

int main() {
    long long a, b, m;
    cin >> a >> b >> m;
    cout << power(a, b, m) << '\n';
    return 0;
}
```

**Input**

```text
2 10 1000
```

**Output**

```text
24
```

$2^{10} = 1024$, and $1024 \bmod 1000 = 24$.

**Complexity.** $O(\log b)$ multiplications, $O(1)$ memory.

### The trap

If you write `long long result = 1;` and the modulus is $1$, you print $1$ where the answer is $0$. Moduli of $1$ appear in statements about "a single position" and in stress tests, never in samples.

The second trap is the base: `a %= m` inside `power` is essential when $a$ can be $10^{18}$ and $m$ is small, because otherwise `a * a` overflows before the first reduction. The function above reduces at entry, so it is safe for any `long long` base and any exponent.

</details>

---

## 7. Multiplication overflow: three ways to be safe

When $a$ and $b$ are both below $m \approx 10^{18}$, the product $ab$ can reach $10^{36}$: no 64-bit type holds it. There are three legitimate fixes and one popular non-fix.

**1. A 128-bit intermediate.** On 64-bit GCC and Clang, `__int128` holds $10^{38}$, so `(long long)((__int128)a * b % m)` is exact. It is the fastest and clearest option where it exists; note that 32-bit MinGW builds do not have it.

**2. Add-and-double.** Multiply by repeated doubling, the same bit loop as `power`, which needs $O(\log b)$ additions instead of one multiplication:

```cpp
// a * b mod m for 0 <= a, b < m, safe for m up to 1e18.
long long mulMod(long long a, long long b, long long m) {
    long long result = 0;
    a %= m;
    while (b > 0) {
        if (b & 1LL) { result += a; if (result >= m) result -= m; }
        a += a; if (a >= m) a -= m;
        b >>= 1;
    }
    return result;
}
```

Each `+=` stays below $2m \le 2 \times 10^{18}$, which fits, and the conditional subtraction keeps it below $m$. This is slower by a factor of about sixty but needs no extensions.

**3. Restructure the problem.** If the modulus is large because the *answer* is large, you often do not need the product at all: keep prime exponents, use a bigger type, or find a smaller modulus that the statement also allows.

**The non-fix** is casting to `long double` or using `unsigned long long` wraparound. Floating point loses the low bits of a 60-bit integer, and unsigned wraparound computes modulo $2^{64}$, not modulo $m$; the result is correct only for the rare moduli that divide $2^{64}$.

---

## 8. Division does not exist (until it does)

In the integers, $a/b$ is defined when $b \ne 0$. In the modular world, "$1/b$" means a residue $x$ with $bx \equiv 1 \pmod m$. Such an $x$ is called the **modular inverse** of $b$, written $b^{-1}$.

The inverse exists **exactly when** $\gcd(b, m) = 1$, and when it exists it is unique modulo $m$. Both halves follow from Bézout: $bx + my = 1$ has a solution $(x,y)$ iff $\gcd(b,m) \mid 1$, and reducing that equation modulo $m$ gives $bx \equiv 1$. Uniqueness: if $bx \equiv bx' \equiv 1$, then $b(x - x') \equiv 0$, and since $b$ is invertible, $x \equiv x'$.

So "divide by $b$" in a modular problem is always "multiply by $b^{-1}$", and it is legal only under the coprimality condition. Two consequences that people violate constantly:

- $a / b \bmod m$ is **not** `a / b % m`. The latter is integer division, which throws away exactly the information you need.
- If $b$ shares a factor with $m$, there is no inverse, and the expression $a/b \bmod m$ may still have a well-defined meaning only because $a$ is divisible by the shared factor in the integers. See §15.

There are two standard ways to compute an inverse, and they are not interchangeable in cost or in requirements:

| Method | Requires | Cost | Works when $m$ is composite |
|---|---|---|---|
| Fermat: $b^{m-2} \bmod m$ | $m$ prime, $m \nmid b$ | $O(\log m)$ | no |
| Extended Euclid | $\gcd(b,m) = 1$ | $O(\log m)$, smaller constant | yes |

---

## 9. Fermat's little theorem and inverse by exponentiation

If $p$ is prime and $p \nmid a$, then

$$a^{p-1} \equiv 1 \pmod p .$$

Multiply both sides by $a^{-1}$ (which exists, since $p \nmid a$) to get $a^{p-2} \equiv a^{-1} \pmod p$. So for a prime modulus, the inverse is one call to `power(a, p - 2, p)`.

The general statement is **Euler's theorem**: for any $m$ and any $a$ coprime to $m$, $a^{\varphi(m)} \equiv 1 \pmod m$, hence $a^{-1} \equiv a^{\varphi(m)-1}$. With $m$ prime, $\varphi(m) = m-1$ and you recover Fermat.

Two warnings:

- Using $m-2$ when $m$ is **not** prime is a classic wrong answer. It happens to give the right answer for some composite moduli and some bases, which is worse than failing - it fails on one test in twenty.
- $p = 2$ gives exponent $0$, so the inverse is $1$, which is correct since the only non-zero residue is $1$. The formula survives; your `power` must handle exponent $0$ and $1 \bmod 2 = 1$.

**Fermat is not the fastest.** For a single inverse, extended Euclid is usually faster because it avoids sixty modular multiplications. Fermat wins only when you already need a power, or when the modulus is prime and you are computing many inverses with the factorial trick in §13.

---

## 10. Solved Problem 3 — The Key Maker

> **Statement.** A safe's lock is built on a prime number $p$: the key for a code $a$ is the unique value $k$ with $0 \le k < p$ such that $a \cdot k$ leaves remainder $1$ when divided by $p$. The code is never a multiple of $p$. The safe's manufacturer chose $p$ as large as $2 \times 10^{9}$, and the code can be any value in $[1, p-1]$.

<details>
<summary>Solution</summary>

### Why this approach

"Unique value $k$ with $a \cdot k \equiv 1$" is the definition of the modular inverse. Because $p$ is prime and $a$ is not a multiple of $p$, the inverse exists, and Fermat gives it as $a^{p-2} \bmod p$. The bounds ($p$ up to $2 \times 10^{9}$) make the $O(\log p)$ route mandatory; even the $O(p)$ "try every $k$" is $2 \times 10^{9}$ steps.

Since $p < 2^{31}$, products of two residues are below $4 \times 10^{18}$, which fits in `long long` - no `mulMod` needed.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long power(long long a, long long e, long long m) {
    long long result = 1 % m;
    a %= m;
    while (e > 0) {
        if (e & 1LL) result = result * a % m;
        a = a * a % m;
        e >>= 1;
    }
    return result;
}

int main() {
    long long a, p;
    cin >> a >> p;
    cout << power(a, p - 2, p) << '\n';
    return 0;
}
```

**Input**

```text
3 7
```

**Output**

```text
5
```

$3 \cdot 5 = 15 = 2 \cdot 7 + 1$, so $5$ is the key.

**Complexity.** $O(\log p)$ multiplications, $O(1)$ memory.

### The trap

Exponent $p-2$ requires $p \ge 2$; if a test gives $p = 2$ and $a = 1$, the answer is $1$ and the code above prints $1$, because `power(1, 0, 2) = 1 % 2 = 1`. Good. But with $m = 1$ the same line returns $0$; moduli of $1$ are legal in several judge problems and break every implementation that writes `result = 1`.

The deeper trap is applying this to a composite modulus: the statement says prime, so it is safe *here*, but copying `p - 2` into a problem whose modulus is $10^9$ or $12$ produces wrong answers on a minority of inputs. That pattern - right code, wrong theorem - is the most common source of "why does it pass 9 tests out of 10" in this topic.

</details>

---

## 11. Inverse by extended Euclid (composite moduli)

The inverse is exactly the Bézout coefficient from Topic 01. To solve $ax \equiv 1 \pmod m$, run the extended Euclidean algorithm on $(a, m)$: it returns $(g, x, y)$ with $ax + my = g$. If $g = 1$, then $ax \equiv 1 \pmod m$ and $x \bmod m$ is the inverse. If $g > 1$, no inverse exists, and the algorithm has told you so instead of silently producing nonsense.

```cpp
// Inverse of a modulo m, or -1 if gcd(a, m) != 1. m >= 2.
long long inverse(long long a, long long m) {
    long long x, y;
    long long g = extGcd(a % m, m, x, y);
    if (g != 1) return -1;
    x %= m;
    if (x < 0) x += m;
    return x;
}
```

Why this is the general method: it never assumes $m$ is prime, only that the gcd is $1$. Fermat's theorem is the special case $m$ prime. So when a statement hands you a modulus you have not inspected, this is the function to call; when you know the modulus is prime *and* you need a table of inverses, §13 is faster.

**What if the gcd is not $1$?** Then $a$ is a zero divisor modulo $m$ and has no inverse; division by $a$ is not a legal operation, and a problem asking for it is either ill-posed or relying on the numerator being divisible (see §15). Do not "divide anyway" - the result depends on which representative of the residue class you picked, which is the definition of not being well defined.

---

## 12. Solved Problem 4 — The Safe Combination

> **Statement.** A safe is opened with a dial of $m$ positions. Each key code $a$ has a matching turn count $k$ such that moving $k$ steps $a$ times returns the dial exactly to position $1$ - that is, $a \cdot k$ leaves remainder $1$ modulo $m$. The manufacturer guarantees that every valid key code is coprime to $m$, and that $m \ge 2$. The dial can have up to $10^{12}$ positions, and it is not guaranteed to have a prime number of them. Print the turn count $k$ with $0 \le k < m$.

<details>
<summary>Solution</summary>

### Why this approach

The matching turn count is the modular inverse. The statement is careful to say $m$ is not necessarily prime, so Fermat's $m-2$ exponent is not available; extended Euclid works for any modulus as long as the gcd is $1$, which the manufacturer guarantees.

$m \le 10^{12}$ also rules out any table of size $m$. Euclid touches $O(\log m)$ values and never looks at the modulus's size, which is exactly the property being tested.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long extGcd(long long a, long long b, long long &x, long long &y) {
    if (b == 0) { x = 1; y = 0; return a; }
    long long x1, y1;
    long long g = extGcd(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}

int main() {
    long long a, m;
    cin >> a >> m;
    a %= m;

    long long x, y;
    long long g = extGcd(a, m, x, y);
    if (g != 1) { cout << -1 << '\n'; return 0; }

    x %= m;
    if (x < 0) x += m;
    cout << x << '\n';
    return 0;
}
```

**Input**

```text
3 10
```

**Output**

```text
7
```

$3 \cdot 7 = 21 = 2 \cdot 10 + 1$. Note that $10$ is composite, which is the point. Watch what happens if you apply Fermat anyway: the exponent would be $m-2 = 8$, and $3^{8} = 6561 \equiv 1 \pmod{10}$, so the "inverse" would come out as $1$ - which is false, because $3 \cdot 1 = 3 \not\equiv 1$. The congruence $3^{8} \equiv 1$ is Euler's theorem with $\varphi(10) = 4$, and it says nothing about inverses.

**Complexity.** $O(\log m)$ time and recursion depth, $O(1)$ memory.

### The trap

The trap is copying the previous problem's `p - 2` with `p` renamed. For $a = 3$, $m = 10$, `power(3, 8, 10)` happens to give $1$, so the "inverse" becomes $1$ - which is wrong, since $3 \cdot 1 = 3 \not\equiv 1$. Any test with a composite modulus exposes it, and such tests are precisely why the statement bothered to say "not guaranteed to be prime".

The other trap is the sign: `extGcd` may return a negative $x$, and the statement demands $0 \le k < m$. Normalise before printing.

</details>

---

## 13. Inverses for everything at once: tables, factorials, inverse factorials

When a problem needs many inverses modulo a **prime** $m$, there are two precomputations worth knowing.

### All inverses up to $n$, in $O(n)$

```cpp
// inv[i] = inverse of i modulo prime m, for 1 <= i <= n < m.
vector<long long> inverses(int n, long long m) {
    vector<long long> inv(n + 1);
    inv[1] = 1;
    for (int i = 2; i <= n; ++i)
        inv[i] = m - (m / i) * inv[m % i] % m;   // result in [1, m)
    return inv;
}
```

The identity behind it: $m = \lfloor m/i \rfloor \cdot i + (m \bmod i)$, reduced modulo $m$, gives $\lfloor m/i \rfloor \cdot i \equiv -(m \bmod i)$, and multiplying by $i^{-1}(m \bmod i)^{-1}$ solves for $i^{-1}$. The subtlety is that it uses `inv[m % i]`, and $m \bmod i < i$, so the value is already computed - which is why `i < m` is required (otherwise the index does not exist).

### Factorials and inverse factorials

For binomial coefficients you need $n!$ and $(n!)^{-1}$. Compute the factorials upward, invert the last one once with Fermat, then walk downward:

```cpp
vector<long long> fact(n + 1), invFact(n + 1);
fact[0] = 1;
for (int i = 1; i <= n; ++i) fact[i] = fact[i - 1] * i % m;

invFact[n] = power(fact[n], m - 2, m);          // requires m prime, m > n
for (int i = n; i >= 1; --i) invFact[i - 1] = invFact[i] * i % m;
```

This costs one exponentiation plus $2n$ multiplications, and then every $\binom{n}{r}$ is three multiplications. The condition `m > n` matters: if $m \le n$ then $n! \equiv 0 \pmod m$ and it has no inverse. Problems that need binomials with $n \ge m$ are a different, harder species (Lucas' theorem, Level 4).

---

## 14. Solved Problem 5 — The Lottery Draw

> **Statement.** A lottery draws $r$ balls from a drum of $n$ numbered balls, and the order of the draw does not matter. The organiser must publish the number of possible draws modulo a registry prime $p$; the registry prime is larger than $n$, and $n$ can be up to one million. Was the draw of $r$ balls one of how many possibilities?

<details>
<summary>Solution</summary>

### Why this approach

The number of unordered selections is $\binom{n}{r} = \frac{n!}{r!(n-r)!}$. In a modular world the division must become multiplication by inverses, and since $p$ is prime and larger than $n$, every factorial involved is non-zero modulo $p$ and invertible.

The bound $n \le 10^{6}$ is the permission slip for a factorial table; the answer is then $O(1)$ per query.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

long long power(long long a, long long e, long long m) {
    long long result = 1 % m;
    a %= m;
    while (e > 0) {
        if (e & 1LL) result = result * a % m;
        a = a * a % m;
        e >>= 1;
    }
    return result;
}

int main() {
    int n, r;
    long long p;
    cin >> n >> r >> p;

    vector<long long> fact(n + 1), invFact(n + 1);
    fact[0] = 1;
    for (int i = 1; i <= n; ++i) fact[i] = fact[i - 1] * i % p;

    invFact[n] = power(fact[n], p - 2, p);
    for (int i = n; i >= 1; --i) invFact[i - 1] = invFact[i] * i % p;

    if (r < 0 || r > n) { cout << 0 << '\n'; return 0; }

    long long answer = fact[n] * invFact[r] % p * invFact[n - r] % p;
    cout << answer << '\n';
    return 0;
}
```

**Input**

```text
6 3 1000000007
```

**Output**

```text
20
```

$\binom{6}{3} = \frac{6 \cdot 5 \cdot 4}{3 \cdot 2 \cdot 1} = 20$, and the registry prime is large enough that the division is a genuine inverse.

**Complexity.** $O(n + \log p)$ preprocessing, $O(1)$ per query, $O(n)$ memory.

### The trap

Using `fact[n] * invFact[r] * invFact[n - r] % p` in one expression is a right-associative chain of `%` that is usually fine, but if `p` is near $10^{9}$ the *intermediate* `fact[n] * invFact[r]` is below $10^{18}$ and safe; with `p` near $2 \times 10^{9}$ it reaches $4 \times 10^{18}$ and is still safe. It becomes unsafe only for $p$ near $10^{18}$, where you need `mulMod`.

The real trap is that $\binom{n}{r}$ with $r > n$ must be $0$, and the formula will happily compute garbage - or index out of bounds if you compute `n - r` as an `int`. Check the range before the arithmetic.

</details>

---

## 15. Composite moduli: when division is illegal, and cancelling

Everything hard about modular arithmetic happens when the modulus is not prime. Three rules cover almost every situation.

**Rule 1 - invertibility is the gcd.** $a$ is invertible modulo $m$ if and only if $\gcd(a,m)=1$. With $m = 12$, the invertible residues are $1, 5, 7, 11$: exactly $\varphi(12) = 4$ of them. Everything else is a zero divisor: $6 \cdot 2 \equiv 0$, so $6$ cannot be cancelled.

**Rule 2 - cancelling is allowed, at a price.** If $ac \equiv bc \pmod m$ and $g = \gcd(c,m)$, then

$$a \equiv b \pmod{m/g}.$$

With $c$ invertible ($g=1$) you recover the usual cancellation. With $g > 1$ the conclusion becomes weaker: $2 \cdot 3 \equiv 8 \cdot 3 \pmod 6$ gives $2 \equiv 8 \pmod 2$, which is true and useless.

**Rule 3 - quotients of exact integers are not quotients of residues.** "Print $\binom{n}{r} / 2 \bmod m$" cannot be computed as `C(n,r) % m * inv(2) % m` when $m$ is even, because $2$ has no inverse. You must either do the division before reducing (when the numbers are small), or keep prime exponents and cancel there. This is the reason several counting problems ask for the answer modulo a prime: the setter wants you to be able to divide.

### The exponent-reduction rule, precisely

Euler's theorem lets you reduce exponents: if $\gcd(a,m)=1$ then $a^{k} \equiv a^{k \bmod \varphi(m)} \pmod m$ for $k \ge 0$ (careful with $k=0$ and $\varphi(m)=1$). If $a$ is **not** coprime to $m$, this reduction can be wrong, and problems that invite it - "compute $a^b \bmod m$ where $b$ has a thousand digits" with composite $m$ - require the extended Euler theorem, which is Level 4 material. The safe answer: reduce exponents only when the base is coprime to the modulus, or when the modulus is prime and the base is not a multiple of it.

---

## 16. Choosing a tool: bounds and complexity

| Task | Condition | Tool | Cost |
|---|---|---|---|
| $a^b \bmod m$ | $b \le 10^{18}$, $m \le 10^{9}$ | binary exponentiation | $O(\log b)$ |
| $a^b \bmod m$ | $a, m$ up to $10^{18}$ | exponentiation + `mulMod` | $O(\log b \log m)$ |
| $a^{-1} \bmod p$ | $p$ prime, one value | Fermat: $a^{p-2}$ | $O(\log p)$ |
| $a^{-1} \bmod m$ | $\gcd(a,m)=1$, any $m$ | extended Euclid | $O(\log m)$ |
| all inverses $1..n$ | $m$ prime, $n < m$ | linear recurrence | $O(n)$ |
| $\binom{n}{r} \bmod p$ | $p$ prime, $p > n \le 10^{6}$ | factorials + inverse factorials | $O(n)$ once, $O(1)$ each |
| $\binom{n}{r} \bmod p$ | many queries, $n \le 10^{6}$ | same tables | $O(n + q)$ |
| $\binom{n}{r} \bmod p$ | $n \ge p$ | Lucas' theorem | Level 4 |
| division modulo $m$ | $\gcd(b,m) > 1$ | impossible in general | restructure |

The decision procedure is two questions, in order: *is the modulus prime?* and *how many inverses do I need?* Prime plus many gives tables; prime plus one gives Fermat or Euclid (Euclid is faster); composite gives Euclid or a redesign.

---

## 17. Unsolved problems

Six problems. Two are straightforward exponentiation, two are about choosing the right inverse method, and two require you to notice something the statement does not say out loud.

### 17.1 The Rotating Dial

> A machine has two independently rotating dials with $m$ positions each, and a job advances the first by $a$ steps $b$ times and the second by $c$ steps $d$ times. The supervisor needs the total advance, taken modulo $m$, to schedule the next job. All of $a, c, m$ are at most $10^{9}$, and the step counts $b$ and $d$ are at most $10^{18}$. Print the total advance.

### 17.2 The Ticket Booth

> A ticket booth sells unordered sets of $r$ seats from a row of $n$ seats. The manager wants the number of possible sets modulo a prime $p$ that is larger than $n$, and there are $q$ customer enquiries, each with its own $r$. The row has at most $10^{6}$ seats, and there are at most $10^{5}$ enquiries.

### 17.3 The Combination Lock

> A lock accepts a code $a$ modulo $m$ and opens only if there is a turn count $k$ with $a \cdot k \equiv 1 \pmod m$. Given $a$ and $m$ with $2 \le m \le 10^{12}$, print the smallest such $k$ with $0 \le k < m$, or $-1$ if no such turn count exists. The lock's manufacturer makes no promise about $m$ being prime.

### 17.4 The Signal Tower

> A tower flashes a lamp with period $a$ seconds and a receiver samples it every second, recording the lamp's state modulo $m$. The engineer wants the smallest positive number of samples after which the pattern repeats exactly, that is, the smallest $k \ge 1$ with $a^{k} \equiv 1 \pmod m$. If the pattern never repeats, print $-1$. Here $2 \le a < m \le 10^{6}$.

### 17.5 The Vending Machine

> A vending machine's coin mechanism computes $a^{b} \bmod m$ as part of its firmware self-test, where $a$, $b$ and $m$ can each be as large as $10^{18}$. The machine prints the result. Note that the intermediate products can be far beyond 64 bits.

### 17.6 The Wall

> A wall is built from $n$ bricks, each either red or blue, and no two red bricks may stand next to each other. The mason wants the number of possible colourings modulo $m$, where $n$ can be $10^{18}$ and $m$ is any integer up to $10^{9}$. Print the count.

---

## Key Takeaways

1. Congruence is compatible with $+$, $-$ and $\times$, so you may reduce at every step; it is not compatible with division unless the divisor is invertible.
2. `%` in C++ follows the sign of the dividend. `a %= m; if (a < 0) a += m;` is the normalisation to use everywhere.
3. Binary exponentiation needs about $2\log_2 b$ multiplications: $b = 10^{18}$ costs about $120$, not $10^{18}$.
4. `1 % m`, not `1`, for the identity - moduli of $1$ exist in real tests.
5. $a$ is invertible modulo $m$ exactly when $\gcd(a,m)=1$; that is the whole criterion, and Fermat is only the prime special case.
6. Fermat ($a^{p-2}$) needs a prime modulus; extended Euclid works for any modulus with gcd $1$ and is usually faster for a single inverse.
7. The linear inverse table and the factorial/inverse-factorial tables turn per-query inverses into $O(1)$ lookups - but only for prime moduli larger than the largest index.
8. When $a \cdot b$ can exceed $9.22 \times 10^{18}$, use a 128-bit intermediate or add-and-double multiplication; never floating point, never unsigned wraparound.
9. $\operatorname{lcm}$-like division and quotients of huge integers cannot be pushed through a composite modulus; track prime exponents instead.
10. Most modular wrong answers are one of three bugs: a negative residue, an overflowed product, or a prime-modulus theorem applied to a composite modulus.

---

## Common Mistakes & Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `(a - b) % m` used as a residue | negative answer | `(a - b % m + m) % m` |
| `a * b % m` with $a, b \approx 10^{18}$ | wildly wrong values | `mulMod` or `__int128` |
| `result = 1` in `power` | wrong answer for $m = 1$ | `1 % m` |
| `int` exponent in `power` | infinite loop or wrong for $b > 2^{31}$ | `long long e`, `e >>= 1` |
| `a / b % m` | integer division silently applied | multiply by `inverse(b, m)` |
| Fermat on a composite modulus | passes small tests, fails others | extended Euclid |
| `invFact[n] = power(fact[n], p-2, p)` with $p \le n$ | inverse of $0$ | require $p > n$, or Lucas |
| forgetting `a %= m` before `a * a` | overflow at large bases | reduce at entry |
| reducing an exponent modulo $\varphi(m)$ when $\gcd(a,m) > 1$ | wrong for some inputs | only reduce when coprime |
| assuming `%` gives a value in $[0,m)$ for negative inputs | negative residues propagating into comparisons | `norm()` |
| writing `1e9+7` as a `double` | silent precision loss | integer literal `1000000007LL` |

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Clockmaker's Ledger | $+,-,\times$ with negative entries | §4 |
| 2 | The Paper Folding | $a^b \bmod m$, $b \le 10^{18}$ | §6 |
| 3 | The Key Maker | inverse modulo a prime, by Fermat | §10 |
| 4 | The Safe Combination | inverse modulo a composite, by Euclid | §12 |
| 5 | The Lottery Draw | $\binom{n}{r}$ with factorial tables | §14 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Rotating Dial | $b, d \le 10^{18}$ | two exponentiations, then one addition |
| 2 | The Ticket Booth | $n \le 10^{6}$, $q \le 10^{5}$ | precompute once, answer in $O(1)$ |
| 3 | The Combination Lock | $m \le 10^{12}$ | Euclidean inverse, with a real "no inverse" case |
| 4 | The Signal Tower | $m \le 10^{6}$ | the order divides $\varphi(m)$; factor it |
| 5 | The Vending Machine | all values $10^{18}$ | exponentiation with overflow-safe multiplication |
| 6 | The Wall | $n \le 10^{18}$, $m \le 10^{9}$ | exponentiate a linear recurrence |

Problem 4 is the hardest: once you know the order divides $\varphi(m)$, you still have to find $\varphi(m)$, factor it, and test the divisors - all with the tools of Topic 01. Problem 6 is the bridge to the next chapter: the multiplication in the exponentiation loop becomes a matrix product.

---

## Next Up

Topic 03 turns these primitives into **counting**: binomial coefficients and their identities, stars and bars, derangements, and expected values computed by linearity rather than by enumerating outcomes. The modular machinery is the vehicle; the new content is the combinatorial modelling. If you want a warm-up, the identity $\binom{n}{r} = \binom{n-1}{r-1} + \binom{n-1}{r}$ is a recurrence - and recurrences with a matrix are exactly what problem 6 above is asking for.

