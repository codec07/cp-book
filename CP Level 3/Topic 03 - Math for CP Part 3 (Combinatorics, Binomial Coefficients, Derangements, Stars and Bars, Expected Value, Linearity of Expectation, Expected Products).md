# Topic 03 — Math for CP Part 3: Combinatorics, Binomials, Derangements, Stars and Bars, Expected Value

There is a particular kind of contest problem that reads like a short story about seating arrangements and ends with "print the answer modulo $10^9+7$". It has no data structure, no graph, no array to scan. What it has is a **count** - usually a count so large that the only way to print it is modulo something - and your job is to find a formula, or, more often, to decompose the count into pieces you already know.

This topic is the vocabulary for that. Binomial coefficients, stars and bars, derangements, and expectation: four tools, all of them just careful counting, all of them appearing again and again in Div. 1 problems. The good news is that the mathematics is elementary. The bad news is that "elementary" in combinatorics means "easy to state, easy to get subtly wrong", and the subtleties are exactly what setters test.

> **Prerequisites:** Topic 02 (factorials, inverse factorials, modular inverses). You need to be able to compute $\binom{n}{r} \bmod p$ before you can compute anything else in this topic.
>
> **You will be able to:** translate a counting statement into a product of binomial coefficients; choose the right stars-and-bars variant in seconds; derive derangement counts by two independent routes; and compute expected values with linearity, without ever enumerating the sample space.

---

## 1. Counting is the real subject

Almost every combinatorial problem reduces to one of five questions:

1. **How many ordered selections?** $n \cdot (n-1) \cdots (n-k+1)$.
2. **How many unordered selections?** $\binom{n}{k}$.
3. **How many ways to distribute identical items?** stars and bars.
4. **How many arrangements with no object in its own place?** derangements, or inclusion-exclusion.
5. **What is the average of some quantity over all outcomes?** expectation.

The single most valuable skill is deciding *which* of the five a statement is asking. And the single most common mistake is answering a different one - counting ordered selections when the statement wanted unordered, or allowing a distribution pattern that the statement forbade.

### The words that give it away

| Phrase in the statement | What it usually means |
|---|---|
| "choose", "select", "committee", "subset" | unordered: $\binom{n}{k}$ |
| "arrange", "order", "line up", "permutation" | ordered: $n!$ or $P(n,k)$ |
| "distribute", "identical", "indistinguishable" | stars and bars |
| "no one gets their own", "nobody is in the right seat" | derangements |
| "expected", "average over all", "on average" | expectation |
| "at least one" | total minus "none" |
| "exactly $k$" | choose the $k$, then constrain the rest |
| "modulo $10^9+7$" with no input array | a formula, not a loop |

Keep the right-hand column in mind when reading. Half of combinatorics in contests is translation, and the other half is being careful about whether the objects are distinct.

---

## 2. Three principles, and the complement trick

**Product principle.** If a construction consists of independent stages with $a$ and $b$ choices, there are $ab$ constructions. "Independent" is the load-bearing word: the choices at stage two must not depend on which choice you made at stage one.

**Sum principle.** If the cases are disjoint and cover everything, add their counts. If your cases overlap, you are not using the sum principle - you are double counting, and you need inclusion-exclusion instead.

**Complement principle.** When "at least one" is awkward, count "none" and subtract from the total. This is the highest-yield trick in the topic, because "none" usually factors: at least one collision in a hash table, at least one shared birthday, at least one fixed point.

$$\#(\text{at least one}) = \#(\text{all}) - \#(\text{none}).$$

A small but decisive point: these principles count **constructions**, not numbers. If you find yourself adding counts of *values* that could coincide, you are about to double count. Deciding what the objects are - strings, subsets, functions, matchings - is the first line of every solution, and often the whole solution.

---

## 3. Permutations, combinations, and the words that signal them

$$P(n,k) = n(n-1)\cdots(n-k+1) = \frac{n!}{(n-k)!}, \qquad \binom{n}{k} = \frac{n!}{k!\,(n-k)!}.$$

The binomial coefficient counts $k$-element subsets of an $n$-element set. Equivalently, it counts the ways to choose *which* $k$ positions of $n$ are special. That second reading is the one that generalises: the moment a problem says "choose which ones", you are looking at a binomial.

Four identities used constantly:

- **Symmetry.** $\binom{n}{k} = \binom{n}{n-k}$.
- **Pascal.** $\binom{n}{k} = \binom{n-1}{k-1} + \binom{n-1}{k}$: either the last element is in the subset or not.
- **Vandermonde.** $\sum_{i} \binom{m}{i}\binom{n}{k-i} = \binom{m+n}{k}$: split a set into two parts and choose $k$ overall.
- **Hockey stick.** $\sum_{i=0}^{n} \binom{i}{k} = \binom{n+1}{k+1}$.

When $n$ is small (below about $5000$) and the modulus is arbitrary, build Pascal's triangle - addition is safe with any modulus. When $n$ is large and the modulus is prime, use factorials and inverse factorials, as in Topic 02.

```cpp
// Pascal's triangle modulo m, m arbitrary. O(n^2) time, O(n) extra space.
vector<vector<long long>> pascal(int n, long long m) {
    vector<vector<long long>> C(n + 1);
    for (int i = 0; i <= n; ++i) {
        C[i].assign(i + 1, 1);
        for (int j = 1; j < i; ++j)
            C[i][j] = (C[i - 1][j - 1] + C[i - 1][j]) % m;
    }
    return C;
}
```

---

## 4. Solved Problem 1 — The Banner Line

> **Statement.** A street is decorated with a row of $n$ coloured flags, and the municipality wants to print every distinct ordering of those flags on banners, one ordering per banner. Flags of the same colour are indistinguishable, so two orderings that differ only by swapping two identical flags are the same banner and must not be printed twice. The row can hold up to one million flags, and the number of banners is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

If all $n$ flags were distinct, there would be $n!$ orderings. Flags sharing a colour inflate that count by a factor of (number of ways to permute the identical copies), which is $c!$ for a colour appearing $c$ times. So

$$\frac{n!}{\prod_{\text{colours}} c!}$$

distinct banners. Modulo a prime larger than $n$, each $c!$ is invertible, so this is a multinomial coefficient computed with the factorial and inverse-factorial tables from Topic 02.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

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
    string s;
    cin >> s;
    int n = (int)s.size();

    vector<long long> fact(n + 1), invFact(n + 1);
    fact[0] = 1;
    for (int i = 1; i <= n; ++i) fact[i] = fact[i - 1] * i % MOD;
    invFact[n] = power(fact[n], MOD - 2, MOD);
    for (int i = n; i >= 1; --i) invFact[i - 1] = invFact[i] * i % MOD;

    long long answer = fact[n];
    vector<int> count(26, 0);
    for (char c : s) count[c - 'a']++;

    for (int c = 0; c < 26; ++c)
        if (count[c] > 0) answer = answer * invFact[count[c]] % MOD;

    cout << answer << '\n';
    return 0;
}
```

**Input**

```text
aabbcc
```

**Output**

```text
90
```

$\frac{6!}{2!\,2!\,2!} = \frac{720}{8} = 90$.

**Complexity.** $O(n + 26\log MOD)$ time, $O(n)$ memory.

### The trap

Dividing by the multinomial denominator **after** reducing is the only legal order: $n!$ modulo $p$ is not divisible by $c!$ in the integers, so `fact[n] / fact[c]` computes a meaningless quotient. Invert and multiply, always.

The second trap is the modulus: if the statement used $m = 10^9+7$ but $n \ge m$ (impossible here, but common in other problems), some factorial would be $0$ modulo $m$ and the inverse would not exist. Reading whether $n < p$ is part of reading the constraints.

</details>

---

## 5. Binomial coefficients as a modelling tool

Most appearances of $\binom{n}{k}$ are not "select $k$ from $n$" but "**choose which** $k$ of these $n$ slots carry the special property".

Three typical forms:

- **Exactly $k$ of something.** Choose the $k$ positions that have it: $\binom{n}{k}$, then fill the rest under the remaining constraint.
- **Binary strings.** A string of length $n$ with exactly $k$ ones: $\binom{n}{k}$.
- **Non-decreasing sequences.** Choosing a $k$-element subset of $\{1,\dots,n\}$ is the same as choosing a strictly increasing sequence, which is the same as choosing a non-decreasing sequence with repetitions allowed from a set of size $n-k+1$... this correspondence is stars and bars in disguise, and it is worth internalising: *weakly increasing sequences correspond to multisets, and multisets are stars and bars.*

The two-dimensional version - choose $k$ objects split between two groups - is Vandermonde, and it is the reason you can multiply generating functions later.

---

## 6. Solved Problem 2 — The Staircase Delivery

> **Statement.** A courier stands at the south-west corner of a grid of streets with $n$ avenues and $m$ streets, and must walk to the north-east corner. Every step goes either one block north or one block east - the courier never walks backwards and never cuts across a block. The city wants to know how many distinct routes exist, because it prints a map for each one. The grid can have up to a million blocks in each direction, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

Every route consists of exactly $n$ east steps and $m$ north steps, in some order. A route is therefore determined by *which* of the $n + m$ steps are the east ones, so the count is

$$\binom{n+m}{n} = \binom{n+m}{m}.$$

With $n + m$ up to $2 \times 10^{6}$, the factorial table is affordable and the answer is one multiplication per binomial - after building the inverse factorials once.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

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
    int n, m;
    cin >> n >> m;

    int N = n + m;
    vector<long long> fact(N + 1), invFact(N + 1);
    fact[0] = 1;
    for (int i = 1; i <= N; ++i) fact[i] = fact[i - 1] * i % MOD;
    invFact[N] = power(fact[N], MOD - 2, MOD);
    for (int i = N; i >= 1; --i) invFact[i - 1] = invFact[i] * i % MOD;

    long long answer = fact[N] * invFact[n] % MOD * invFact[m] % MOD;
    cout << answer << '\n';
    return 0;
}
```

**Input**

```text
3 2
```

**Output**

```text
10
```

Five steps, of which three are east: $\binom{5}{3} = 10$.

**Complexity.** $O(n + m)$ preprocessing, $O(1)$ per query, $O(n+m)$ memory.

### The trap

The statement says "a grid of streets with $n$ avenues and $m$ streets", which is deliberately easy to misread: it is $n$ east steps and $m$ north steps, not $n - 1$ and $m - 1$. If the coordinates were given as corners $(0,0)$ to $(n,m)$, the answer would be $\binom{n+m}{n}$ with the coordinate differences. Read which convention the statement uses - it is the difference between an accepted and a wrong-answer submission on the sample's cousin.

Once obstacles appear, the same technique still applies but needs subtraction over obstacle pairs; that is a Level 6 problem and it is built entirely from this binomial.

</details>

---

## 7. Stars and bars: three variants, and how to tell them apart

The problem: distribute $k$ **identical** items among $n$ **distinct** recipients.

| Constraint | Count | Why |
|---|---|---|
| each may get zero | $\binom{k+n-1}{n-1}$ | arrange $k$ stars and $n-1$ bars |
| each must get at least one | $\binom{k-1}{n-1}$ | give one each first, then distribute the rest |
| each gets at most $u$ | inclusion-exclusion, §9 | subtract the overfull cases |

The formula is a bijection: a distribution is a row of $k$ stars with $n-1$ bars inserted, and the $i$-th gap between bars is recipient $i$'s share. Two small but decisive details:

- The **recipients are distinct** (children, boxes, days) while the **items are identical** (candies, coins). If the items are distinct, this is a completely different problem - a function from items to recipients, counted by $n^k$.
- "At least one" costs one item per recipient, so subtract $n$ first; if $k < n$ the answer is $0$.

A useful restatement: the number of solutions of $x_1 + \cdots + x_n = k$ in non-negative integers is $\binom{k+n-1}{n-1}$, and in positive integers is $\binom{k-1}{n-1}$.

```cpp
// Solutions of x1 + ... + xn = k with xi >= 0, modulo prime MOD (k + n < MOD).
long long weakCompositions(int k, int n, const vector<long long>& fact,
                           const vector<long long>& invFact) {
    return fact[k + n - 1] * invFact[n - 1] % (long long)1000000007 * invFact[k] % (long long)1000000007;
}
```

---

## 8. Solved Problem 3 — The Sweet Distribution

> **Statement.** A teacher has a jar of $k$ identical sweets and $n$ children sitting in a row, and she hands out the entire jar, one child at a time. To avoid arguments, every child must receive at least one sweet. How many different distributions are possible? The jar can hold up to a million sweets, there can be up to a million children, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

Sweets are identical, children are distinct, so this is stars and bars with a positive lower bound. Hand one sweet to each child first, which uses $n$ sweets and is forced; the remaining $k - n$ sweets are then distributed freely among the $n$ children, which has $\binom{(k-n) + n - 1}{n-1} = \binom{k-1}{n-1}$ solutions.

If $k < n$, no distribution exists and the answer is $0$ - the formula would ask for a factorial of a negative index, so the check must come first.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

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
    int k, n;
    cin >> k >> n;

    if (k < n) { cout << 0 << '\n'; return 0; }

    int N = k;                          // largest factorial index needed is k - 1
    vector<long long> fact(N + 1), invFact(N + 1);
    fact[0] = 1;
    for (int i = 1; i <= N; ++i) fact[i] = fact[i - 1] * i % MOD;
    invFact[N] = power(fact[N], MOD - 2, MOD);
    for (int i = N; i >= 1; --i) invFact[i - 1] = invFact[i] * i % MOD;

    long long answer = fact[k - 1] * invFact[n - 1] % MOD * invFact[k - n] % MOD;
    cout << answer << '\n';
    return 0;
}
```

**Input**

```text
7 3
```

**Output**

```text
15
```

Hand one sweet to each child, leaving four; distribute four among three children in $\binom{4+2}{2} = 15$ ways.

**Complexity.** $O(k)$ preprocessing, $O(1)$ query, $O(k)$ memory.

### The trap

The frequent error is to treat "at least one" as a subtraction at the end: $\binom{k+n-1}{n-1} - (\text{something})$. That shortcut has no simple form - the correct adjustment is the shift $k \to k - n$, which is why the formula uses $k-1$ on top. Inclusion-exclusion for the "at least one" constraint would work, but it is needless work compared with the shift.

The second trap is the argument order in the code. `fact[k - 1]` with `k = n` gives `fact[n-1]`, and the answer is $\binom{n-1}{n-1} = 1$, which is correct: one sweet each. Off-by-one here produces a plausible zero or a crash on the boundary test.

</details>

---

## 9. Derangements and inclusion-exclusion

A **derangement** is a permutation with no fixed point: nobody gets their own hat. Write $D_n$ for the count.

The clean derivation is inclusion-exclusion over the set of people who *do* get their own hat. For a chosen set $S$ of $j$ people who are fixed, the remaining $n - j$ can be arranged in $(n-j)!$ ways, so

$$D_n = \sum_{j=0}^{n} (-1)^j \binom{n}{j} (n-j)! = n! \sum_{j=0}^{n} \frac{(-1)^j}{j!}.$$

The sum converges to $1/e$, so $D_n \approx n!/e$ - derangements are about $37\%$ of all permutations for any large $n$. That is a good sanity check on any answer you compute.

The recurrence is usually better for programming, because it needs no division and works modulo any modulus:

$$D_0 = 1, \quad D_1 = 0, \quad D_n = (n-1)\left(D_{n-1} + D_{n-2}\right).$$

The combinatorial reason: person $1$ must go to someone else's place, say person $k$'s ($n-1$ choices); then either $k$ goes to $1$'s place - leaving a derangement of the other $n-2$ - or $k$ does not, which is a derangement of $n-1$ people with $1$'s place renamed to $k$'s.

```cpp
// Derangements modulo m, for all n up to N. O(N) time.
vector<long long> derangements(int N, long long m) {
    vector<long long> D(N + 1, 0);
    D[0] = 1 % m;
    if (N >= 1) D[1] = 0;
    for (int n = 2; n <= N; ++n)
        D[n] = (n - 1LL) * ((D[n - 1] + D[n - 2]) % m) % m;
    return D;
}
```

**The same inclusion-exclusion, generalised.** The number of permutations with **exactly** $k$ fixed points is

$$\binom{n}{k} D_{n-k}:$$

choose which $k$ people are fixed, then derange the rest. This pattern - "exactly $k$" becomes "choose $k$, then force the rest away from a property" - is worth memorising as a template.

---

## 10. Solved Problem 4 — The Secret Santa

> **Statement.** A class of $n$ students draws names for a gift exchange. The rules require that nobody draws their own name, and the organiser wants to know how many valid assignments of givers to receivers there are. Students are distinguishable, names are drawn without replacement, and each student gives exactly one gift and receives exactly one. The class can have up to one million students, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

An assignment is a permutation: giver $i$ is matched to receiver $\pi(i)$, and the condition "nobody draws their own name" says $\pi(i) \ne i$ for every $i$. That is the definition of a derangement, so the answer is $D_n$.

The bound $n \le 10^{6}$ and the modulus rule out the inclusion-exclusion sum's divisions (they are legal here, since $p$ is prime and $n < p$, but the recurrence is cheaper and safer). The recurrence needs $O(n)$ time and $O(1)$ memory if you keep only the last two values - the code below keeps the full array only to make the shape obvious.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int n;
    cin >> n;

    vector<long long> D(n + 1, 0);
    D[0] = 1;
    if (n >= 1) D[1] = 0;
    for (int i = 2; i <= n; ++i)
        D[i] = (i - 1LL) * ((D[i - 1] + D[i - 2]) % MOD) % MOD;

    cout << D[n] << '\n';
    return 0;
}
```

**Input**

```text
4
```

**Output**

```text
9
```

The derangements of four items are $2143, 2341, 2413, 3142, 3412, 3421, 4123, 4312, 4321$ - nine of them, out of $24$ permutations.

**Complexity.** $O(n)$ time, $O(n)$ memory (or $O(1)$ with two variables).

### The trap

The base cases. $D_0 = 1$ is not a convention invented for the recurrence's convenience: there is exactly one way to assign nothing to nobody. $D_1 = 0$ because a single student must draw their own name. Start the recurrence with $D_1 = 1$ instead and you get $D_2 = 2$, then $D_4 = 11$ instead of $9$ - and the sample happens to catch that, so submissions die on the boundary inputs $n = 0$ and $n = 1$ instead, which no sample shows.

</details>

---

## 11. Probability spaces and expectation

A **probability space** is a finite set of outcomes $\Omega$ with weights $p(\omega) \ge 0$ summing to $1$. A **random variable** is a function $X : \Omega \to \mathbb{R}$, and its **expectation** is the weighted average

$$E[X] = \sum_{\omega \in \Omega} p(\omega)\, X(\omega).$$

Equivalently, group the outcomes by value: $E[X] = \sum_{v} v \cdot P(X = v)$.

For the uniform case - which is what most contest problems are - every outcome has probability $1/|\Omega|$ and expectation is just "sum over outcomes divided by the number of outcomes". That is worth stating because it removes the mystique: **expectation is an average**, and every expectation problem can be solved by summing a quantity over all outcomes and dividing.

The whole point of the next section is that you should almost never do that sum directly.

Two properties to keep straight:

- **Linearity.** $E[X + Y] = E[X] + E[Y]$, always, independent or not.
- **Monotone scaling.** $E[cX] = cE[X]$ for a constant $c$, and $E[c] = c$.

And one property that does **not** hold in general: $E[XY] \ne E[X]E[Y]$ unless $X$ and $Y$ are independent. Section 14 is about when you may and may not split a product.

---

## 12. Linearity of expectation: the technique that beats enumeration

Split a complicated random variable into a sum of indicators:

$$X = I_1 + I_2 + \cdots + I_m, \qquad I_j = \begin{cases} 1 & \text{event } A_j \text{ happens} \\ 0 & \text{otherwise}\end{cases}$$

Then $E[X] = \sum_j E[I_j] = \sum_j P(A_j)$. Each $P(A_j)$ is computed locally, without knowing anything about the other events - **independence is not required**. That sentence is the entire technique, and it is why expectation problems that look impossible become one-liners.

Three classic applications:

- **Fixed points.** A random permutation of $n$: for each $i$, $P(\pi(i) = i) = 1/n$, so the expected number of fixed points is $n \cdot 1/n = 1$, for every $n$.
- **Inversions.** A random permutation of $n$: for each pair $i < j$, $P(\pi(i) > \pi(j)) = 1/2$, so the expected number of inversions is $\binom{n}{2}/2 = n(n-1)/4$.
- **Coupon-ish counts.** The expected number of positions where a random string matches a pattern: one indicator per alignment, each with probability $2^{-k}$.

The art is choosing the indicators. They should be **simple events** whose probability you can compute in isolation - usually indexed by a position, a pair, or a subset of fixed size. If your indicator's probability needs to know what other indicators did, you have chosen badly.

---

## 13. Solved Problem 5 — The Raffle Drum

> **Statement.** A raffle drum holds $n$ balls numbered $1$ through $n$. Two balls are drawn without replacement, and the prize is the absolute difference between their numbers. The organiser must publish the expected prize, but the charity's accountant insists on the result modulo a prime $p$: print the value of the expectation as an element of $\mathbb{Z}_p$. The drum can hold up to $10^{18}$ balls, and $p$ is a prime larger than $3$.

<details>
<summary>Solution</summary>

### Why this approach

The sample space is the set of unordered pairs, all equally likely, so

$$E\big[|X - Y|\big] = \frac{1}{\binom{n}{2}} \sum_{i<j} (j - i).$$

Group the pairs by difference: for $d = 1, \dots, n-1$ there are $n - d$ pairs with $j - i = d$. Hence

$$\sum_{i<j}(j-i) = \sum_{d=1}^{n-1} d(n-d) = \frac{n(n-1)(n+1)}{6},$$

and dividing by $\binom{n}{2} = \frac{n(n-1)}{2}$ gives the remarkably clean

$$E = \frac{n+1}{3}.$$

The bound $n \le 10^{18}$ rules out touching the pairs; the formula is one multiplication. The answer is a rational number, so printing it modulo $p$ means multiplying by the inverse of $3$.

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
    long long n, p;
    cin >> n >> p;

    long long inv3 = power(3, p - 2, p);        // p is prime and p > 3
    long long answer = (n % p + 1) % p * inv3 % p;
    cout << answer << '\n';
    return 0;
}
```

**Input**

```text
4 1000000007
```

**Output**

```text
666666673
```

For $n = 4$ the expectation is $5/3$, and $5 \cdot 3^{-1} \equiv 5 \cdot 333333336 \equiv 666666673 \pmod{10^9+7}$. The six unordered pairs have differences $1, 2, 3, 1, 2, 1$, averaging $10/6 = 5/3$.

**Complexity.** $O(\log p)$ time, $O(1)$ memory.

### The trap

The expectation is a **fraction**, not an integer. For $n = 4$ it is $5/3$, and no integer division produces it: `(n + 1) / 3` gives $1$, which is a completely different number. Modular expectation means "the fraction $A/B$, mapped to $A \cdot B^{-1} \bmod p$", and that map is legal exactly when the denominator is invertible.

The second trap is $n$ itself. Reduce before adding: `(n % p + 1) % p` is safe for any value a signed 64-bit type can hold, while `n + 1` is not, and the difference only shows up on a test you will not write yourself. The formula also needs $n \ge 2$; the drum guarantees it, but guard it if you reuse the code.

</details>

---

## 14. Expected products and independence

Two random variables are **independent** when $P(X = x, Y = y) = P(X = x)P(Y = y)$ for every pair of values. For independent variables,

$$E[XY] = E[X]\,E[Y],$$

and this is the only situation in which you may split a product. When variables are dependent, the product rule fails, sometimes spectacularly:

$$E[I_1 I_2] = P(A_1 \cap A_2) \quad\text{but}\quad E[I_1]E[I_2] = P(A_1)P(A_2),$$

and these differ exactly when the events are dependent. For a permutation's fixed points, $P(\pi(1)=1 \text{ and } \pi(2)=2) = 1/(n(n-1))$, while the product of probabilities is $1/n^2$; the two are close but not equal, and a wrong answer built on the second one will pass small tests or fail them, depending on rounding luck.

If a statement asks for $E[XY]$ with dependent $X, Y$, expand instead:

$$E[(X_1 + \cdots + X_m)(Y_1 + \cdots + Y_r)] = \sum_{i,j} E[X_i Y_j],$$

and compute each $E[X_i Y_j] = P(A_i \cap B_j)$ from the joint event - which is usually a small counting problem in itself. That expansion is still linearity; it is just linearity applied twice.

A useful sanity check: expectation is **additive regardless of dependence**, so any answer that needs the joint distribution must be a product *or* an explicit intersection. If you cannot point to which one you used, you have guessed.

---

## 15. Printing a rational answer modulo a prime

Expectations are rational. To print one modulo a prime $p$, write it as $A/B$ in lowest terms, then print $A \cdot B^{-1} \bmod p$. Two conditions:

- $p$ must not divide $B$ (otherwise no inverse exists). For $p > 3$ and denominators like $2, 3, 6$, this holds automatically; for a general $p$, check.
- The modulus must be prime, or at least coprime to $B$; the technique of Topic 02 applies exactly as it did for binomials.

A useful discipline when writing such a solution: derive the fraction **symbolically** first, then substitute. If you reduce modulo $p$ early, you lose the ability to cancel common factors in $A/B$, and the inverse you compute is of a residue class rather than of the actual denominator - usually fine when everything is coprime to $p$, but a trap when the problem's denominator is a multiple of $p$.

For probabilities as inputs - "coin $i$ shows heads with probability $a_i/b_i$" - each probability is itself a residue, and independence lets you multiply them directly. That is the shape of two of the unsolved problems below.

---

## 16. A pattern table for combinatorial statements

| The statement says | Model | Formula |
|---|---|---|
| exactly $k$ positions are special | choose which | $\binom{n}{k}$ |
| ordered list of $k$ from $n$ | arrange | $n!/(n-k)!$ |
| arrange $n$ distinct in a row | permute | $n!$ |
| arrange $n$ distinct in a circle | fix rotation | $(n-1)!$ |
| multiset arrangement (repeats) | multinomial | $n!/\prod c_i!$ |
| distribute identical items, no minimum | stars and bars | $\binom{k+n-1}{n-1}$ |
| distribute identical, at least one each | shift | $\binom{k-1}{n-1}$ |
| distribute identical, upper bound $u$ | inclusion-exclusion | $\sum_j (-1)^j \binom{n}{j}\binom{k-ju+n-1}{n-1}$ |
| no object in its own place | derangement | $D_n$ |
| exactly $k$ misplaced | choose + derange | $\binom{n}{k} D_{n-k}$ |
| at least one | complement | total $- $ none |
| expected count of local events | indicators | $\sum P(A_j)$ |
| expected product, independent | factorise | $E[X]E[Y]$ |

Read a statement, find its row, then check the three things this table hides: whether the objects are distinct, whether positions matter, and whether the modulus permits the division your formula needs.

---

## 17. Unsolved problems

Six problems. Two are one-formula, two need a table or a recurrence, and two are expectation problems where the difficulty is entirely in choosing the right decomposition.

### 17.1 The Medal Table

> A championship has $n$ athletes, and the organisers award a gold, a silver and a bronze medal to three different athletes. The medal table must be printed for the sponsors, with the number of possible podiums modulo $10^9+7$. There can be up to $10^9$ athletes.

### 17.2 The Seating Plan

> A banquet seats $n$ guests around one circular table, all facing the middle. Two seatings that differ only by rotating everybody one place are considered the same arrangement; reflections are different arrangements, because the table has a door. The host wants the number of distinct seatings modulo $10^9+7$. There can be up to $10^{6}$ guests.

### 17.3 The Donut Box

> A baker has $k$ identical donuts and $n$ children, and may hand out any number of donuts to each child, including none, as long as every donut is given away. The baker wants the number of possible distributions modulo $10^9+7$. Both $k$ and $n$ can be up to $10^{6}$.

### 17.4 The Locker Room

> A gym's lockers are numbered $1$ to $n$, and each of $n$ members is assigned a locker by a random but fixed permutation. The manager wants to count the permutations in which exactly $k$ members keep the locker matching their member number, modulo $10^9+7$. Both $n$ and $k$ can be up to $10^{6}$, with $k \le n$.

### 17.5 The Coin Toss

> A row of $n$ coins is tossed, one at a time. Coin $i$ comes up heads with probability $a_i/b_i$, independently of the others, where $0 \le a_i \le b_i < 10^9+7$ and $b_i \ge 1$. The organiser pays one rupee per head and wants the expected payout as a residue modulo $10^9+7$. There can be up to $2 \times 10^{5}$ coins.

### 17.6 The Card Pair

> A deck of $n$ distinct cards is shuffled uniformly at random into a row. A pair of positions $i < j$ is called an inversion if the card at $i$ is larger than the card at $j$. The dealer wants the expected number of inversions modulo $10^9+7$, for $n$ up to $10^{18}$.

Problem 6 is the cleanest test of §12: if you find yourself imagining permutations, you will not finish; if you find yourself counting pairs, you are done in two lines.

---

## Key Takeaways

1. Combinatorics is translation first: decide whether objects are distinct, whether order matters, and whether repetition is allowed, before writing any formula.
2. $\binom{n}{k} = \frac{n!}{k!(n-k)!}$ counts subsets; the "choose which positions" reading is the one that generalises to most problems.
3. "At least one" is almost always "total minus none". Complement counting is the highest-yield trick in this topic.
4. Stars and bars: no minimum gives $\binom{k+n-1}{n-1}$; at least one gives $\binom{k-1}{n-1}$; upper bounds need inclusion-exclusion.
5. Derangements satisfy $D_n = (n-1)(D_{n-1} + D_{n-2})$ with $D_0 = 1$, $D_1 = 0$, and $D_n \approx n!/e$.
6. "Exactly $k$ fixed points" is $\binom{n}{k} D_{n-k}$ - the template for every "exactly $k$" problem with a forbidden-position constraint.
7. Expectation is a weighted average, but you should compute it with indicators: $E[X] = \sum P(A_j)$, with no independence needed.
8. $E[XY] = E[X]E[Y]$ only for independent variables; for dependent ones, expand the product and compute joint probabilities.
9. Rational answers modulo $p$ require the denominator to be invertible: reduce the fraction symbolically before mapping it into $\mathbb{Z}_p$.
10. If $n$ is up to $10^{18}$ and the answer is a count, there is a closed form - the constraints are telling you to find it, not to loop.

---

## Common Mistakes & Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| using $n!$ when the statement wants subsets | answers too large by $k!$ per selection | decide ordered vs unordered first |
| multinomial without inverting the denominators | integer division of residues | multiply by inverse factorials |
| stars and bars for distinct items | formula wildly too small | distinct items mean $n^k$ or inclusion-exclusion |
| forgetting the "at least one" shift | answer $\binom{k+n-1}{n-1}$ instead of $\binom{k-1}{n-1}$ | subtract $n$ from $k$ first |
| $D_1 = 1$ | every derangement wrong | $D_0 = 1$, $D_1 = 0$ |
| counting circular arrangements as $n!$ | factor of $n$ too large | $(n-1)!$ |
| treating a reflection as a rotation | factor of $2$ off | read whether the table has a distinguished side |
| applying $E[XY] = E[X]E[Y]$ to dependent events | small errors that pass samples | expand and use joint probabilities |
| printing a fraction with integer division | truncated value, often $0$ or $1$ | multiply by the modular inverse |
| computing $\binom{n}{k}$ with $n \ge p$ using factorials | inverse of $0$ | Lucas' theorem, or a different modulus |
| off-by-one in lattice paths | $\binom{n+m-1}{n}$ instead of $\binom{n+m}{n}$ | count the steps, not the corners |
| allocating factorials up to $n$ when $n = 10^{18}$ | crash | closed form, or a smaller derived bound |

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Banner Line | multinomial with repeated colours | §4 |
| 2 | The Staircase Delivery | lattice paths as one binomial | §6 |
| 3 | The Sweet Distribution | stars and bars with a positive bound | §8 |
| 4 | The Secret Santa | derangement recurrence | §10 |
| 5 | The Raffle Drum | expectation as a closed-form fraction | §13 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Medal Table | $n \le 10^{9}$ | a falling factorial modulo $p$ |
| 2 | The Seating Plan | $n \le 10^{6}$ | circular permutations |
| 3 | The Donut Box | $k, n \le 10^{6}$ | stars and bars, no minimum |
| 4 | The Locker Room | $n, k \le 10^{6}$ | choose the fixed points, derange the rest |
| 5 | The Coin Toss | $n \le 2 \cdot 10^{5}$ | linearity plus per-coin inverses |
| 6 | The Card Pair | $n \le 10^{18}$ | one indicator per pair, then a closed form |

Problems 4 and 6 are the ones that separate "knows the formulae" from "can model". Problem 4 needs two tables (binomials and derangements) composed correctly; Problem 6 needs you to stop thinking about permutations altogether.

---

## Next Up

Topic 04 opens the other half of this chapter: **dynamic programming**. The connection to what you just did is direct - many of the counts above are recurrences in disguise ($D_n$, Pascal's identity, the lattice-path count), and dynamic programming is the systematic way to evaluate a recurrence when no closed form exists. If you can see why $\binom{n}{k} = \binom{n-1}{k-1} + \binom{n-1}{k}$ is "a DP table read along a diagonal", you are already thinking in the right shape.

