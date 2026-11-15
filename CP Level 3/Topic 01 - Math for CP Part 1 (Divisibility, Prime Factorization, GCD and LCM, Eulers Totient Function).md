# Topic 01 — Math for CP Part 1: Divisibility, Prime Factorization, GCD and LCM, Euler's Totient

Here is a thing that surprises almost everyone at some point: in a two-hour contest, the problem that looks like a story about crates, gears, or seating plans is very often a problem about **divisors**. The statement never says "divisors". It says "the warehouse can stack the crates into rectangles with no gaps", and your job is to notice that this is asking how many factorisations a number has.

The arithmetic itself is not hard. Long division was school. What contests test is the *translation*: from a sentence about physical objects to one of maybe four arithmetic quantities, and then the engineering choice of how to compute that quantity inside the time limit. This topic builds both halves. By the end you will have the four primitives - divisor counting, factorisation, gcd, and Euler's totient - as reusable code, and, more importantly, a habit of asking what a statement is really counting.

> **Prerequisites:** loops, arrays, `vector`, `long long` versus `int`, and the idea of time complexity. Nothing from earlier chapters is assumed beyond that.
>
> **You will be able to:** decide in under a minute whether a problem needs a sieve or trial division; factorise numbers up to $10^{12}$ fast; compute $\gcd$, $\operatorname{lcm}$ and the extended Euclidean coefficients without overflow; and recognise Euler's totient in disguise.

---

## 1. The three questions number theory keeps asking

Almost every elementary number theory problem in competitive programming asks one of these:

1. **How many divisors does $n$ have?** Or, equivalently, in how many ways can $n$ identical objects be arranged into a rectangle? Sometimes it is "how many divisors are even", "how many are perfect squares", "what is the largest divisor below $k$".
2. **What is the greatest common divisor of these numbers?** Often hidden as "the largest tile that can pave the floor exactly", "the largest step size that visits both marks", "the last common point of two repeating cycles".
3. **How many numbers are coprime to $n$, or in lowest terms?** This is Euler's totient, and it appears in fraction counting, in modular inverses (next topic), and in "how many fractions between 0 and 1 with denominator at most $n$".

Everything else in this topic is machinery for those three. Keep this list in mind when you read a statement: nine times out of ten, one of those three questions is what the story is about.

### The number you will actually work with

Contest constraints decide your algorithm, and the relevant number is not $n$ but $n$'s **size in the worst case**:

| Bound on $n$ | What is feasible | What is not |
|---|---|---|
| $n \le 10^{6}$ | guessing every integer, a full array indexed by value | - |
| $n \le 10^{7}$ | a sieve of size $n$ (about 10 MB) | a nested loop over $n$ |
| $n \le 10^{12}$ | $\sqrt{n} \approx 10^{6}$ trial divisions per query | a sieve up to $n$ |
| $n \le 10^{18}$ | $\sqrt{n} \approx 10^{9}$ operations only if there is **one** query | trial division per query, in bulk |

Read that table backwards when you write a statement's solution: the allowed $\sqrt{n}$ or $n$ is the hint the setter is legally allowed to give you.

---

## 2. Divisibility: the rules that carry the most weight

For integers $a$ and $b$ with $b \neq 0$, we say $b$ **divides** $a$, written $b \mid a$, when $a = b \cdot k$ for some integer $k$. Three facts do most of the work:

- **Transitivity.** If $a \mid b$ and $b \mid c$, then $a \mid c$.
- **Linear combinations.** If $d \mid a$ and $d \mid b$, then $d \mid (ax + by)$ for *any* integers $x, y$ - including $d \mid (a - b)$ and $d \mid (a \bmod b)$.
- **The division lemma.** For $a \ge 0$ and $b > 0$ there are unique $q, r$ with $a = qb + r$ and $0 \le r < b$.

The second fact is the reason the Euclidean algorithm works, and the third is the reason it terminates. Remember that pair; they are the proof of the whole next section.

### The trap that eats first submissions

In C++, `%` is a **remainder**, not a mathematical modulus: the sign follows the left operand.

```cpp
-7 % 3     // -1, not 2
7 % -3     // 1,  not 1 (fine here, but not by accident)
```

So a test like `if (a % b == 0)` is safe, but anything that treats `%` as a value in $[0, b)$ is not. When you need a true mathematical modulus, write

```cpp
long long mod(long long a, long long m) {      // result in [0, m)
    long long r = a % m;
    return r < 0 ? r + m : r;
}
```

and be suspicious of every `%` in code you did not write yourself. The rest of this topic assumes non-negative inputs unless it says otherwise.

### Reading divisibility out of a statement

Statements almost never say "divides". They say:

- "can be split into groups of equal size" - some divisor is the group size;
- "fits exactly", "no gaps", "tiles the floor" - a divisor;
- "every $k$-th item", "starting from position $k$, every $k$" - multiples of $k$;
- "the largest step that hits both marks" - a gcd;
- "repeats after" - an lcm.

Train yourself to underline those phrases. It is the single highest-yield reading skill in this chapter.

---

## 3. From factorisation to divisors: $\tau$ and $\sigma$

Every integer $n \ge 2$ has a unique factorisation

$$n = p_1^{e_1} p_2^{e_2} \cdots p_k^{e_k}$$

with distinct primes $p_i$ and positive exponents $e_i$. This is the fundamental theorem of arithmetic, and every divisor formula in this book is a corollary of it.

A divisor of $n$ must use each prime $p_i$ between $0$ and $e_i$ times, independently. So the number of divisors is

$$\tau(n) = (e_1 + 1)(e_2 + 1) \cdots (e_k + 1),$$

and the sum of divisors is

$$\sigma(n) = \prod_{i=1}^{k} \left(1 + p_i + p_i^2 + \cdots + p_i^{e_i}\right) = \prod_{i=1}^{k} \frac{p_i^{e_i+1} - 1}{p_i - 1}.$$

For $n = 36 = 2^2 \cdot 3^2$: $\tau(36) = 3 \cdot 3 = 9$ (they are $1, 2, 3, 4, 6, 9, 12, 18, 36$) and $\sigma(36) = (1+2+4)(1+3+9) = 7 \cdot 13 = 91$.

Two useful consequences:

- $\tau(n)$ is odd **exactly when** $n$ is a perfect square, because every exponent is even only then. This is why "how many divisors are odd" has a one-line answer.
- The product formula for $\sigma$ behaves badly modulo a prime when $p_i \equiv 1 \pmod{M}$, because you would divide by zero. Either use the geometric sum directly or handle that case separately.

### How large can $\tau(n)$ get?

For $n \le 10^{9}$ the maximum is $1344$. For $n \le 10^{12}$ it is $6720$, and for $n \le 10^{18}$ it is $103680$. This is why "enumerate all divisors of $n \le 10^{12}$" is feasible: there are at most $6720$ of them, so even an $O(\tau(n)^2)$ loop is fine. It is also why you should never allocate arrays indexed by a divisor count.

```cpp
// Number of divisors of n, in O(sqrt(n)). Safe for n up to about 1e18.
long long countDivisors(long long n) {
    long long count = 1;
    for (long long p = 2; p <= n / p; ++p) {   // p <= n/p, not p*p <= n
        if (n % p != 0) continue;
        int e = 0;
        while (n % p == 0) { n /= p; ++e; }
        count *= (e + 1);
    }
    if (n > 1) count *= 2;                      // one prime factor larger than sqrt(n)
    return count;
}
```

Note the loop condition `p <= n / p`. The popular `p * p <= n` overflows when $n$ is near $10^{18}$ and $p$ is near $10^{9}$ - and the pair $10^{9}$ squared is $10^{18}$, which is *just* inside `long long`, so the bug appears only for the largest inputs. Division costs a few cycles more and never lies.

---

## 4. Solved Problem 1 — The Locked Drawers

> **Statement.** A workshop in an old observatory keeps $n$ drawers, numbered $1$ through $n$. The director has a superstition about drawers whose number can be written as a product of two integers both greater than $1$: she calls those *composite* and refuses to store instruments in them. Every other drawer above number $1$ she calls *lonely*. Before the annual inventory she wants to know how many lonely drawers there are, because that is how many brass labels she must order. The workshop can hold up to ten million drawers.

<details>
<summary>Solution</summary>

### Why this approach

"Not a product of two integers greater than $1$" is exactly primality, with $1$ excluded by saying "above number 1". So we are asked for $\pi(n)$, the number of primes up to $n$, with $n \le 10^{7}$.

Trial division per number would be $n \sqrt{n} \approx 3 \times 10^{10}$ operations - hopeless. But $n$ is small enough to hold a boolean array of size $n + 1$ in memory (10 MB), and that is the entire point of the bound: the setter is telling you a sieve fits.

### Solution

Mark composites instead of testing primes. For every prime $p$ you meet, cross out $p^2, p^2 + p, p^2 + 2p, \dots$ Starting at $p^2$ is not an optimisation detail but a correctness-of-cost detail: the smaller multiples $2p, 3p, \dots$ have already been crossed out by smaller primes.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<bool> composite(n + 1, false);
    int lonely = 0;

    for (int i = 2; i <= n; i++) {
        if (composite[i]) continue;
        lonely++;                                  // i is prime
        if (1LL * i * i <= n)
            for (long long j = 1LL * i * i; j <= n; j += i)
                composite[j] = true;
    }

    cout << lonely << '\n';
    return 0;
}
```

**Input**

```text
30
```

**Output**

```text
10
```

The lonely numbers are $2, 3, 5, 7, 11, 13, 17, 19, 23, 29$, ten of them.

**Complexity.** $O(n \log \log n)$ time, $O(n)$ memory, and `vector<bool>` packs the flags into bits so the array costs about $n/8$ bytes.

### The trap

Three ways this goes wrong in a contest:

1. Counting $1$ as lonely. It has no divisor other than itself, so the naive predicate accepts it. The statement's "above number 1" exists precisely because setters know this.
2. Writing `i * i <= n` with `int i`. For $n \le 10^{7}$ this is harmless, but the same template pasted into a $10^{9}$ problem silently overflows. Use `1LL * i * i`.
3. Crossing out from $2p$ instead of $p^2$. Correct, but roughly twice as slow; at $n = 10^{7}$ that is the difference between 40 ms and 90 ms - not fatal here, fatal in a tight multi-test file.

</details>

---

## 5. The sieve of Eratosthenes, carefully

The sieve is the first algorithm in this book where a constant factor decides a verdict, so it is worth writing properly once.

```cpp
// All primes up to n. O(n log log n) time, O(n) bits for the flags.
vector<int> sieve(int n) {
    vector<bool> composite(n + 1, false);
    vector<int> primes;
    for (int i = 2; i <= n; ++i) {
        if (composite[i]) continue;
        primes.push_back(i);
        if (1LL * i * i <= n)
            for (long long j = 1LL * i * i; j <= n; j += i) composite[j] = true;
    }
    return primes;
}
```

**How much memory?** A `vector<char>` of size $10^{8}$ is 100 MB, which will exceed a 256 MB limit together with everything else. `vector<bool>` is a bitset: $10^{8}$ flags cost 12.5 MB. If you need `char` speed, sieve only odd numbers and index $i$ as the number $2i+1$, halving both time and memory.

**How much time?** The number of marking operations is $n \sum_{p \le n} 1/p \approx n \ln \ln n$. For $n = 10^{7}$ that is about $3 \times 10^{7}$ marks: well under a second. For $n = 10^{8}$ it is about $3 \times 10^{8}$: borderline, and you should reduce it with the odd-only trick.

**When the bound does not fit.** If $n \le 10^{9}$ you cannot sieve to $n$, but you *can* sieve to $\sqrt{n}$ and use that to test individual numbers, or use a segmented sieve that processes the range in blocks of $10^{6}$. Both are Level 4 material; here the rule of thumb is enough: **a sieve needs $n$; trial division needs $\sqrt{n}$ per query.**

---

## 6. Trial division: factorising what you cannot sieve

To factorise $n$, divide out every prime you find, starting from $2$. The loop condition deserves attention:

```cpp
// Prime factorisation as (prime, exponent) pairs. O(sqrt(n)) worst case.
vector<pair<long long,int>> factorise(long long n) {
    vector<pair<long long,int>> factors;
    for (long long p = 2; p <= n / p; ++p) {
        if (n % p != 0) continue;
        int e = 0;
        while (n % p == 0) { n /= p; ++e; }
        factors.push_back({p, e});
    }
    if (n > 1) factors.push_back({n, 1});   // the leftover prime
    return factors;
}
```

Why does the loop only need to reach $\sqrt{n}$? Suppose $n$ has two prime factors both larger than $\sqrt{n}$. Their product exceeds $n$ - contradiction. So **after** dividing out everything up to $\sqrt{n}$, whatever remains is either $1$ or a single prime. The `if (n > 1)` line is not a technicality; omit it and $n = 14$ factorises to just $2$.

### The 6k ± 1 wheel

Every prime above $3$ is of the form $6k \pm 1$, because $6k$, $6k+2$, $6k+3$, $6k+4$ are all divisible by $2$ or $3$. Testing only those candidates cuts the work by a factor of three:

```cpp
vector<pair<long long,int>> factoriseWheel(long long n) {
    vector<pair<long long,int>> factors;
    for (long long p : {2LL, 3LL}) {
        if (n % p) continue;
        int e = 0;
        while (n % p == 0) { n /= p; ++e; }
        factors.push_back({p, e});
    }
    for (long long p = 5; p <= n / p; p += 6) {
        for (long long q : {p, p + 2}) {
            if (n % q) continue;
            int e = 0;
            while (n % q == 0) { n /= q; ++e; }
            factors.push_back({q, e});
        }
    }
    if (n > 1) factors.push_back({n, 1});
    return factors;
}
```

At $n \approx 10^{12}$, this is about $3.3 \times 10^{5}$ iterations of the outer loop instead of $10^{6}$.

### Which one should I write?

If the statement gives you **one** number up to $10^{12}$ and a two-second limit, the wheel version is sufficient at $\approx 3 \times 10^{5}$ divisions. If it gives you $10^{5}$ numbers up to $10^{12}$, no per-query factorisation will pass - you need the next section's sieve, or a different observation entirely.

---

## 7. Solved Problem 2 — The Chocolate Tray

> **Statement.** A factory packs $n$ identical chocolate bars into a single rectangular tray. Every bar must be used, bars may not be cut, and a tray $a$ bars wide and $b$ bars long is the same tray as $b$ wide and $a$ long. The planner needs to know how many genuinely different trays are possible, so she can price the packaging. The order size can be as large as one trillion bars.

<details>
<summary>Solution</summary>

### Why this approach

A tray is a pair $(a, b)$ with $a \cdot b = n$; identifying $(a,b)$ with $(b,a)$ means counting the divisors $a$ up to $\sqrt{n}$ and doubling, with care when $a = b$. So the answer is $\tau(n)$, the divisor count.

$n \le 10^{12}$ means $\sqrt{n} \le 10^{6}$: a single trial-division loop of at most a million iterations, far inside any limit. No sieve is possible ($10^{12}$ flags is a terabyte) and none is needed.

### Solution

The clean way is to use the factorisation formula: $\tau(n) = \prod (e_i + 1)$. Writing it as a loop over candidate divisors and doubling would also work, but the factorisation version answers "how many" without ever visiting the divisors, which matters the moment $\tau$ gets large.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    long long n;
    cin >> n;

    long long ways = 1;
    for (long long p = 2; p <= n / p; p++) {
        if (n % p != 0) continue;
        int e = 0;
        while (n % p == 0) { n /= p; e++; }
        ways *= (e + 1);
    }
    if (n > 1) ways *= 2;

    cout << ways << '\n';
    return 0;
}
```

**Input**

```text
36
```

**Output**

```text
9
```

The trays are $1\times36$, $2\times18$, $3\times12$, $4\times9$, $6\times6$, $9\times4$, $12\times3$, $18\times2$, $36\times1$ - nine, and $6 \times 6$ is counted once because the two orders are the same tray.

**Complexity.** $O(\sqrt{n})$ time, $O(1)$ memory.

### The trap

The statement says a tray $a \times b$ equals $b \times a$, so the answer is $\tau(n)$ and **not** the number of ordered factorisations. If you loop $a$ from $1$ to $\sqrt{n}$ and add $2$ per divisor, remember the square case: $a = b$ must add $1$. The factorisation formula avoids the question entirely, which is one reason to prefer it under time pressure.

</details>

---

## 8. Smallest prime factor sieve: $O(\log n)$ per query

When a statement hands you up to $2 \times 10^{5}$ numbers below $10^{6}$ and asks something about each one's prime factors, per-query trial division at $10^{3}$ steps sounds fine ($2 \times 10^{8}$ total - too slow in practice). The fix is to precompute, for every number up to the limit, its **smallest prime factor**:

```cpp
// spf[x] = smallest prime dividing x, with spf[0] = 0 and spf[1] = 1.
vector<int> buildSPF(int N) {
    vector<int> spf(N + 1);
    for (int i = 0; i <= N; ++i) spf[i] = i;
    for (int i = 2; 1LL * i * i <= N; ++i) {
        if (spf[i] != i) continue;                 // i is composite
        for (long long j = 1LL * i * i; j <= N; j += i)
            if (spf[j] == j) spf[j] = i;           // first prime to reach j wins
    }
    return spf;
}
```

Once built, factorisation is a division chain: repeatedly take `p = spf[n]`, divide it out, and never trial-divide anything. The number of steps is the number of prime factors with multiplicity, so at most $\log_2 n$ - about $20$ for a million.

```cpp
vector<pair<int,int>> factorWithSPF(int x, const vector<int>& spf) {
    vector<pair<int,int>> factors;
    while (x > 1) {
        int p = spf[x], e = 0;
        while (x % p == 0) { x /= p; ++e; }
        factors.push_back({p, e});
    }
    return factors;
}
```

**Memory is the real constraint here.** An `int` per number means $4$ bytes: $10^{7}$ numbers is 40 MB, which is acceptable; $10^{8}$ is 400 MB, which is not. For $10^{8}$ you would store the smallest prime factor as a `uint16_t` only if the limit allows it (no - the values need 27 bits), or use a different structure. In practice: SPF for limits up to $10^{7}$, sieve of primes plus trial division for larger values.

**Build cost.** $O(N \log \log N)$, the same order as the sieve, with a constant that is maybe twice as large. So building SPF is worth it from roughly $10^{3}$ queries upward.

---

## 9. GCD and the Euclidean algorithm

The greatest common divisor of $a$ and $b$ is the largest $d$ with $d \mid a$ and $d \mid b$. The algorithm is over two thousand years old and still the fastest thing of its kind:

$$\gcd(a, b) = \gcd(b, a \bmod b), \qquad \gcd(a, 0) = a.$$

**Why it is true.** Write $a = qb + r$. Any common divisor of $a$ and $b$ divides $a - qb = r$, so it divides $b$ and $r$. Conversely any common divisor of $b$ and $r$ divides $qb + r = a$. The two pairs therefore have exactly the same set of common divisors, hence the same greatest one.

**Why it is fast.** Each step replaces $(a, b)$ by $(b, r)$ with $r < b$, and in fact $r < a/2$ whenever $b \le a$ - so the pair shrinks geometrically. The worst case is consecutive Fibonacci numbers, giving $O(\log_\varphi \min(a,b))$ steps: about $90$ steps for $10^{18}$.

```cpp
long long gcdEuclid(long long a, long long b) {
    while (b != 0) {
        long long r = a % b;
        a = b;
        b = r;
    }
    return a;                     // gcd(a, 0) = a
}
```

Two practical notes:

- C++17 provides `std::gcd` in `<numeric>` and GCC provides `__gcd`. Both return a value of the argument type, and both are fine for non-negative inputs. `std::gcd` requires C++17 library support; GCC 6.3 (MinGW) predates it in some builds, so `__gcd` or your own function is the portable choice.
- `gcd` is defined up to sign. Take absolute values at the start if inputs can be negative; otherwise `gcd(-4, 6)` may return `-2` depending on implementation.

---

## 10. Solved Problem 3 — The Two Gear Wheels

> **Statement.** A machine has two gear wheels meshing together, with $a$ and $b$ teeth. One tooth on each wheel is painted red. A mechanic has two questions. First: if she wants to divide both wheels into the same number of equal sections - each section containing a whole number of teeth on both wheels - what is the largest number of sections she can make? Second: letting the wheels turn together from the moment the two red teeth are touching, after how many teeth of the first wheel have passed the red teeth touch again? Tooth counts are up to $10^{9}$.

<details>
<summary>Solution</summary>

### Why this approach

The first question is the greatest common divisor. Suppose both wheels are cut into the same number $N$ of equal sections, each section containing a whole number of teeth. Then $N$ divides $a$ and $N$ divides $b$, so $N$ divides $\gcd(a,b)$; the largest possible $N$ is $\gcd(a,b)$, and each section then holds $a/N$ and $b/N$ teeth respectively.

The second is the least common multiple: positions align after $k$ teeth of the first wheel pass, where $k$ is a multiple of $a$ and of $b$. The first such $k$ is $\operatorname{lcm}(a,b)$.

So the problem is a single line of number theory, and the entire difficulty is doing the lcm without overflow: $a \cdot b$ reaches $10^{18}$ at $a = b = 10^{9}$, which *just* fits, but $a = b = 2 \times 10^{9}$ would not, and the habit is what matters.

### Solution

Use $\operatorname{lcm}(a,b) = a / \gcd(a,b) \cdot b$, dividing **before** multiplying. The division is exact because $\gcd(a,b)$ divides $a$.

```cpp
#include <bits/stdc++.h>
using namespace std;

long long gcdEuclid(long long a, long long b) {
    while (b != 0) {
        long long r = a % b;
        a = b;
        b = r;
    }
    return a;
}

int main() {
    long long a, b;
    cin >> a >> b;

    long long g = gcdEuclid(a, b);
    long long l = a / g * b;      // divide first: no overflow at 1e9 * 1e9

    cout << g << ' ' << l << '\n';
    return 0;
}
```

**Input**

```text
12 18
```

**Output**

```text
6 36
```

Six sections of 2 and 3 teeth respectively, and the red teeth meet again after 36 teeth of the first wheel: three full turns of the 12-wheel and two of the 18-wheel.

**Complexity.** $O(\log \min(a,b))$ time, $O(1)$ memory.

### The trap

Writing `a * b / g`. It is mathematically identical and fails in `long long` exactly when $a \cdot b > 9.22 \times 10^{18}$ - that is, for many pairs the statement allows. The error is invisible in the sample, which is why the sample is not a test. Divide first, then multiply, and check the result against `LLONG_MAX / b` if the statement's bound is near the edge.

A second trap: if the statement asks for an lcm of a whole array, the lcm grows astronomically - often past $10^{18}$ after five or six elements. Most such problems ask for it modulo something, which requires prime exponents rather than the lcm itself; that is a Level 3 twist covered in the unsolved set.

</details>

---

## 11. Extended Euclid and the Bézout identity

The Euclidean algorithm computes $\gcd(a,b)$. The **extended** version also produces integers $x, y$ with

$$ax + by = \gcd(a, b).$$

This is Bézout's identity, and its coefficients may be negative - that is not a defect, it is the point: "three steps forward, two steps back" is how you land on $1$ when the step sizes are coprime.

The recursion is the Euclidean algorithm read backwards. At the base, $\gcd(a,0) = a$ is witnessed by $1 \cdot a + 0 \cdot 0$. Then, from $b x_1 + (a \bmod b) y_1 = g$, substitute $a \bmod b = a - \lfloor a/b \rfloor b$ and collect terms:

$$g = y_1 a + \left(x_1 - \left\lfloor a/b \right\rfloor y_1\right) b .$$

```cpp
// Returns gcd(a,b) and sets x, y with a*x + b*y = gcd(a,b).
long long extGcd(long long a, long long b, long long &x, long long &y) {
    if (b == 0) { x = 1; y = 0; return a; }
    long long x1, y1;
    long long g = extGcd(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}
```

**Why you should care now.** Three things in this book depend on it:

- **Modular inverse.** When $\gcd(a, m) = 1$, the $x$ from $ax + my = 1$ satisfies $ax \equiv 1 \pmod m$. That is the inverse, and it exists *only* when the gcd is $1$. This is Topic 02's core.
- **Linear Diophantine equations.** $ax + by = c$ has solutions exactly when $\gcd(a,b) \mid c$, and then all solutions are $x + k\,b/g$, $y - k\,a/g$.
- **Chinese remainder theorem.** Two congruences are combined by constructing the inverse of one modulus modulo the other.

**Coefficient size.** The coefficients stay bounded by about $\max(a,b)$, so `long long` is safe for inputs up to $10^{9}$. Do not feed it $10^{18}$ and expect the intermediate `(a / b) * y1` to survive; that product can overflow. If you need huge inputs, widen the intermediates to a 128-bit type (`__int128` on 64-bit GCC and Clang; it does not exist on 32-bit MinGW builds) or restructure the multiplication.

---

## 12. Solved Problem 4 — The Merchant's Balance

> **Statement.** A merchant owns a balance scale and can place any number of copies of two weights, of sizes $a$ and $b$ grams, on either pan. She has discovered that the smallest positive weight difference she can ever produce is $g$. For the customs ledger she must write a recipe: how many copies of the $a$-gram weight to put on the left pan, how many copies of the $b$-gram weight to put on the right pan, and the resulting difference. A negative count means that weight goes on the other pan. The ledger is checked by machine, so any valid recipe is accepted. Weight sizes are up to $10^{9}$.

<details>
<summary>Solution</summary>

### Why this approach

Achievable differences are exactly the integers $ax - by$ with $x, y \ge 0$. Allowing the sign to flip, the set of values is $\{ax + by\}$, which by Bézout is precisely the multiples of $\gcd(a,b)$. The smallest positive one is the gcd itself.

So the merchant's question is the Bézout identity, and the "recipe" is a pair $(x, y)$ with $ax + by = g$, printed in the statement's order: count of $a$, count of $b$, then $g$.

### Solution

The statement's odd convention - a negative count means "the other pan" - is exactly what makes the answer printable: Bézout coefficients are signed, so no conversion is needed.

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
    long long a, b;
    cin >> a >> b;

    long long x, y;
    long long g = extGcd(a, b, x, y);

    cout << x << ' ' << y << ' ' << g << '\n';
    return 0;
}
```

**Input**

```text
30 18
```

**Output**

```text
-1 2 6
```

Read it as: one 30-gram weight on the right pan, two 18-gram weights on the left, difference $36 - 30 = 6$. Check: $30 \cdot (-1) + 18 \cdot 2 = 6 = \gcd(30,18)$, and indeed six is the smallest positive difference obtainable, since two weights of 30 and 18 share a factor of 6.

**Complexity.** $O(\log \min(a,b))$ time and recursion depth, $O(1)$ memory.

### The trap

The coefficients are **not unique**. For $30$ and $18$, $(x,y) = (2, -3)$ also works: $60 - 54 = 6$. Any correct pair passes, but a hand-derived expected answer in a book or a debug print will differ from your program's - and that difference is not a bug. This is the reason the statement says "any valid recipe is accepted": judges use a special checker here.

The second trap is the base case order. If you write `if (a == 0) { ... }` instead of `if (b == 0)`, the identity still holds but the coefficient signs flip, which matters when you compute a modular inverse in the next topic (you would return $-a^{-1}$ and get wrong answers only for some moduli). Standardise on the `b == 0` base case and never think about it again.

</details>

---

## 13. LCM, coprimality and the growth of lcm

The least common multiple is the smallest positive integer divisible by both arguments:

$$\operatorname{lcm}(a,b) = \frac{a \cdot b}{\gcd(a,b)} = \frac{a}{\gcd(a,b)} \cdot b .$$

Two facts worth memorising:

- $\gcd(a,b) \cdot \operatorname{lcm}(a,b) = ab$, so computing one gives the other.
- $a$ and $b$ are **coprime** (written $a \perp b$) exactly when $\gcd(a,b) = 1$.

### Coprimality is a statement about prime supports

$a \perp b$ means no prime divides both - their prime sets are disjoint. This is why coprime conditions are so often "no common letter", "no shared factor", "different prime supports"; and why bitmask techniques over primes appear in Level 3 problems.

Careful: **pairwise coprime** is much stronger than "gcd of all of them is 1". The numbers $6, 10, 15$ have overall gcd $1$, but every pair shares a factor. Statements that say "any two of them are coprime" are giving you a strong condition that you should exploit; statements that say "their gcd is 1" give you almost nothing.

### lcm grows absurdly fast

Landau's result is $\operatorname{lcm}(1, 2, \dots, n) = e^{n(1 + o(1))}$. Numerically, $\operatorname{lcm}(1, 2, \dots, 43)$ is already larger than $10^{18}$. Two consequences:

- An lcm of an array of numbers around $10^{9}$ overflows almost immediately; if a statement asks for it, it either bounds the answer or asks for it modulo something.
- To compute an lcm **modulo** $M$ you cannot work with residues: the division by the gcd is not well defined modulo $M$. You must track prime exponents (take, for each prime, the maximum exponent across the array) and only then exponentiate. That is the standard Level 3 trap.

A safe array-lcm loop, with an overflow guard, looks like this:

```cpp
const long long LIM = (long long)1e18 + 0;      // whatever the statement promises

long long lcmArray(const vector<long long>& v, bool &ok) {
    long long l = 1;
    for (long long x : v) {
        long long g = gcdEuclid(l, x);
        if (l / g > LIM / x) { ok = false; return 0; }   // l * x would pass LIM
        l = l / g * x;
    }
    ok = true;
    return l;
}
```

The guard must be written as `l / g > LIM / x`, not as `l / g * x > LIM` - the latter computes the overflowing product first, which is precisely the thing you are trying to avoid.

---

## 14. Euler's totient: counting the units

Euler's totient function $\varphi(n)$ counts the integers in $[1, n]$ that are coprime to $n$:

$$\varphi(n) = \#\{k : 1 \le k \le n,\ \gcd(k, n) = 1\}.$$

For a prime $p$, every $k < p$ is coprime, so $\varphi(p) = p - 1$. For a prime power, $\varphi(p^e) = p^e - p^{e-1}$, since exactly the multiples of $p$ fail. And because coprimality depends only on prime supports, $\varphi$ is multiplicative over coprime arguments, giving the master formula

$$\varphi(n) = n \prod_{p \mid n} \left(1 - \frac{1}{p}\right).$$

For $n = 36 = 2^2 \cdot 3^2$: $\varphi(36) = 36 \cdot \tfrac{1}{2} \cdot \tfrac{2}{3} = 12$. (Check: the 12 numbers are those not divisible by 2 or 3.)

```cpp
// Euler's totient of a single n, by factorisation. O(sqrt(n)).
long long phi(long long n) {
    long long result = n;
    for (long long p = 2; p <= n / p; ++p) {
        if (n % p != 0) continue;
        while (n % p == 0) n /= p;
        result -= result / p;          // multiply by (1 - 1/p) without fractions
    }
    if (n > 1) result -= result / n;
    return result;
}
```

Note the arithmetic `result -= result / p`: it applies the factor $(1 - 1/p)$ using integer division only, and it is exact because $p$ divides `result` at that moment. Never write $n \cdot (p-1) / p$ in floating point.

**Why it matters beyond counting.** $\varphi(n)$ is the size of the multiplicative group modulo $n$, which is why it appears in Euler's theorem $a^{\varphi(n)} \equiv 1 \pmod n$ for $a \perp n$, and why modular inverses exist exactly for the $\varphi(n)$ residues coprime to $n$. Topic 02 leans on this heavily.

---

## 15. Solved Problem 5 — The Lighthouse Slate

> **Statement.** A lighthouse keeper keeps herself awake by writing fractions on a slate. She writes every fraction $p/q$ with $1 \le p \le q \le n$, but only in lowest terms - she refuses to write the same value twice, and she reduces anything reducible. When she is done she counts the entries. How many does she write? The number $n$ can be up to one million.

<details>
<summary>Solution</summary>

### Why this approach

For a fixed denominator $q$, the numerators $p$ with $1 \le p \le q$ and $\gcd(p,q) = 1$ are exactly $\varphi(q)$ of them. The fraction $1$ appears as $1/1$; every other value appears once, with a unique reduced denominator. So the slate holds

$$\sum_{q=1}^{n} \varphi(q).$$

A single $\varphi$ evaluation costs $O(\sqrt{n})$; doing that a million times is $10^{9}$ operations - too slow. The values are needed for *every* $q$ up to $n$, so sieve them: initialise $\varphi(q) = q$ and, for each prime $p$, multiply every multiple by $(1 - 1/p)$. That is the same structure as the sieve of Eratosthenes.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> phi(n + 1);
    for (int i = 0; i <= n; i++) phi[i] = i;

    for (int p = 2; p <= n; p++) {
        if (phi[p] != p) continue;               // p is composite
        for (int m = p; m <= n; m += p)
            phi[m] -= phi[m] / p;                // multiply by (1 - 1/p)
    }

    long long total = 0;
    for (int i = 1; i <= n; i++) total += phi[i];

    cout << total << '\n';
    return 0;
}
```

**Input**

```text
10
```

**Output**

```text
32
```

For $q = 1 \dots 10$, the totients are $1, 1, 2, 2, 4, 2, 6, 4, 6, 4$, which sum to $32$. The denominators do the work: $1/1$; then $1/2$; $1/3, 2/3$; $1/4, 3/4$; four fifths; and so on.

**Complexity.** $O(n \log \log n)$ time, $O(n)$ memory. The sum over primes of $n/p$ is $n \ln \ln n$, and the prefix sum afterwards is linear.

### The trap

The sieve's primality test is `phi[p] != p`. That works only because every composite $p$ has already been touched by a smaller prime factor, so its value dropped below $p$. If you instead initialise the array to $1$ or to $0$, the test silently mis-fires and you sieve with composites too - the result is still correct (composites would apply redundant factors) but slower, or incorrect if you use the wrong test.

The other trap is overflow of the **sum**: $\sum_{q \le 10^{6}} \varphi(q) \approx \frac{3}{\pi^2} n^2 \approx 3 \times 10^{11}$, which needs `long long`. The individual totients fit in `int`, the sum does not.

</details>

---

## 16. The identity $\sum_{d \mid n} \varphi(d) = n$, and what it buys you

There is a second way to count the fractions with denominator exactly $n$: the fractions $k/n$ for $k = 1 \dots n$, before reducing. Grouping them by the reduced denominator $d$ gives

$$n = \sum_{d \mid n} \varphi(d).$$

Two useful consequences.

**Counting coprime pairs.** The number of pairs $(a,b)$ with $1 \le a \le b \le n$ and $\gcd(a,b)=1$ is exactly $\sum_{b \le n} \varphi(b)$, because for each $b$ there are $\varphi(b)$ valid numerators. For $1 \le a < b \le n$ it is $\sum_{b \le n} \varphi(b) - 1$, the subtraction removing the pair $(1,1)$, which $\varphi(1)$ counts but which violates $a < b$. That single subtraction is a favourite of setters, and it is easy to forget.

**The gcd-sum identity.** For fixed $n$,

$$\sum_{i=1}^{n} \gcd(i, n) = \sum_{d \mid n} d \cdot \varphi\!\left(\frac{n}{d}\right) = \sum_{d \mid n} \varphi(d) \cdot \frac{n}{d}.$$

Proof: group the $i$ in $[1,n]$ by $\gcd(i,n) = d$. Dividing by $d$ turns that into counting $j$ with $\gcd(j, n/d) = 1$, of which there are $\varphi(n/d)$.

With a totient sieve up to $n$ and the divisors of $n$, this answers "sum of gcds" in $O(\sqrt{n})$ per query after an $O(n \log \log n)$ precomputation - the intended solution of several Div. 2 problems, including one in this topic's unsolved set.

The same grouping argument is your first taste of **Möbius inversion**, which generalises all of this. If you have seen the USACO problem that asks for a Dirichlet inverse of a function supported on floor-quotients, this identity is the small version of it: count by the gcd or by the reduced denominator, then subtract.

---

## 17. Choosing a tool: bounds and complexity

| Task | Constraint | Tool | Cost |
|---|---|---|---|
| primes up to $n$ | $n \le 10^{7}$ | sieve of Eratosthenes | $O(n \log \log n)$ |
| primes up to $n$ | $n \le 10^{9}$ | segmented sieve | $O(n \log \log n)$ with small memory |
| is $n$ prime? | $n \le 10^{12}$, one query | trial division to $\sqrt{n}$ | $O(\sqrt{n})$ |
| factorise $n$ | $n \le 10^{12}$, one query | trial division (wheel) | $O(\sqrt{n})$ |
| factorise many $x_i$ | $x_i \le 10^{6}$, $q \le 2 \cdot 10^{5}$ | smallest-prime-factor sieve | $O(N \log \log N + q \log N)$ |
| $\tau(n)$ | $n \le 10^{12}$ | factorisation formula | $O(\sqrt{n})$ |
| $\gcd$ of two | any 64-bit | Euclidean algorithm | $O(\log n)$ |
| $\gcd$ of many | any 64-bit | fold Euclidean | $O(n \log n)$ |
| Bézout coefficients | $\lvert a \rvert, \lvert b \rvert \le 10^{9}$ | extended Euclid | $O(\log n)$ |
| $\operatorname{lcm}$ of array | answer $\le 10^{18}$ | fold with overflow guard | $O(n \log n)$ |
| $\operatorname{lcm}$ of array, mod $M$ | any | track prime exponents | $O(n \sqrt{\max a})$ or sieve |
| $\varphi(n)$ | $n \le 10^{12}$, one query | factorisation | $O(\sqrt{n})$ |
| $\varphi(1..n)$ | $n \le 10^{7}$ | totient sieve | $O(n \log \log n)$ |
| $\sum_{d \mid n} \varphi(d)$ | any | it equals $n$ | $O(1)$ |

Read the table as a decision procedure: the constraint column tells you which row you are in, and the row tells you the algorithm. Almost every "wrong answer, time limit exceeded" in this topic is a row mismatch - a sieve attempted where only trial division fits, or trial division attempted where the query count forbids it.

---

## 18. Unsolved problems

Six problems. The statements are written the way a judge writes them: nothing tells you which row of the table above applies, and in two of them the obvious reading is a trap. Work out the quantity first, then pick the tool.

### 18.1 The Inventory Audit

> A warehouse stores $n$ identical machine parts, and the auditor wants two numbers about the storage plan: how many ways the parts can be split into equal groups of size at least $1$ (groupings that differ only in the order of the groups are the same), and how many parts a single container holds in the largest plan that uses more than one container. The number of parts can be as large as $10^{12}$.

### 18.2 The Consignment Forms

> A customs office processes consignments in batches. Each consignment is identified by an integer $x$, and for every consignment the clerk fills two boxes: the smallest prime that divides $x$, and the number of distinct primes that divide $x$. There can be $2 \times 10^{5}$ consignments, each with $2 \le x \le 10^{6}$.

### 18.3 The Bead Necklace

> A jeweller has $n$ identical beads and wants to lay them out in a rectangle with at least two rows and at least two columns, using every bead. Two layouts that differ only by a quarter turn of the page are the same layout. How many layouts are possible? The order can hold up to $10^{12}$ beads.

### 18.4 The Staircase of Fractions

> A mathematics teacher writes on the board every fraction $a/b$ with $1 \le a < b \le n$ that cannot be reduced. She then wants to know how many such fractions there are, and, separately, the value of $\sum_{i=1}^{n} \gcd(i, n)$ for the same $n$. Both answers fit in a signed 64-bit integer. The bound is $n \le 10^{7}$, and the teacher may ask this for many values of $n$ in one session - up to $10^{5}$ of them, each at most $10^{7}$.

### 18.5 The Assembly Line

> A factory's components come in lengths given by an array of $m$ positive integers, each at most $10^{9}$. A batch can be assembled only if all components fit into a single repeating cycle, so the foreman needs the smallest cycle length that is a multiple of every length in the array. If that number exceeds $10^{18}$, he reports $-1$ instead; the factory never needs such a batch. The array can have up to $2 \times 10^{5}$ entries.

### 18.6 The Calendar Wheel

> A printer must cut a sheet whose size is the smallest number divisible by every integer from $1$ to $x$, where $x$ is the number of markings on a ceremonial wheel. Given a target $N$ up to $10^{18}$, find the smallest $x$ whose sheet size reaches at least $N$. Report $-1$ if that would require $x > 100$.

---

## Key Takeaways

1. Nearly every elementary number theory statement asks for one of three things: a divisor count, a gcd, or a totient. Translate the story into one of the three before writing any code.
2. $\tau(n) = \prod (e_i + 1)$ and $\sigma(n) = \prod \frac{p^{e_i+1}-1}{p-1}$ come straight from unique factorisation; you rarely need to enumerate divisors.
3. The loop condition is `p <= n / p`, never `p * p <= n`, when $n$ can approach $10^{18}$.
4. A sieve is an array indexed by value, so it needs the value bound to be small; trial division needs $\sqrt{n}$ per query. Read the constraint to know which one the setter intends.
5. $\gcd(a,b) = \gcd(b, a \bmod b)$ is fast because the pair at least halves every two steps, and the worst case is Fibonacci.
6. $\operatorname{lcm} = a/\gcd(a,b) \cdot b$: divide first. For arrays, guard with `l / g > LIMIT / x`.
7. Extended Euclid gives $ax + by = g$, which is exactly a modular inverse when $g = 1$; the coefficients are not unique, so never compare them against a hand-computed answer.
8. $\varphi(n) = n \prod_{p \mid n}(1 - 1/p)$, computed in integers by `result -= result / p`; sieve it when you need all values up to $n$.
9. $\sum_{d \mid n} \varphi(d) = n$ and $\sum_{i \le n} \gcd(i,n) = \sum_{d \mid n} d\,\varphi(n/d)$ are the two grouping identities worth knowing cold.
10. Off-by-one errors in coprime-pair counting (the pairs $(1,1)$ and $(1,b)$) cost more submissions than the arithmetic does.

---

## Common Mistakes & Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `p * p <= n` with `int p`, $n$ near $10^{18}$ | passes samples, fails the largest test | `p <= n / p` |
| `a * b / g` for lcm | wrong sign/negative values at large inputs | `a / g * b`, plus a guard if bounded |
| treating `%` as always non-negative | negative shifts in difference/counting tasks | normalise with `(r + m) % m` |
| counting $1$ as prime in a sieve | answer off by one on every test | start at $2$, or special-case $\pi(1) = 0$ |
| crossing out multiples from $2p$ | correct but 2x slower, TLE at $10^{7}$-scale | start at $p^2$ |
| forgetting the leftover prime after the loop | misses the largest factor, e.g. $14 \to 2$ only | `if (n > 1) push(n, 1)` |
| sieving with `char` at $10^{8}$ | memory limit exceeded | `vector<bool>`, or odd-only sieve |
| `list.spf` sized $10^{8}$ as `int` | 400 MB allocation | bound at $10^{7}$, or segmented approach |
| recomputing $\varphi$ per query up to $10^{7}$ | TLE on $10^{5}$ queries | precompute the totient sieve once |
| `sum += phi[i]` in `int` | overflow around $n = 10^{5}$ | `long long` accumulator |
| assuming the Bézout pair is unique | "expected `2 -3 6`, got `-1 2 6`" | any valid pair is correct; use a checker |
| `std::gcd` on a pre-C++17 library | compile error on MinGW | `__gcd` or write your own |

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Locked Drawers | $\pi(n)$ by sieve | §4 |
| 2 | The Chocolate Tray | $\tau(n)$ by factorisation | §7 |
| 3 | The Two Gear Wheels | $\gcd$ and overflow-safe $\operatorname{lcm}$ | §10 |
| 4 | The Merchant's Balance | Bézout coefficients | §12 |
| 5 | The Lighthouse Slate | prefix sums of $\varphi$ | §15 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Inventory Audit | $n \le 10^{12}$ | factorisation, then two derived quantities |
| 2 | The Consignment Forms | $q \le 2 \cdot 10^{5}$, $x \le 10^{6}$ | sieve once, answer per query in $O(\log x)$ |
| 3 | The Bead Necklace | $n \le 10^{12}$ | $\tau(n)$ minus the degenerate layouts, square case |
| 4 | The Staircase of Fractions | $n \le 10^{7}$, many queries | precomputation, then $O(\sqrt{n})$ per query |
| 5 | The Assembly Line | $m \le 2 \cdot 10^{5}$, values $\le 10^{9}$ | fold with an overflow guard, then a second pass |
| 6 | The Calendar Wheel | $N \le 10^{18}$, $x \le 100$ | increasing sequence, no search needed |

If you can classify all six into the table in §17 without opening a compiler, you have the reading skill this topic is really about. The code follows in minutes; the classification is the contest.

---

## Next Up

Topic 02 takes the same arithmetic into the **modular world**, where division becomes multiplication by an inverse and exponents are computed in $O(\log e)$. Everything here reappears there: you will need gcd to know when an inverse exists, extended Euclid to compute it, and the totient to know how many inverses a modulus has. The natural bridge problem is the Bézout identity you just solved - the moment you set the right-hand side to $1$, it becomes an inverse.

