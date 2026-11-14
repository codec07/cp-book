# Topic 02 — Time Complexity (Basics to Advanced)

Every contest problem has a time limit, and every solution has a cost. Complexity analysis is the bridge: given $n = 2 \times 10^5$ and a 2-second limit, can your $\mathcal{O}(n \log n)$ idea pass, and would $\mathcal{O}(n^2)$ die? Getting this right on paper — before you type — is the difference between a 20-minute solve and a 40-minute TLE-and-retry.

This is the most load-bearing topic in the book. Almost every TLE you will ever receive was a complexity mistake you could have caught in thirty seconds of arithmetic.

> **Prerequisites:** [Topic 01 — Introduction to CP and ICPC](Topic%2001%20-%20Introduction%20to%20CP%20and%20ICPC.md), [Chapter 1, Topic 07 — Loops](../CPP%20for%20CP/Topic%2007%20-%20Loops%20&%20'Auto'.md)
> **You will be able to:**
> - Read any loop nest and state its complexity in under a minute.
> - Use the $10^8$ rule to predict pass/fail before writing code.
> - Analyse recursive functions with a recurrence, including the Master Theorem.
> - Distinguish amortised, average and worst-case cost, and know which one matters.
> - Recognise the hidden costs: string copies, `map` lookups, cache misses, I/O.
> - Budget memory as rigorously as you budget time.

---

## 1. Why complexity is the whole game

A judge runs your program on hidden tests with a time limit, typically **1–3 seconds**. Your program is not "fast" or "slow" in the abstract — it either fits in that window for the largest test, or it does not. Nothing else matters.

The trap is that your machine lies to you. A solution that runs in 0.3 s locally on $n = 1000$ may take 300 s on the judge's $n = 10^5$ test. Growth is not intuitive:

| $n$ | $\log_2 n$ | $n$ | $n \log_2 n$ | $n^2$ | $2^n$ |
|---|---|---|---|---|---|
| 10 | 3 | 10 | 33 | 100 | 1,024 |
| 100 | 7 | 100 | 664 | 10,000 | $1.3 \times 10^{30}$ |
| 1,000 | 10 | 1,000 | 9,966 | $10^6$ | — |
| $10^5$ | 17 | $10^5$ | $1.7 \times 10^6$ | $10^{10}$ | — |
| $10^6$ | 20 | $10^6$ | $2 \times 10^7$ | $10^{12}$ | — |

Look at the $n^2$ column: at $n = 10^5$ it is $10^{10}$. Even at a billion operations per second that is ten seconds. This is the entire reason $\mathcal{O}(n \log n)$ matters.

---

## 2. The $10^8$ rule

A modern CPU does roughly $10^8$ to $5 \times 10^8$ **simple operations** per second in compiled C++ with `-O2`. A "simple operation" is an addition, a comparison, an array read. Judges are usually a few years behind consumer hardware, so budget conservatively:

$$\boxed{\text{1 second} \approx 10^8 \text{ simple operations}}$$

| Operations | Realistic time (C++ `-O2`) | Verdict on a 1 s limit |
|---|---|---|
| $10^6$ | ~1–3 ms | trivially fine |
| $10^7$ | ~10–30 ms | fine |
| $10^8$ | ~100–300 ms | fine |
| $5 \times 10^8$ | ~0.5–1.5 s | borderline |
| $10^9$ | ~1–3 s | likely TLE |
| $10^{10}$ | ~10–30 s | certain TLE |

Three important caveats:

1. **"Simple" is doing a lot of work.** An array read is simple. A `map` lookup is a tree descent with cache misses — roughly 20–50 simple operations. A `std::string` comparison is $\mathcal{O}(\text{length})$. The rule applies to the *inner* loop body, whatever it contains.
2. **The constant factor can be 10x either way.** A tight loop summing an array of `int` runs ~4x faster than the same loop over a `vector<vector<int>>`. See §12.
3. **Time limits scale with the language.** Some judges give Python 3x the C++ limit. Assume the C++ limit when designing.

*Contest story: an $\mathcal{O}(n \log n)$ solution with $n = 10^6$ was estimated at $2 \times 10^7$ operations and should have run in 30 ms. It TLE'd at 2.1 s, because the "operation" was a `std::map` insert — $\log n$ tree descents with pointer chasing, ~40 ns each. The complexity was right and the model was wrong. Counting operations means counting the expensive ones.*

---

## 3. Big-O, properly

Formally, $f(n) = \mathcal{O}(g(n))$ if there exist constants $c > 0$ and $n_0$ such that

$$f(n) \le c \cdot g(n) \quad \text{for all } n \ge n_0$$

In words: beyond some size, $f$ is bounded above by a constant multiple of $g$. That is all. Big-O is an **upper bound** and it deliberately throws away constants and lower-order terms.

| Notation | Meaning | Analogy |
|---|---|---|
| $\mathcal{O}(g)$ | $f$ grows **no faster** than $g$ | $\le$ |
| $\Omega(g)$ | $f$ grows **at least as fast** as $g$ | $\ge$ |
| $\Theta(g)$ | $f$ grows **exactly like** $g$ | $=$ |

Why drop constants? Because $100n$ and $3n$ are both "linear", and the difference between them is a factor that hardware, compiler flags and cache behaviour can swamp. The growth class predicts what happens when $n$ doubles; the constant predicts what happens at a fixed $n$. **Both matter in practice**, and §12 is about the constant.

Why drop lower-order terms? Because $n^2 + 100n + 5000$ is dominated by $n^2$ for large $n$. At $n = 10^5$, $n^2$ is $10^{10}$ and $100n$ is $10^7$ — a thousandth of the total.

> **A note on sloppy notation.** Contest write-ups say "$\mathcal{O}(n)$" when they mean $\Theta(n)$ — a tight bound. Nobody writes $\Theta$. Just remember that when you read "this is $\mathcal{O}(n)$", the author usually means "this is linear" and not "this is at most linear".

---

## 4. The complexity zoo

Ordered from cheapest to most expensive. Learn the shape of each.

| Class | Name | $n$ for 1 s | Example |
|---|---|---|---|
| $\mathcal{O}(1)$ | constant | any | Array index, hash lookup |
| $\mathcal{O}(\log n)$ | logarithmic | $10^{18}$ | Binary search, `lower_bound` |
| $\mathcal{O}(\log^2 n)$ | log-squared | $10^{18}$ | Binary search on a segment tree |
| $\mathcal{O}(\sqrt n)$ | square root | $10^{14}$ | Trial division, sqrt decomposition block |
| $\mathcal{O}(n^{1/3})$ | cube root | $10^{24}$ | Integer cube root |
| $\mathcal{O}(n)$ | linear | $10^8$ | Single pass, prefix sums |
| $\mathcal{O}(n \log n)$ | linearithmic | $10^7$ | Sorting, merge sort, `set` build |
| $\mathcal{O}(n \log^2 n)$ | | $10^6$ | Nested binary searches |
| $\mathcal{O}(n \sqrt n)$ | | $10^5$ | Mo's algorithm |
| $\mathcal{O}(n^2)$ | quadratic | $10^4$ | All pairs, naive DP |
| $\mathcal{O}(n^3)$ | cubic | $500$ | Floyd–Warshall, interval DP |
| $\mathcal{O}(n^4)$ | quartic | $100$ | 4D DP |
| $\mathcal{O}(2^n)$ | exponential | $25$ | Subset enumeration |
| $\mathcal{O}(n!)$ | factorial | $11$ | Permutations |
| $\mathcal{O}(n^n)$ | | $8$ | Very rare |

The jump from polynomial to exponential is the wall. Do not expect to push past it with optimisation — you need a different algorithm.

### Visualising growth

```
ops
 ^
 |                                                    n!
 |                                              /  2^n
 |                                     /
 |                            /    n^3
 |                     /    n^2
 |              /    n log n
 |        /   n
 |    /  log n
 +----------------------------------------> n
```

Logarithmic growth is so flat it looks like a horizontal line. That is why "binary search on the answer" turns an impossible problem into an easy one: $\log_2(10^{18}) = 60$ steps.

---

## 5. Counting operations in code

Four rules cover almost everything.

### Rule 1 — Sequential blocks add

```cpp
for (int i = 0; i < n; i++) sum += a[i];        // O(n)
for (int i = 0; i < n; i++) prod *= b[i];       // O(n)
```

Total $\mathcal{O}(n) + \mathcal{O}(n) = \mathcal{O}(n)$. Adding two linear passes is still linear.

### Rule 2 — Nested loops multiply

```cpp
for (int i = 0; i < n; i++)          // n iterations
    for (int j = 0; j < m; j++)      // m iterations each
        c[i][j] = a[i] + b[j];       // O(n*m)
```

With both equal to $n$: $\mathcal{O}(n^2)$.

### Rule 3 — Halving gives a logarithm

```cpp
while (n > 0) {
    n /= 2;                          // O(log n) iterations
}
```

Each step halves the remaining work, so the loop runs $\lceil \log_2 n \rceil$ times. This is the signature of binary search, and of any "divide by 2" or "shift right" loop.

### Rule 4 — Dependent bounds need summing

```cpp
for (int i = 0; i < n; i++)
    for (int j = i; j < n; j++)
        work();
```

The inner loop runs $n - i$ times. The total is

$$\sum_{i=0}^{n-1} (n - i) = n + (n-1) + \cdots + 1 = \frac{n(n+1)}{2} = \mathcal{O}(n^2)$$

Half of $n^2$ is still $n^2$. Do not be fooled by the triangle.

### The one that surprises people: harmonic sums

```cpp
for (int i = 1; i <= n; i++)
    for (int j = i; j <= n; j += i)
        work();
```

The inner loop runs $\lfloor n/i \rfloor$ times, so the total is

$$\sum_{i=1}^{n} \frac{n}{i} = n \cdot H_n = n\left(1 + \tfrac12 + \tfrac13 + \cdots + \tfrac1n\right) \approx n \ln n$$

That is $\mathcal{O}(n \log n)$, not $\mathcal{O}(n^2)$. This is the divisor-sieve pattern — iterating over multiples of $i$ — and it is why the sieve of Eratosthenes is $n \log \log n$ rather than $n^2$.

| Loop nest | Total iterations | Complexity |
|---|---|---|
| `i<n`, `j<n` | $n^2$ | $\mathcal{O}(n^2)$ |
| `i<n`, `j=i..n` | $n(n+1)/2$ | $\mathcal{O}(n^2)$ |
| `i<n`, `j=i..n`, `j+=i` | $n H_n$ | $\mathcal{O}(n \log n)$ |
| `i<n`, `j=0..i*i` | $n^3/3$ | $\mathcal{O}(n^3)$ |
| `i<n`, `i*=2` | $\log n$ | $\mathcal{O}(\log n)$ |
| `i<n`, `i++`, inner halves | $n \log n$ | $\mathcal{O}(n \log n)$ |

---

## 6. Analysing six real snippets

### 6.1 Two pointers on a sorted array

```cpp
int i = 0, j = n - 1;
while (i < j) {
    int s = a[i] + a[j];
    if (s == target) break;
    if (s < target) i++; else j--;
}
```

Not $\mathcal{O}(n^2)$ despite the two moving indices. Each iteration moves one pointer inward, and the gap `j - i` decreases by exactly 1. Total iterations $\le n$. **$\mathcal{O}(n)$.**

The key insight: a `while` loop's cost is the number of iterations, not the number of variables.

### 6.2 Sieve of Eratosthenes

```cpp
vector<bool> isPrime(n + 1, true);
for (int i = 2; i * i <= n; i++)
    if (isPrime[i])
        for (int j = i * i; j <= n; j += i)
            isPrime[j] = false;
```

The inner loop runs $n/p$ times for each prime $p \le \sqrt n$. Total is $n \sum_{p \le \sqrt n} 1/p \approx n \log \log n$. **$\mathcal{O}(n \log \log n)$** — effectively linear, which is why a sieve to $10^7$ takes under 100 ms.

### 6.3 GCD

```cpp
long long gcd(long long a, long long b) {
    while (b) { long long t = a % b; a = b; b = t; }
    return a;
}
```

Euclid's algorithm — one of the oldest in existence. Each step at least halves the smaller number, so **$\mathcal{O}(\log \min(a,b))$**. In practice the constant is tiny: about 45 steps for 64-bit inputs.

### 6.4 A function called inside a loop

```cpp
for (int i = 0; i < n; i++)
    total += f(a[i]);
```

The complexity is $n \times \mathcal{O}(f)$. If `f` is $\mathcal{O}(1)$, this is $\mathcal{O}(n)$. If `f` sorts a subarray of length $k$, this is $\mathcal{O}(nk\log k)$. **Always look inside the function.**

A real trap: `s += t` on strings is $\mathcal{O}(\text{len}(s))$, so a loop building a string with `+` is $\mathcal{O}(n^2)$ while `+=` is amortised $\mathcal{O}(n)$.

### 6.5 Recursion without memoisation

```cpp
long long fib(int n) {
    if (n < 2) return n;
    return fib(n-1) + fib(n-2);
}
```

Two calls per level, depth $n$ → $\mathcal{O}(2^n)$. `fib(45)` is about 3.6 billion calls and would take ~10 s. With memoisation it becomes $\mathcal{O}(n)$. This is the entire motivation for dynamic programming.

### 6.6 Sorting with an expensive comparator

```cpp
sort(v.begin(), v.end(), [](const string &a, const string &b) {
    return a < b;                       // O(L) comparison
});
```

`sort` performs $\mathcal{O}(n \log n)$ **comparisons**, but each comparison here is $\mathcal{O}(L)$ where $L$ is the string length. Total $\mathcal{O}(n L \log n)$. For $n = 10^5$ strings of length $10^3$, that is $10^5 \times 10^3 \times 17 = 1.7 \times 10^9$ character operations — and it will TLE.

| Function | What the "unit operation" is | Real cost |
|---|---|---|
| `sort` on `int` | comparison, $\mathcal{O}(1)$ | $n\log n$ |
| `sort` on `string` | comparison, $\mathcal{O}(L)$ | $n L \log n$ |
| `set::insert` on `int` | tree descent, ~20 memory hops | $n \log n$ with a big constant |
| `map::find` on `string` | hash/tree + string compare | $n L \log n$ |

---

## 7. Amortised analysis

Some operations are occasionally expensive and usually cheap. **Amortised** complexity averages the cost over a sequence of operations — not over random inputs, but *guaranteed over any sequence*.

### `vector::push_back`

A vector doubles its capacity when full and copies everything. For $n$ insertions:

$$1 + 2 + 4 + \cdots + n = 2n - 1 \text{ copies total}$$

So $n$ `push_back`s cost $\mathcal{O}(n)$ total → **amortised $\mathcal{O}(1)$ per operation**. The individual worst case is still $\mathcal{O}(n)$.

This is why `reserve(n)` helps: it removes the copying entirely, but it does not change the complexity class.

### Union–Find with path compression and union by size

A sequence of $m$ operations on $n$ elements costs $\mathcal{O}(m \alpha(n))$, where $\alpha$ is the inverse Ackermann function — smaller than 5 for any conceivable input. Effectively $\mathcal{O}(1)$ amortised.

### Two pointers / sliding window

Each pointer only moves forward and never resets. Total moves $\le 2n$ across the whole algorithm, so the pattern is $\mathcal{O}(n)$ even though individual iterations do variable work.

| Structure / technique | Per-operation worst | Amortised |
|---|---|---|
| `vector::push_back` | $\mathcal{O}(n)$ | $\mathcal{O}(1)$ |
| `vector::pop_back` | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ |
| DSU union + path compression | $\mathcal{O}(\log n)$ | $\mathcal{O}(\alpha(n))$ |
| Two pointers | variable | $\mathcal{O}(1)$ per step, $\mathcal{O}(n)$ total |
| `map`/`set` insert | $\mathcal{O}(\log n)$ | $\mathcal{O}(\log n)$ — no amortisation applies |
| `unordered_map` insert | $\mathcal{O}(n)$ (rehash) | $\mathcal{O}(1)$ average |

---

## 8. Space complexity

Memory is a first-class constraint, usually **256 MB**. Note the byte arithmetic:

$$\text{bytes} = N \times \texttt{sizeof(T)} \times \text{number of dimensions or arrays}$$

| Structure | Bytes | Fits 256 MB? |
|---|---|---|
| `int a[10^6]` | 4 MB | yes |
| `int a[10^8]` | 400 MB | **no** |
| `long long a[10^7]` | 80 MB | yes |
| `int dp[5000][5000]` | 100 MB | yes |
| `int dp[10^4][10^4]` | 400 MB | **no** |
| `vector<vector<int>>` of $10^4 \times 10^4$ | 400 MB + overhead | **no** |
| `bool vis[10^7]` | 10 MB | yes |

Two rules:

- A 2D table is **quadratic** in its dimension. $5000 \times 5000$ `int` is the practical ceiling under 256 MB.
- Recursion uses stack memory: depth $d$ with $b$ bytes of locals per frame costs $d \times b$ bytes, against a stack of only ~1–8 MB.

Space and time trade off constantly. Precomputing a table costs memory to save time; recomputing costs time to save memory. The intended solution is chosen so both fit.

---

## 9. Worst, average and expected

| Measure | Meaning | When it applies |
|---|---|---|
| **Worst case** | Maximum over all inputs of size $n$ | What judges test; always your primary bound |
| **Average case** | Mean over a distribution of inputs | Rarely used in CP, distributions are unspecified |
| **Expected** | Mean over the algorithm's own randomness | Randomised algorithms (hashing, treaps) |
| **Amortised** | Total cost of a sequence divided by its length | Data structure operations |

**Judges test the worst case.** If your solution is $\mathcal{O}(n)$ on average and $\mathcal{O}(n^2)$ on adversarial input, a setter will construct the adversarial input. This is the entire basis of the anti-hash attack on `unordered_map` (Chapter 1, Topic 13).

Two things to remember:

- `std::sort` is $\mathcal{O}(n \log n)$ worst case because introsort falls back to heapsort. A naive quicksort with a bad pivot is $\mathcal{O}(n^2)$ — which is why `sort` is not just "quicksort".
- Randomised algorithms need their expected bound to hold against an adversary who does not know your seed. `mt19937` with `chrono::steady_clock` seeding is the standard defence.

---

## 10. Recurrences and the Master Theorem

Recursive functions need a recurrence. Write the cost of one call in terms of smaller calls.

| Algorithm | Recurrence | Solution |
|---|---|---|
| Binary search | $T(n) = T(n/2) + \mathcal{O}(1)$ | $\mathcal{O}(\log n)$ |
| Merge sort | $T(n) = 2T(n/2) + \mathcal{O}(n)$ | $\mathcal{O}(n \log n)$ |
| Naive Fibonacci | $T(n) = T(n-1) + T(n-2)$ | $\mathcal{O}(2^n)$ |
| Binary tree traversal | $T(n) = 2T(n/2) + \mathcal{O}(1)$ | $\mathcal{O}(n)$ |
| Linear scan | $T(n) = T(n-1) + \mathcal{O}(1)$ | $\mathcal{O}(n)$ |
| Selection sort recursive | $T(n) = T(n-1) + \mathcal{O}(n)$ | $\mathcal{O}(n^2)$ |
| Karatsuba multiply | $T(n) = 3T(n/2) + \mathcal{O}(n)$ | $\mathcal{O}(n^{1.585})$ |

### The Master Theorem

For $T(n) = a\,T(n/b) + f(n)$ with $a \ge 1$, $b > 1$, compare $f(n)$ against $n^{\log_b a}$:

| Case | Condition | Result |
|---|---|---|
| 1 | $f(n) = \mathcal{O}(n^{\log_b a - \epsilon})$ | $T(n) = \Theta(n^{\log_b a})$ |
| 2 | $f(n) = \Theta(n^{\log_b a})$ | $T(n) = \Theta(n^{\log_b a} \log n)$ |
| 3 | $f(n) = \Omega(n^{\log_b a + \epsilon})$ and regularity holds | $T(n) = \Theta(f(n))$ |

Applied:

- Merge sort: $a=2$, $b=2$, $f(n)=n$. $n^{\log_2 2} = n^1 = n$. Equal → case 2 → $\Theta(n \log n)$.
- Binary search: $a=1$, $b=2$, $f(n)=1$. $n^{\log_2 1} = n^0 = 1$. Equal → case 2 → $\Theta(\log n)$.
- Karatsuba: $a=3$, $b=2$, $f(n)=n$. $n^{\log_2 3} \approx n^{1.585}$ dominates $n$ → case 1 → $\Theta(n^{1.585})$.

**When the Master Theorem does not apply** (unequal splits, $T(n-1)$ terms), fall back to:

- **Unrolling / substitution:** expand a few levels, spot the pattern, bound the sum.
- **The recursion tree:** draw the cost at each level and sum across levels.
- **Guess and verify:** propose a bound, prove it by induction.

> **Practical shortcut:** recursion depth contributes to stack usage, and the number of calls contributes to time. A DFS over $10^5$ nodes with 100 bytes per frame needs 10 MB of stack — more than the default 8 MB. See [Chapter 1, Topic 10](../CPP%20for%20CP/Topic%2010%20-%20Memory%20Management%20in%20C++%20and%20Pointers.md) for converting recursion to an explicit stack.

---

## 11. The constraint ↔ complexity master table

The most useful table in the book. Read the constraints, read the intended complexity.

| $n$ | Feasible complexity | Technique that usually fits |
|---|---|---|
| $\le 8$ | $\mathcal{O}(n^n)$, $\mathcal{O}(n!)$ | Full permutation / TSP brute force |
| $\le 11$ | $\mathcal{O}(n!)$ | Permutation enumeration ($11! \approx 4 \times 10^7$) |
| $\le 20$ | $\mathcal{O}(2^n \cdot n)$ | Bitmask DP, meet in the middle |
| $\le 25$ | $\mathcal{O}(2^n)$ | Subset enumeration ($2^{25} \approx 3.4 \times 10^7$) |
| $\le 60$ | $\mathcal{O}(n^3)$, $\mathcal{O}(\log n)$ heavy | Matrix exponentiation |
| $\le 100$ | $\mathcal{O}(n^4)$ | 4D DP, brute force with pruning |
| $\le 300$ | $\mathcal{O}(n^3)$ | Floyd–Warshall, interval DP |
| $\le 500$ | $\mathcal{O}(n^3)$ with a small constant | BFS on dense graphs |
| $\le 2000$ | $\mathcal{O}(n^2)$ | Pairwise DP, LCS |
| $\le 5000$ | $\mathcal{O}(n^2)$ tight | DP with 1D memory roll |
| $\le 10^5$ | $\mathcal{O}(n \log n)$, $\mathcal{O}(n\sqrt n)$ | Sort, binary search, segment tree, Mo's |
| $\le 10^6$ | $\mathcal{O}(n \log n)$ or $\mathcal{O}(n)$ | Sieve, prefix sums, two pointers |
| $\le 10^7$ | $\mathcal{O}(n)$ with tiny constant | Single pass, no allocation, fast I/O |
| $\le 10^8$ | $\mathcal{O}(n)$ extremely tight | Rarely intended; usually needs $\mathcal{O}(\log n)$ |
| $\le 10^{18}$ | $\mathcal{O}(\log n)$, $\mathcal{O}(\log^2 n)$ | Binary exponentiation, matrix power, digit DP |
| $\le 10^{1000}$ | $\mathcal{O}(\text{digits})$ | String arithmetic |

### Transposing the table

When stuck, run it backwards:

- The problem says $n \le 2 \cdot 10^5$ → an $\mathcal{O}(n \log n)$ solution exists. What gives me a $\log$ per element? Sorting, binary search, a heap, a balanced BST, a segment tree.
- The problem says $n \le 20$ → the answer is exponential. What is the state? Bitmask over the $n$ items.
- The problem says $n \le 10^{18}$ and asks for a count → the answer is logarithmic or closed-form. Look for binary exponentiation, a matrix recurrence, or digit DP.
- The problem says $n \le 10^9$ but you need to iterate → you cannot. There must be a formula, or a sqrt decomposition, or binary search on the answer.

> *Contest story: a problem asked to count pairs summing to a target with $n \le 2 \cdot 10^5$. Half the field wrote the $\mathcal{O}(n^2)$ double loop and TLE'd. The constraints were the hint: $2 \cdot 10^5$ is exactly the size where $\mathcal{O}(n \log n)$ fits and $\mathcal{O}(n^2)$ does not. Sorting plus two pointers, 20 lines.*

---

## 12. Beyond Big-O: the constant factor

Two $\mathcal{O}(n)$ algorithms can differ by 10x. At $n = 10^7$ that is 30 ms versus 300 ms — the difference between AC and TLE.

### Where the constant goes

| Effect | Cost | Fix |
|---|---|---|
| Cache miss | ~100 cycles vs ~4 for L1 | Iterate contiguously |
| Pointer chase (`list`, `map`) | ~100 cycles per hop | Use contiguous containers |
| `unordered_map` lookup | ~20–50 ns | Array if the key range is small |
| `std::function` call | 10–100x a direct call | Template or a lambda |
| `endl` | flushes the buffer every call | `'\n'` |
| `cin` without sync-off | ~3x slower than `scanf` | `sync_with_stdio(false)` |
| `vector<vector<int>>` access | 2 loads instead of 1 | Flatten to `i*m + j` |
| Recursion | call overhead + stack | Iterate, or inline |
| `%` operator | ~20–40 cycles on x86 | Bitmask when the modulus is a power of 2 |

### The cache hierarchy, in practical terms

| Level | Latency | Size |
|---|---|---|
| L1 | ~4 cycles | 32–64 KB |
| L2 | ~12 cycles | 256 KB – 1 MB |
| L3 | ~40 cycles | 4–32 MB |
| RAM | ~100–300 cycles | GBs |

An array of $10^6$ `int` is 4 MB — it no longer fits in L2, so every access past the first is an L3 or RAM latency. This is why **the order you iterate a 2D array matters** (Chapter 1, Topic 09) and why a `vector` beats a `list` by 3–10x on traversal.

### The doubling experiment

The most reliable way to identify a complexity class empirically: run at $n$, $2n$, $4n$ and compare times.

| Ratio $t(2n)/t(n)$ | Complexity |
|---|---|
| ~1 | $\mathcal{O}(1)$ or $\mathcal{O}(\log n)$ |
| ~2 | $\mathcal{O}(n)$ |
| ~2.1–2.3 | $\mathcal{O}(n \log n)$ |
| ~4 | $\mathcal{O}(n^2)$ |
| ~8 | $\mathcal{O}(n^3)$ |
| ~4 (from doubling $n$) when $n$ is exponent | $\mathcal{O}(2^n)$ — absurdly fast growth |

```cpp
auto start = chrono::steady_clock::now();
solve();
auto ms = chrono::duration_cast<chrono::milliseconds>(
              chrono::steady_clock::now() - start).count();
cerr << ms << " ms\n";            // cerr: not buffered, never affects the output
```

Measure with `cerr`, never `cout` — `cerr` is unbuffered so it appears even if the program crashes, and it will not pollute the answer file.

---

## 13. Advanced: the log factors that hide in plain sight

### $\mathcal{O}(n \log n)$ versus $\mathcal{O}(n \log^2 n)$

At $n = 10^6$: $n \log n \approx 2 \times 10^7$ (fine), $n \log^2 n \approx 4 \times 10^8$ (borderline). One extra log factor costs a factor of 20. This is why "binary search for each element" is a different solution from "sort once and two-pointer".

### Sum over divisors

$$\sum_{i=1}^{n} \left\lfloor \frac{n}{i} \right\rfloor \approx n \ln n = \mathcal{O}(n \log n)$$

This is the cost of "for each $i$, do something for all its multiples" — divisor counting, the sieve, and many number-theory loops. It looks like $n^2$ and is not.

### Sqrt decomposition

Splitting an array into blocks of size $\sqrt n$ gives $\mathcal{O}(\sqrt n)$ per query and $\mathcal{O}(n \sqrt n)$ overall. At $n = 10^5$: $3 \times 10^7$ operations — comfortable. At $n = 10^6$: $10^9$ — too slow. Sqrt decomposition lives in a narrow band.

### Bit complexity

When the input is a number rather than an array, the size is the number of *bits*, not the value. Multiplication of two $k$-bit numbers is $\mathcal{O}(k^2)$ with the schoolbook algorithm and $\mathcal{O}(k^{1.585})$ with Karatsuba. Factoring an $n$-bit number is believed to be hard — which is the basis of RSA, and why "$\mathcal{O}(\sqrt N)$ trial division" is hopeless for $N$ near $10^{18}$ but fine for $10^{12}$.

### Polynomial versus pseudo-polynomial

The subset-sum DP is $\mathcal{O}(n \cdot S)$ where $S$ is the target sum. That is polynomial in $S$ but **exponential in the number of bits** of $S$. Problems with $S \le 10^5$ are solvable this way; problems with $S \le 10^{18}$ are NP-hard in practice. The constraint tells you which one you are facing.

---

## 14. A complexity analysis checklist

When you finish writing a solution, before submitting:

1. **Identify the dominant loop or recursion.** Ignore everything else.
2. **Count iterations of the inner body**, not the outer.
3. **Check for hidden calls:** `sort`, `find`, `substr`, `map` lookups, string concatenation.
4. **Compute the total** and compare it against $10^8$ for the maximum $n$.
5. **Multiply by the constant factor** — is the inner body a pointer chase or an array read?
6. **Check the space** against 256 MB.
7. **Check the recursion depth** against ~1–8 MB of stack.
8. **Check the per-test-case reset.** With $T = 10^4$ test cases and an $\mathcal{O}(n)$ clear each, an $\mathcal{O}(n)$ solution becomes $\mathcal{O}(Tn)$.

> Step 8 catches a surprising number of TLEs. A solution that is $\mathcal{O}(n)$ per test with $T = 10^5$ and $n = 10^5$ is $\mathcal{O}(Tn) = 10^{10}$ overall — even though "each test is linear".

---

## Key Takeaways

- **1 second $\approx 10^8$ simple operations** in C++ with `-O2`. Budget conservatively: $5 \times 10^8$ is borderline, $10^9$ is usually TLE.
- Big-O is an upper bound that ignores constants. It predicts growth, not wall-clock time. Both matter.
- Nested loops **multiply**; sequential blocks **add**; halving gives $\log$.
- Dependent bounds need summing: `j = i..n` is still $\mathcal{O}(n^2)$, but `j += i` is $\mathcal{O}(n \log n)$ via the harmonic sum.
- Two pointers is $\mathcal{O}(n)$ — cost is the number of iterations, not the number of variables.
- `vector::push_back` is amortised $\mathcal{O}(1)$ because the doubling copies total $2n$. DSU is $\mathcal{O}(\alpha(n))$, effectively constant.
- `sort` performs $n \log n$ **comparisons**; if each comparison is $\mathcal{O}(L)$ (strings), the real cost is $nL\log n$.
- Judges test the **worst case**. An average-case-fast algorithm with an adversarial worst case will be broken by the setter.
- Recurrences: use the Master Theorem for $T(n) = aT(n/b) + f(n)$; unroll for everything else.
- **The constraints encode the intended complexity.** $n \le 10^5$ means $\mathcal{O}(n\log n)$ exists; $n \le 20$ means exponential.
- The constant factor can be 10x: cache misses, `endl`, `map` lookups, `std::function`, and `vector<vector<>>` all cost real time.
- Do not forget the **per-test-case** cost. $T$ tests × $\mathcal{O}(n)$ each can be $10^{10}$.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Analysing the outer loop only | Inner dependent loops change the class | Sum the inner iterations |
| 2 | Assuming `j = i..n` is $\mathcal{O}(n)$ | It is $n^2/2$ | Sum: $n(n+1)/2$ |
| 3 | Missing that `j += i` gives a harmonic sum | You conclude $n^2$ and reject a valid solution | It is $\mathcal{O}(n\log n)$ |
| 4 | Ignoring the cost of the function called inside a loop | $n \times \mathcal{O}(f)$ | Count what `f` does |
| 5 | Treating `sort` as $\mathcal{O}(1)$ per element | `sort` is $n\log n$ | $n\log n$, times comparison cost |
| 6 | Forgetting string comparison is $\mathcal{O}(L)$ | Sorting strings is $nL\log n$ | Account for the length |
| 7 | Using `s = s + c` in a loop | $\mathcal{O}(n^2)$ copies | `s += c` (amortised $\mathcal{O}(1)$) |
| 8 | Counting a `map` lookup as $\mathcal{O}(1)$ | It is $\mathcal{O}(\log n)$ with a large constant | Use an array when the range is small |
| 9 | Assuming `unordered_map` is always $\mathcal{O}(1)$ | Adversarial input forces $\mathcal{O}(n)$ | Custom hash with a random seed |
| 10 | Forgetting the per-test-case reset | $T \times \mathcal{O}(n)$ dominates | Reset only what you touched |
| 11 | "It runs in 0.2 s locally" | Local $n$ is small; judge $n$ is maximal | Compute the worst case on paper |
| 12 | Optimising before measuring | You fix the wrong thing | Find the dominant term first |
| 13 | Ignoring memory | MLE on the big tests | Budget bytes before declaring arrays |
| 14 | Deep recursion without checking the stack | RE on large inputs | Convert to iterative, or raise the limit |
| 15 | Assuming $n$ in the statement is the real $n$ | Sum over $T$ test cases may be larger | Check whether $n$ is per test or total |
| 16 | Using `pow(2, n)` for exponentials | Float error at $n \ge 31$ | `1LL << n` for $n \le 62$ |
| 17 | Treating $n \log n$ and $n \log^2 n$ as the same | A factor of 20 at $n = 10^6$ | Count the logs |
| 18 | Believing polynomial-time always means feasible | $\mathcal{O}(n^3)$ at $n=10^5$ is $10^{15}$ | Check the constant and the exponent against $n$ |

---

## Practice Problems

Analysis drills, not coding problems — the point is to be right on paper.

| # | Drill | Source | Difficulty |
|---|---|---|---|
| 1 | State the complexity of a 4-level nested loop with dependent bounds | Any analysis exercise | Easy |
| 2 | Given ten $n$ values and a 1 s limit, classify each as pass or TLE | Self-imposed | Easy |
| 3 | Predict the running time of `sort` on $10^6$ ints, then measure | Local | Easy |
| 4 | Run the doubling experiment and identify the complexity class from the ratios | Local | Easy–Medium |
| 5 | Derive the complexity of the sieve of Eratosthenes from the harmonic sum | Textbook reasoning | Medium |
| 6 | Solve a Master Theorem recurrence, all three cases | CLRS-style exercises | Medium |
| 7 | Analyse a recursive function and convert it to memoised form, comparing both | CSES "DP" intro problems | Medium |
| 8 | Find the hidden $\mathcal{O}(n)$ call inside an apparently linear loop | Self-imposed code review | Medium |
| 9 | Given $n \le 2 \cdot 10^5$, argue why $\mathcal{O}(n\sqrt n)$ is too slow but $\mathcal{O}(n\log n)$ is fine | Self-imposed | Medium |
| 10 | Compute the memory of a 3D DP table and decide whether it fits 256 MB | Self-imposed | Easy–Medium |
| 11 | Explain why subset-sum DP with $S \le 10^{18}$ is not polynomial | Theory | Hard |
| 12 | Optimise an $\mathcal{O}(n^2)$ solution to $\mathcal{O}(n\log n)$ and measure both | Any judge, same problem | Medium–Hard |

---

## Next Up

→ [Topic 03 — Pairs, Tuples and Simulation](Topic%2003%20-%20Pairs%2C%20Tuples%20and%20Simulation.md)

Complexity tells you what you can afford. Topic 03 gives you the two workhorses for the cheapest class of problems in every contest: `pair` and `tuple` for grouping values (and for sorting by multiple keys), and **simulation** — the discipline of implementing exactly what the statement says. It ends with five fully worked simulation programs, which is where the "read carefully, code carefully" skill becomes concrete.