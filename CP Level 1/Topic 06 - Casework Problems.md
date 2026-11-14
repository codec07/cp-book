# Topic 06 — Solving Casework Problems

Casework is the technique of splitting a problem into a small number of **disjoint, exhaustive** situations and solving each one separately. It is unglamorous, it is the most common technique on Div. 2 C, and it is where most contestants lose points — not because the cases are hard, but because one boundary between two cases was drawn in the wrong place.

The five worked problems below are all casework problems in the olympiad sense: the difficulty is the analysis, and the code is short. That ratio is the point.

> **Prerequisites:** [Topic 05 — Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md), [Topic 04 — Complete Searching](Topic%2004%20-%20Complete%20Searching.md)
> **You will be able to:**
> - State the two obligations every case split must satisfy, and check them.
> - Choose the right axis to split on: sign, parity, size, structure, or position.
> - Recognise the six boundary cases that appear in almost every problem.
> - Compress a case split back into a single formula when one exists.
> - Test casework by exhaustive enumeration on small inputs.

---

## 1. What casework is, and why it is not a cop-out

A case split says: *the input has one of these $k$ shapes; for each shape, here is a direct answer.*

```
if (shape A) return solveA();
else if (shape B) return solveB();
else return solveC();
```

This is not a failure to find the general solution. For a large class of problems **there is no single formula** — the answer genuinely depends on structure, and the intended solution is the enumeration of structures. A setter who writes "if $n$ is odd, the answer is $-1$" is not being lazy; the parity of $n$ is a real feature of the problem.

Casework is legitimate when:

- The cases correspond to genuinely different mathematical situations.
- Each case is simple enough to verify by hand on a small example.
- The split is small (usually 2–5 cases).

Casework is a smell when:

- You have more than about six cases, or nested cases.
- Two cases share 90% of their code and differ in one sign.
- You cannot say why the cases are exhaustive without waving.

---

## 2. The two obligations

Every case split must satisfy both. Skipping either check is how wrong answers happen.

### Obligation 1 — Exhaustive

Every possible input falls into at least one case. There is no input that reaches no branch.

### Obligation 2 — Disjoint

No input falls into two cases, **or** if it does, the cases agree on that input. Overlap is tolerable only when you can prove the answers coincide; otherwise the order of your `if` statements silently decides the answer, which is a bug waiting to be exposed by a test.

### The check

Write the split as a table with the *condition* in one column and *why it is covered* in another:

| Case | Condition | Answer | Covered because |
|---|---|---|---|
| A | $n$ odd | $-1$ | |
| B | $n$ even, $n = 2$ | $-1$ | boundary |
| C | $n$ even, $n \ge 4$ | $n/2 - 1$ | $n \ge 4$ is the entire remainder |

If the "covered because" column for the union of the rows is not "all inputs", the split is incomplete. If two rows can both be true, you need a tie-break rule.

---

## 3. Where to cut: the catalogue of split axes

Experienced solvers do not invent case splits from nothing. They run through a fixed list.

| Axis | Split on | Typical problem |
|---|---|---|
| **Sign** | negative / zero / positive | Products, comparisons, absolute values |
| **Parity** | odd / even | Tilings, sums, colourings, operations |
| **Size** | $n = 1$, $n = 2$, $n \ge 3$ | Constructions, boundary-heavy statements |
| **Structure** | all equal / all distinct / has duplicates | Array rearrangement, mode questions |
| **Count** | 0 / 1 / ≥2 occurrences | Counting, matching, greedy feasibility |
| **Position** | first / last / interior | Rotations, cyclic arrays, endpoints |
| **Shape** | sorted / unsorted, convex / concave | Geometry, monotonicity |
| **Feasibility** | possible / impossible, and why | Construction problems |

Run the list top to bottom against the statement. The right axis is usually the one the statement's constraints mention indirectly — "rearrange the digits" hints at zero-handling, "colour the grid" hints at parity.

---

## 4. The six boundary cases that decide contests

Almost every casework bug is one of these. Check them explicitly before submitting.

1. **$n = 1$.** Loops that read `a[i-1]` or `a[i+1]` do not run. Constructions with "the rest" have no rest.
2. **$n = 2$.** Some constructions need three elements to make a pattern.
3. **Zero.** Zero is even, it is its own negative, and it can be a valid answer. `abs(0)`, `0 % 2`, `0` as a divisor.
4. **All elements equal.** Any "find a different element" logic fails.
5. **The maximum value.** Overflow, array bounds at `a[n]`, and time limits all bite here.
6. **The "impossible" answer.** If the statement says "print $-1$ if impossible", at least one test is impossible. Find it.

A useful discipline: before coding, write down the answer for $n = 1$, $n = 2$, $n = 3$, all-equal, and all-zero by hand. If your program disagrees on any of them, the split is wrong — and you found out in two minutes instead of after three wrong submissions.

---

## 5. Compressing casework back into a formula

A good case split is often the *derivation* of a one-line answer. After you have written the cases, look for the pattern:

| Case split | Compressed form |
|---|---|
| if $n$ even → $n/2$; if odd → $(n-1)/2$ | `n / 2` (integer division) |
| if $x > 0$ → 1; if $x < 0$ → $-1$; else 0 | `(x > 0) - (x < 0)` |
| if $a < b$ → $a$; else $b$ | `min(a, b)` |
| three cases on parity of two numbers | `(a + b) & 1` |

Compression is not required — correct casework is fine — but the compressed form is shorter, has no branch mispredictions, and cannot have an ordering bug. **Derive with cases, ship the formula, keep the cases as a comment.**

```cpp
// Cases: both even -> 0; both odd -> 2; mixed -> 1
// Compressed: (a % 2) + (b % 2)          (for non-negative a, b)
```

---

## 6. The code shape

Two structures, and only two.

```cpp
// A. Early returns — best when each case is a complete answer
long long solve(long long n) {
    if (n == 1) return 0;
    if (n == 2) return 1;
    if (n % 2 == 0) return n / 2;
    return n - 1;
}

// B. Guards then general case — best when one case is "everything else"
void solve() {
    if (n < 2) { cout << -1 << '\n'; return; }
    if (n * m % 2 == 0) { cout << 0 << '\n'; return; }
    // the general case, never indented by a special case
}
```

Prefer early returns. They make the case list readable top to bottom and remove nesting, which is where overlap bugs hide.

**Order the cases from most specific to most general.** `if (n == 1)` must come before `if (n % 2 == 1)`, or the first case is unreachable for $n = 1$.

---

## 7. Solved Problem 1 — Maximum Product of Three

> **Statement.** Given $n \ge 3$ integers $a_1, \dots, a_n$ with $|a_i| \le 10^6$, choose three distinct indices so that the product of the chosen values is as large as possible. Output that maximum product.

### The trap

The intuitive answer is "the three largest values". That is wrong whenever two large-magnitude negatives are available: $-10 \times -9 \times 8 = 720$ beats $8 \times 7 \times 6 = 336$.

### Case analysis on the sign pattern of the optimal triple

Sort the array ascending. Any triple falls into one of three sign classes:

| Class | Product sign | Best selection |
|---|---|---|
| Three non-negative | $\ge 0$ | The three largest values |
| One negative, two non-negative | $\le 0$ | Never optimal unless all triples are negative |
| Two negatives, one non-negative | $\ge 0$ | The two **most negative** and the **largest** |
| Three negatives | $\le 0$ | The three largest (least negative) |

So the maximum is the larger of exactly two candidates:

1. $a_{n-1} \cdot a_{n-2} \cdot a_{n-3}$ — the three largest.
2. $a_0 \cdot a_1 \cdot a_{n-1}$ — the two smallest (most negative) times the largest.

**Why these two suffice.** A product is maximised either by making all factors large and positive, or by making the sign positive through an even number of negatives while using the largest magnitudes. Candidate 1 covers zero-or-more-negatives with maximum positives; candidate 2 covers exactly two negatives with maximum magnitude. Any other triple is dominated: replacing a non-extreme element by a more extreme one of the same sign does not decrease the product.

### Exhaustiveness check

- $n = 3$: candidates 1 and 2 are the same triple (indices $\{0,1,2\}$ and $\{0,1,2\}$). Both evaluated, answer correct.
- All negative: candidate 1 uses the three least-negative; candidate 2 uses two most-negative and the least negative. The maximum over these covers both sign classes.
- Contains zero: a product of $0$ is compared normally; if every product is negative, $0$ wins as it should.

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    cin >> n;
    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    sort(a.begin(), a.end());

    long long top3  = a[n-1] * a[n-2] * a[n-3];
    long long bot2  = a[0] * a[1] * a[n-1];

    cout << max(top3, bot2) << '\n';
    return 0;
}
```

**Input**
```text
6
-10 -9 1 2 3 8
```

**Output**
```text
720
```

### Complexity

$\mathcal{O}(n \log n)$ for the sort. If the values fit in `int`, we could find the three largest and two smallest in one pass for $\mathcal{O}(n)$, but the sort is simpler and fast enough.

### Pitfalls

- **Overflow.** With $|a_i| \le 10^9$ the product reaches $10^{27}$ and overflows `long long`. Either constrain the input or use `__int128`. This is why the statement above says $10^6$.
- Using `int` for the product. Three values of $10^6$ already reach $10^{18}$.
- Taking the three smallest instead of the two smallest times the largest.

---

## 8. Solved Problem 2 — Tiling a Board With One Cell Removed

> **Statement.** An $n \times m$ board has cell $(r, c)$ removed (1-indexed). Can the remaining cells be tiled exactly by $1 \times 2$ dominoes? Output `YES` or `NO`.

### The invariant

Colour the board like a chessboard, with cell $(i,j)$ black when $i + j$ is even. Every domino covers exactly one black and one white cell. Therefore **a tiling exists only if the numbers of black and white cells are equal.**

### Case analysis

The full board has $nm$ cells. After removing one, $nm - 1$ cells remain, which must be even — so $nm$ must be **odd**.

| Case | Condition | Colour counts after removal | Tiling? |
|---|---|---|---|
| $nm$ even | any | the two colours differ by 1 | **NO** |
| $nm$ odd, majority colour removed | $r+c$ even | equal | **YES** |
| $nm$ odd, minority colour removed | $r+c$ odd | differ by 2 | **NO** |

When $nm$ is odd, both $n$ and $m$ are odd, and the corner $(1,1)$ has $r + c = 2$, even. The even-parity class has $(nm+1)/2$ cells — one more than the odd class. So the **majority colour is even parity**, and the removed cell must satisfy $(r+c) \bmod 2 = 0$.

### Why sufficiency holds

The necessary condition is also sufficient. For odd $n$ and $m$ with an even-parity cell removed, a tiling can be constructed: tile each row from the left in pairs up to the removed column, then handle the remaining rectangle. The colouring argument supplies the impossibility half, which is the part that is hard to see — the construction half is standard and is why this is a classically complete case split rather than a one-sided bound.

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    long long n, m, r, c;
    cin >> n >> m >> r >> c;

    if ((n * m) % 2 == 0) {
        cout << "NO\n";
    } else if ((r + c) % 2 == 0) {
        cout << "YES\n";
    } else {
        cout << "NO\n";
    }
    return 0;
}
```

**Input**
```text
3 3 2 2
```

**Output**
```text
YES
```

### Complexity

$\mathcal{O}(1)$.

### Pitfalls

- Using `int` for `n * m` when $n, m$ can be $10^9$. The product overflows.
- Getting the majority colour backwards. The safe habit: evaluate the condition on the corner $(1,1)$ by hand before trusting a `% 2` test. Note that switching to 0-indexed coordinates shifts both $r$ and $c$ by 1, so $r+c$ changes by 2 and its parity is unchanged — the condition is the same under either convention.
- Forgetting that when $nm$ is even the answer is always `NO`, regardless of the removed cell.

---

## 9. Solved Problem 3 — A String With Exactly $k$ Distinct Characters

> **Statement.** Given $n$ and $k$, construct any string of length $n$ over lowercase English letters containing **exactly** $k$ distinct characters. If impossible, output `-1`.

### Case analysis

| Case | Condition | Answer |
|---|---|---|
| Too many distinct | $k > 26$ | impossible |
| Not enough room | $k > n$ | impossible |
| Empty string wanted | $n = 0$ | empty string (valid iff $k = 0$) |
| Wants no distinct chars | $k = 0$, $n > 0$ | impossible |
| Otherwise | $1 \le k \le \min(n, 26)$ | construct |

### Construction

Place the $k$ distinct characters in the first $k$ positions, then fill the remaining $n - k$ positions with the first character (already present, so no new distinct character is introduced).

```cpp
string s;
for (int i = 0; i < k; i++) s += char('a' + i);   // exactly k distinct
while ((int)s.size() < n) s += 'a';               // repeats, adds nothing new
```

### Exhaustiveness check

- $k > 26$ and $k > n$ can both hold; either order of the two checks yields `-1`, so disjointness is not required here.
- $n = 0$ with $k = 0$: the empty string has zero distinct characters. Correct, and it is the case almost everyone misses.
- $n = 0$ with $k > 0$: caught by $k > n$.
- $k = 1$: the loop adds `'a'`, then the while fills the rest with `'a'`. Exactly one distinct.

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, k;
    cin >> n >> k;

    if (k > 26 || k > n) {
        cout << -1 << '\n';
        return 0;
    }
    if (n == 0) {                 // k must be 0 here
        cout << '\n';
        return 0;
    }
    if (k == 0) {                 // n > 0 but no distinct chars allowed
        cout << -1 << '\n';
        return 0;
    }

    string s;
    for (int i = 0; i < k; i++) s += char('a' + i);
    while ((int)s.size() < n) s += 'a';

    cout << s << '\n';
    return 0;
}
```

**Input**
```text
7 3
```

**Output**
```text
abcaaaa
```
Three distinct characters (`a`, `b`, `c`), length 7.

### Complexity

$\mathcal{O}(n)$.

### Pitfalls

- `k > 26` is not implied by `k > n` — a string of length 30 asking for 30 distinct letters is impossible, and only the first check catches it.
- Building the string with `s = s + c` instead of `s += c` makes it $\mathcal{O}(n^2)$.
- Forgetting $k = 0$ with $n > 0$: the while loop would leave the string empty, which has length 0, not $n$.

---

## 10. Solved Problem 4 — Counting Triples With Even Sum

> **Statement.** Given $n$ integers with $|a_i| \le 10^9$, count the triples $i < j < k$ whose sum is even.

### The parity argument

A sum is even exactly when the number of odd addends is even. With three addends, that means **zero odds or two odds**. This is the entire analysis:

$$\text{answer} = \binom{E}{3} + \binom{O}{2} \cdot E$$

where $E$ is the count of even values and $O$ the count of odd values.

### Case analysis

| Case | Odds in the triple | Count |
|---|---|---|
| All even | 0 | $\binom{E}{3}$ |
| Two odd, one even | 2 | $\binom{O}{2} \cdot E$ |
| One odd, two even | 1 → odd sum | excluded |
| All odd | 3 → odd sum | excluded |

The two included cases are disjoint (they differ in the number of odds) and together cover every triple with an even sum.

### Deriving the closed form

Rather than looping over triples:

$$\binom{x}{3} = \frac{x(x-1)(x-2)}{6}, \qquad \binom{x}{2} = \frac{x(x-1)}{2}$$

Guard each with `x >= 3` and `x >= 2` so the product never goes negative.

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

long long choose2(long long x) {
    if (x < 2) return 0;
    return x * (x - 1) / 2;
}

long long choose3(long long x) {
    if (x < 3) return 0;
    return x * (x - 1) * (x - 2) / 6;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    cin >> n;

    long long even = 0, odd = 0;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        if (x % 2 == 0) even++;
        else            odd++;
    }

    long long ans = choose3(even) + choose2(odd) * even;
    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5
1 2 3 4 5
```

**Output**
```text
6
```

Even values $\{2,4\}$, odds $\{1,3,5\}$. $\binom{2}{3} = 0$, so the answer is $\binom{3}{2} \cdot 2 = 3 \cdot 2 = 6$. The six triples are $(1,3)$ or $(1,5)$ or $(3,5)$ paired with either 2 or 4, with sums $6, 8, 8, 10, 10, 12$ — all even.

### Complexity

$\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space.

### Pitfalls

- $\binom{n}{3}$ for $n = 2 \times 10^5$ is about $1.3 \times 10^{15}$ — it needs `long long`, and the intermediate `x*(x-1)*(x-2)` reaches $8 \times 10^{15}$, which still fits.
- Negative values: `x % 2 == 0` is correct for negatives in C++ (the result is 0 for even negatives). But `x % 2 == 1` is **not** a valid odd test for negatives — it returns $-1$.
- Forgetting the $\binom{x}{3}\text{-with-}x<3$ guard produces a negative product.

---

## 11. Solved Problem 5 — Reduce to Zero

> **Statement.** Starting from a positive integer $n$, each operation costs 1 and is either (a) subtract 1, or (b) if the current value is even, divide it by 2. Find the minimum number of operations to reach 0.

### Case analysis on parity

| Current value | Available moves | Optimal |
|---|---|---|
| Odd, $n > 1$ | Only subtract | Subtract (division is impossible) |
| Even | Subtract, or divide | **Divide** |
| $n = 1$ | Only subtract | Subtract |

### Why divide always wins for even $n$

For even $n \ge 4$, dividing costs 1 and reaches $n/2$, while any path that subtracts first must spend at least 2 operations to get no lower than $n - 2$. Since $n/2 \le n - 2$ for $n \ge 4$, dividing reaches a value at least as small for at most the same cost — and by induction on $n$ that cannot be worse.

The only exception is $n = 2$: dividing reaches 1 in one operation, then 1 needs one more, for a total of 2 — the same as subtracting twice. Both are optimal, so the greedy choice is still safe.

For odd $n > 1$, division is unavailable, so subtraction is forced. In fact every odd step is followed by an even step, so the process costs at most $2\log_2 n$ operations.

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    long long n;
    cin >> n;

    long long ops = 0;
    while (n > 0) {
        if (n % 2 == 0) n /= 2;
        else            n -= 1;
        ops++;
    }

    cout << ops << '\n';
    return 0;
}
```

**Input**
```text
10
```

**Output**
```text
5
```

Trace: $10 \to 5 \to 4 \to 2 \to 1 \to 0$, five operations.

### Complexity

$\mathcal{O}(\log n)$ — each pair of steps at least halves the value.

### Pitfalls

- Testing `n % 2 == 1` for oddness. For positive inputs it works, but a defensive habit is `n & 1`.
- Using this loop when the operation set is "subtract 1 or add 1, then divide" — that is a different (and harder) problem where the optimal move for odd $n$ depends on $n \bmod 4$. Read the operation list carefully.
- Off-by-one in `ops`: the loop counts every operation including the final subtraction from 1 to 0.

---

## 12. Testing casework

Casework bugs are boundary bugs, and boundaries are exactly what random testing finds fast. **Do not test casework by staring at it.**

### The exhaustive small-input sweep

For a problem on one integer, test every value in a range:

```cpp
for (int n = 1; n <= 200; n++) {
    long long fast = solveFast(n);
    long long slow = solveBrute(n);      // e.g. BFS over states for small n
    if (fast != slow) {
        cout << "MISMATCH n=" << n
             << " fast=" << fast << " slow=" << slow << '\n';
        return 1;
    }
}
```

For a problem on an array, generate all arrays of length up to 4 over a small alphabet:

```cpp
// all arrays of length 3 with values in {-2..2}: 125 cases
for (int a = -2; a <= 2; a++)
for (int b = -2; b <= 2; b++)
for (int c = -2; c <= 2; c++)
    check({a, b, c});
```

This finds sign bugs (negative values), parity bugs (both parities present), zero bugs, and duplicate bugs — the four categories that cover most casework failures.

### The targeted boundary test

Add the six cases from §4 explicitly, as literal inputs:

```cpp
vector<long long> boundary = {1, 2, 3, 0, -1, INT_MAX, LLONG_MAX};
```

| Test | Catches |
|---|---|
| $n = 1$ | Loops that read neighbours |
| $n = 2$ | Constructions needing three elements |
| All zeros | Division by zero, `abs(0)`, sign ambiguity |
| All equal | "Find a different element" logic |
| Maximum value | Overflow, off-by-one at `a[n]` |
| The impossible case | Unreachable `-1` branch |

---

## 13. Ten unsolved problems

Olympiad style, with constraints. No solutions — the point is the case analysis.

| # | Problem | Constraints |
|---|---|---|
| 1 | **Two Even Parts.** Decide whether $n$ can be written as the sum of two positive even integers. Print `YES` or `NO`. | $1 \le n \le 100$ |
| 2 | **Odd Sum Pairs.** Count pairs $i < j$ with $a_i + a_j$ odd. | $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$ |
| 3 | **Bishop's Reach.** On an $8 \times 8$ board, decide whether a bishop can travel from $(r_1,c_1)$ to $(r_2,c_2)$ in any number of moves. Note a bishop stays on one colour. | $1 \le r, c \le 8$ |
| 4 | **One Change to Increase.** Decide whether the array can be made strictly increasing by changing the value of at most one element (any new value is allowed). | $n \le 2 \times 10^5$ |
| 5 | **Remove to Make the Sum Even.** Find the minimum number of elements to remove so the remaining sum is even, or report that it is impossible. | $n \le 2 \times 10^5$, $\lvert a_i \rvert \le 10^9$ |
| 6 | **Three Palindromes.** Decide whether $s$ can be split into three non-empty contiguous substrings, each a palindrome. | $\lvert s \rvert \le 2000$ |
| 7 | **Triangle Triples.** Count triples of indices that form a non-degenerate triangle (strict triangle inequality). | $n \le 2000$, $a_i \le 10^9$ |
| 8 | **Consecutive Sum.** Find the smallest $k \ge 1$ such that $n$ is a sum of $k$ consecutive positive integers. | $n \le 10^9$ |
| 9 | **Two-Colour Grid.** Repaint the minimum number of cells so no two side-adjacent cells share a colour, using exactly two colours. | $n, m \le 100$ |
| 10 | **Median After Removal.** Given an array of odd length, remove exactly one element so that the median of the remaining array equals the original median. Count the valid removals. | $n$ odd, $n \le 2 \times 10^5$ |

### Hints, in the spirit of the topic

- **#1** is the classic $n$ even and $n > 2$ split; the $n = 2$ case is the trap.
- **#2** is parity counting: a sum is odd exactly when the parities differ.
- **#3** needs the colour invariant plus the degenerate case where the two squares coincide.
- **#4** has a two-pointer definition of "already a violation" — count the violations, then check whether one edit can fix them all.
- **#5** depends only on whether the total is already even and whether any odd element exists.
- **#6** is a split-position enumeration with a palindrome table; the casework is in which of the three parts can be length 1.
- **#7** after sorting, is a two-pointer count per largest side.
- **#8** splits on the parity of the number of terms and the 2-adic valuation of $n$.
- **#9** is the checkerboard colouring argument again — two colourings, take the cheaper.
- **#10** splits on where the removed element sits relative to the median's position.

---

## Key Takeaways

- A case split must be **exhaustive** and **disjoint**. Write the table with a "covered because" column; if you cannot fill it, the split is incomplete.
- Split on a known axis: sign, parity, size, structure, count, position, shape, feasibility. Run the list top to bottom.
- The six boundary cases — $n=1$, $n=2$, zero, all-equal, maximum value, the impossible branch — are where casework bugs live. Test them literally.
- Order cases from most specific to most general, or a specific case becomes unreachable.
- Derive with cases, ship the formula: `(x > 0) - (x < 0)`, `n / 2`, `(a % 2) + (b % 2)`. Compression removes ordering bugs.
- Sort before casework whenever signs or magnitudes matter. "The three largest" and "the two most negative and the largest" is a two-case split that covers every sign pattern.
- Parity arguments plus a colouring invariant settle most tiling and grid problems. The invariant gives impossibility; the construction gives sufficiency.
- Constructive problems always have a feasibility split: too many distinct, not enough room, empty input. Enumerate them before writing the builder.
- Combinatorial counting is casework on which classes contribute: "even sum" is "zero odds or two odds".
- Test casework with exhaustive small-input sweeps over a small alphabet including negatives and zero, not by inspection.
- `x % 2 == 1` is not a valid oddness test for negative `x`; it returns $-1$.
- Verify a closed form against a hand count before trusting it: count the $\binom{E}{3} + \binom{O}{2}E$ answer for a five-element example and compare with the six triples listed by hand.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Non-exhaustive split | Some input reaches no branch | Fill the "covered because" column |
| 2 | Overlapping cases with different answers | The `if` order silently decides | Make them disjoint, or prove agreement |
| 3 | General case written before the specific one | Specific case unreachable | Order most-specific first |
| 4 | Forgetting $n = 1$ | Neighbour reads, empty "rest" | Test $n=1$ explicitly |
| 5 | Forgetting $n = 2$ | Constructions needing three elements fail | Test $n=2$ explicitly |
| 6 | Treating zero as positive or negative | Sign splits misclassify it | Handle zero as its own case |
| 7 | Assuming "the three largest" for a product | Two negatives beat three positives | Compare both candidates |
| 8 | `x % 2 == 1` for oddness | Negative odd values give $-1$ | `x % 2 != 0` or `x & 1` |
| 9 | `n * m` in `int` | Overflow before the parity test | `long long` |
| 10 | Colour parity computed in 0-indexed coordinates | The majority colour flips | Fix the convention and check $(1,1)$ |
| 11 | Constructive feasibility missing a bound | `k > 26` slipped through because `k <= n` held | List every independent bound |
| 12 | Empty input treated as impossible when it is valid | `n = 0, k = 0` returns `-1` | Handle the empty case explicitly |
| 13 | `s = s + c` in a builder | $\mathcal{O}(n^2)$ | `s += c` |
| 14 | $\binom{x}{k}$ without an $x \ge k$ guard | Negative product from a "zero" combination | Guard each binomial |
| 15 | Intermediate product overflow in a binomial | `x*(x-1)*(x-2)` near $2\times10^5$ is $8\times10^{15}$ — fits, but $n^3$ for $10^6$ does not | `long long`, or divide early |
| 16 | Trusting a closed form without a hand count | A sign or off-by-one in the formula | Count a small case manually |
| 17 | Testing casework by inspection | Boundaries are invisible to inspection | Exhaustive sweep over small inputs |
| 18 | Six or more cases, nested | Overlap bugs multiply | Look for a compressed formula or an invariant |

---

## Practice Problems

The ten unsolved problems above are the practice set for this topic. Worked solutions are deliberately omitted — derive the cases, then check yourself by exhaustive small-input testing.

| # | Where to start | Difficulty |
|---|---|---|
| 1–3 | Pure parity and invariant arguments | Easy |
| 4–5 | Counting violations, then a single fix | Easy–Medium |
| 6–7 | Split enumeration with a helper structure | Medium |
| 8–9 | Number theory on the split, colouring again | Medium |
| 10 | Position-relative casework on a sorted array | Medium–Hard |

---

## Next Up

→ [Topic 07 — Introduction to Greedy Algorithms and Graphs](Topic%2007%20-%20Introduction%20to%20Greedy%20Algorithms%20and%20Graphs.md)

Casework handles problems whose answer *depends on structure*. **Greedy** handles problems where a locally best choice is provably globally best — usually after the right sort, which is why it follows sorting in the curriculum. Topic 07 introduces the exchange argument, then graphs: what a graph is, how it is stored in an adjacency list, and how a traversal explores it.