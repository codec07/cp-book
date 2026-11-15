# Topic 04 — Dynamic Programming Part 1: Introduction

Everyone meets dynamic programming twice. The first time it looks like a trick for Fibonacci; the second time it looks like the only way to solve anything. This topic is the first meeting, done properly, because the habits formed here - how to name a state, how to order the loops, when a table is worth $10^{6}$ cells - are what the next five topics build on.

The core idea is small enough to state in one sentence: **if the same subproblem keeps appearing, solve it once and remember the answer.** What makes DP hard is not the remembering; it is deciding *what the subproblems are*. Get the state right and the transition usually writes itself. Get it wrong and you will spend an hour writing correct code for the wrong recurrence.

> **Prerequisites:** recursion, arrays, and loops. Topic 03 helps for the counting problems but is not required - a DP solution is usually easier to read than the formula it replaces.
>
> **You will be able to:** convert a recursive definition into a memoised solution and then into a table; choose loop orders that distinguish counting combinations from permutations; reconstruct an optimal solution from the table; and recognise the four conditions under which DP is the wrong tool.

---

## 1. What dynamic programming actually is

Dynamic programming applies when a problem has two properties:

- **Overlapping subproblems.** The same smaller instance is needed more than once. Naive recursion recomputes it, which is where the exponential blow-up comes from.
- **Optimal substructure.** The answer for the whole can be assembled from answers to the parts, without needing to know how those answers were obtained.

Compare with divide and conquer, which also splits the problem: merge sort splits into halves that never overlap, so remembering is useless there. DP's distinguishing feature is *sharing*.

The classic demonstration is Fibonacci: $F_n = F_{n-1} + F_{n-2}$ is a two-line recursive function that takes exponential time, because it recomputes $F_{n-2}$ twice, $F_{n-3}$ three times, and so on - the number of calls is itself Fibonacci. Storing each value once drops the cost to $O(n)$.

That ratio - exponential to linear by writing down answers - is the reason DP is the single most valuable technique in this chapter.

---

## 2. From recursion to memoisation

Start with the recursive definition, verbatim, and add a table:

```cpp
vector<long long> memo;

long long fib(int n) {
    if (n <= 1) return n;
    long long &m = memo[n];
    if (m != -1) return m;
    return m = fib(n - 1) + fib(n - 2);
}

// call: memo.assign(n + 1, -1); fib(n);
```

This is called **top-down** DP, or memoisation. Three details are worth noticing:

- The check `if (m != -1)` must come *after* the base case, otherwise the sentinel leaks into the answer.
- A reference `long long &m` lets you write the result once, on the `return` line, which removes the possibility of forgetting to store it.
- The sentence "solve it once" is enforced by the check, not by the recursion. Without it, nothing improves.

The advantage is that memoisation only visits states the recursion actually needs. For Fibonacci that is all $n$ states; for a knapsack it can be a small fraction of the table. The disadvantage is stack depth and function-call overhead, which rarely matters at $n \le 10^{6}$ but does at $n \le 10^{7}$.

---

## 3. From memoisation to tabulation

The **bottom-up** version fills the same table in an order that guarantees every dependency is ready:

```cpp
long long fibTable(int n) {
    if (n <= 1) return n;
    vector<long long> f(n + 1);
    f[0] = 0; f[1] = 1;
    for (int i = 2; i <= n; ++i) f[i] = f[i - 1] + f[i - 2];
    return f[n];
}
```

And since each value depends only on the two before it, the table can shrink to two variables:

```cpp
long long fibConstant(int n) {
    long long a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        long long c = a + b;
        a = b;
        b = c;
    }
    return n == 0 ? 0 : b;
}
```

The three versions are the same algorithm at three levels of memory. In a contest, write the third only when memory is the binding constraint; the second is easier to debug, and the first is easiest to derive. A useful workflow: **derive recursively, submit iteratively, optimise memory last.**

### Which direction should you use?

| Situation | Prefer | Why |
|---|---|---|
| State space sparse, most states unreachable | top-down | visits only what is needed |
| $n$ large ($10^{7}$) and states dense | bottom-up | no recursion depth, no call overhead |
| Transition needs an index order that is not obvious | top-down | you do not have to sort it out |
| Memory must be reduced to a rolling window | bottom-up | the order is explicit |
| Reconstruction needed | either | parent pointers work in both |

---

## 4. Solved Problem 1 — The Staircase of Steps

> **Statement.** A caretaker climbs a tower staircase of $n$ steps. At each stride he goes up either one step or two steps, never down and never skipping more than two. The tower's guidebook lists every distinct sequence of strides a visitor could make from the bottom to the top, and it must be reprinted each year. How many sequences are there? The tower has up to one million steps, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

The last stride either covered one step or two. If it covered one, the climber was at step $n-1$ and anything that gets there works; if two, he was at $n-2$. So the number of sequences satisfies

$$f(n) = f(n-1) + f(n-2), \qquad f(0) = 1, \quad f(1) = 1.$$

This is Fibonacci with different initial conditions. The base case $f(0) = 1$ says "there is one way to climb nothing: do nothing", which is the kind of boundary condition that decides whether a submission passes.

The recurrence is $O(n)$ states with $O(1)$ work each, so the table is linear and even rolling memory suffices.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int n;
    cin >> n;

    long long prev2 = 1;                 // f(0)
    long long prev1 = 1;                 // f(1)

    for (int i = 2; i <= n; ++i) {
        long long cur = (prev1 + prev2) % MOD;
        prev2 = prev1;
        prev1 = cur;
    }

    cout << (n == 0 ? 1 : prev1) << '\n';
    return 0;
}
```

**Input**

```text
5
```

**Output**

```text
8
```

The eight sequences for five steps: $11111$, $1112$, $1121$, $1211$, $2111$, $122$, $212$, $221$.

**Complexity.** $O(n)$ time, $O(1)$ memory.

### The trap

The initial conditions. With $f(0)=1, f(1)=1$ the sequence is $1, 1, 2, 3, 5, 8$; with the Fibonacci convention $f(1)=1, f(2)=2$ you get the same numbers shifted, and a program written for one convention returns the other's answer on $n = 1$. Derive the base cases from the *meaning* ("ways to climb zero steps"), not from a remembered formula.

The second trap is `int` overflow in the addition: two residues below $10^9+7$ sum to below $2 \times 10^{9}+14$, which exceeds a signed 32-bit int. Use `long long`, or subtract the modulus conditionally.

</details>

---

## 5. The four questions of DP design

Every DP solution answers these four questions. Write them down explicitly - literally, on paper - until the habit is automatic.

1. **What does the state mean?** "$f(i)$ is the number of ways to reach step $i$" is a state. "The answer for the prefix of length $i$" is a state only if you say *what* about the prefix you are answering.
2. **What are the transitions?** Which predecessors can produce this state, and with what cost?
3. **What are the base cases?** The smallest states, computed directly. They must be consistent with the recurrence, not with a formula you remember.
4. **In what order are states computed?** Every transition must read states that are already final. This question is where bottom-up solutions break.

A fifth question, asked less often and worth asking anyway: **how big is the state space?** A DP is only as fast as (number of states) × (work per state). If that product exceeds about $10^{8}$, the state is wrong - no amount of coding will rescue it.

The state is the hard part. Two disciplines help:

- Prefer a state that names a *prefix* of the input (or a subarray, or a subset). Prefix states usually give acyclic transitions.
- If the state feels like "everything relevant so far", it is too vague. Write the transition first and let it tell you what the state must remember.

---

## 6. Solved Problem 2 — The Café's Token Jar

> **Statement.** A café sells tokens worth $1$, $2$ or $5$ units and lets customers pay any total exactly using any number of tokens. The owner wants to print, for each total, how many distinct *multisets* of tokens add up to it - the order in which a customer hands over the tokens does not matter. For a total $T$ and a token list, how many multisets are there? The total is at most $10^{5}$, the number of token values at most $100$, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

Let `dp[s]` be the number of multisets of the *first $i$ token values* summing to $s$. Adding the $i$-th value $c$ either does not use it - then `dp[s]` carries over - or uses it at least once, in which case the rest is a multiset of the *same* first $i$ values summing to $s - c$. That self-reference is what "multiset" means, and it is why the inner loop runs **upward**:

```text
for each coin c:
    for s = c .. T:
        dp[s] += dp[s - c]
```

Running the target upward allows the same coin to be used repeatedly within the same coin loop, and iterating coins on the outside fixes the order in which coins are introduced, so multisets are counted once. Swapping the loops counts orderings instead - see §7, which is the single most consequential line of code in introductory DP.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int n, T;
    cin >> n >> T;

    vector<int> coin(n);
    for (int i = 0; i < n; ++i) cin >> coin[i];

    vector<long long> dp(T + 1, 0);
    dp[0] = 1;                                  // the empty multiset

    for (int c : coin)
        for (int s = c; s <= T; ++s) {
            dp[s] += dp[s - c];
            if (dp[s] >= MOD) dp[s] -= MOD;
        }

    cout << dp[T] << '\n';
    return 0;
}
```

**Input**

```text
3 5
1 2 5
```

**Output**

```text
4
```

The multisets are $\{5\}$, $\{2,2,1\}$, $\{2,1,1,1\}$, $\{1,1,1,1,1\}$.

**Complexity.** $O(nT)$ time, $O(T)$ memory.

### The trap

The loop order. Coins outside and sums inside counts multisets; sums outside and coins inside counts **sequences**, so $\{1,2\}$ and $\{2,1\}$ become different answers. Both are correct answers to different questions, which is why statements use the words "ways to make the amount" (usually sequences, in a machine that dispenses coins in order) and "sets of coins" (multisets). Read the sentence, then choose the loop.

The second trap is the empty total: `dp[0] = 1` is not a convention. There is exactly one way to make zero - take nothing - and every recurrence above depends on it.

</details>

---

## 7. Loop order decides what you are counting

The two loops in the previous problem are the same two loops in a different order, and they answer different questions. Keep both skeletons:

```cpp
// Multisets: coins outside, sums ascending.
for (int c : coin)
    for (int s = c; s <= T; ++s)
        dp[s] = (dp[s] + dp[s - c]) % MOD;
```

```cpp
// Ordered sequences: sums outside, coins inside.
for (int s = 1; s <= T; ++s)
    for (int c : coin)
        if (c <= s)
            dp[s] = (dp[s] + dp[s - c]) % MOD;
```

Notice what the second one says: "$dp[s]$ is the number of ordered sequences summing to $s$, because the last token can be any coin." That is a different state description, and the loop order is a consequence of it, not a trick. Once you can state the meaning of `dp[s]` in one sentence that matches the loops, you will stop swapping them by accident.

A third variant, for **minimisation**, is order-insensitive because taking a minimum is idempotent:

```cpp
// Minimum number of coins, order irrelevant.
for (int s = 1; s <= T; ++s) {
    dp[s] = INF;
    for (int c : coin)
        if (c <= s && dp[s - c] + 1 < dp[s]) dp[s] = dp[s - c] + 1;
}
```

---

## 8. Solved Problem 3 — The Arcade's Refund

> **Statement.** An arcade owes a customer a refund of $T$ rupees and must pay it using coins of the denominations it keeps in the till. The manager wants to hand over as few coins as possible; the order in which the coins are handed over is irrelevant. If the refund cannot be made exactly, the manager must report that. The refund is at most $10^{5}$ rupees and the till holds at most $100$ denominations.

<details>
<summary>Solution</summary>

### Why this approach

Let `dp[s]` be the minimum number of coins summing exactly to $s$, with `dp[0] = 0` and `dp[s] = ∞` for unreachable $s$. Then

$$dp[s] = 1 + \min_{c \le s} dp[s - c].$$

This is a shortest-path problem on a line, and the table is its Bellman-style relaxation in increasing order of $s$. Unlike the counting version, the loop order over coins does not matter, because minimum is commutative and idempotent - a fact that reassures you the recurrence is right even if you are unsure about ordering.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, T;
    cin >> n >> T;

    vector<int> coin(n);
    for (int i = 0; i < n; ++i) cin >> coin[i];

    const int INF = 1e9;
    vector<int> dp(T + 1, INF);
    dp[0] = 0;

    for (int s = 1; s <= T; ++s)
        for (int c : coin)
            if (c <= s && dp[s - c] + 1 < dp[s])
                dp[s] = dp[s - c] + 1;

    cout << (dp[T] == INF ? -1 : dp[T]) << '\n';
    return 0;
}
```

**Input**

```text
3 11
1 2 5
```

**Output**

```text
3
```

Eleven rupees as $5 + 5 + 1$.

**Complexity.** $O(nT)$ time, $O(T)$ memory.

### The trap

The infinity sentinel. `INF + 1` overflows if `INF` is `INT_MAX`, and the comparison then silently accepts a garbage value. Use `1e9` (or `INT_MAX / 2`) and test `dp[s - c] < INF` before adding, or make the sentinel small enough that `INF + 1` is still representable.

The second trap is the answer for an impossible refund: printing `INF` is not "report that it is impossible". Statements rarely say "print -1" in bold; they say "if the refund cannot be made exactly, the manager must report it", and the output section tells you the format.

</details>

---

## 9. Two-dimensional states: paths on a grid

The next step is a state indexed by two things. Counting monotone paths is the canonical example, and it introduces the idea that a state can be a *position* rather than a prefix.

```cpp
// Number of monotone paths from (0,0) to (r-1,c-1), moving right or down.
vector<vector<long long>> dp(r, vector<long long>(c, 0));
dp[0][0] = 1;
for (int i = 0; i < r; ++i)
    for (int j = 0; j < c; ++j) {
        if (i == 0 && j == 0) continue;
        if (i > 0) dp[i][j] += dp[i - 1][j];
        if (j > 0) dp[i][j] += dp[i][j - 1];
    }
```

The loop order is not arbitrary: `dp[i][j]` reads from above and from the left, so rows must run downward and columns rightward. Swap them and you read uninitialised cells - the classic silent zero.

Notice also that this DP is computing binomial coefficients in disguise. That is the promised bridge from Topic 03: the same numbers, one counted with a table and one with factorials.

---

## 10. Solved Problem 4 — The Rooftop Courier

> **Statement.** A courier crosses a rectangular rooftop divided into cells. Some cells are skylights and cannot be walked on; every other cell can. The courier starts at the north-west cell, ends at the south-east cell, and may only move one cell east or one cell south per step. The rooftop can be up to $1000 \times 1000$ cells and at least one cell - the start - is walkable. How many routes are there, modulo $10^9+7$?

<details>
<summary>Solution</summary>

### Why this approach

Let `dp[i][j]` be the number of routes from the start to cell $(i,j)$. A route arrives either from the north or from the west, so `dp[i][j] = dp[i-1][j] + dp[i][j-1]`, with skylights forced to $0$. The start cell is $1$ (the empty route), and the first row and column have only one predecessor each - which the general recurrence handles if the out-of-grid terms are zero.

State space $10^{6}$, work per state constant, so the table is fine inside a 256 MB limit; two rows would be enough if it were not, which is the §12 optimisation.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int r, c;
    cin >> r >> c;

    vector<string> grid(r);
    for (int i = 0; i < r; ++i) cin >> grid[i];

    vector<vector<long long>> dp(r, vector<long long>(c, 0));

    for (int i = 0; i < r; ++i)
        for (int j = 0; j < c; ++j) {
            if (grid[i][j] == '#') { dp[i][j] = 0; continue; }
            if (i == 0 && j == 0) { dp[i][j] = 1; continue; }
            if (i > 0) dp[i][j] = (dp[i][j] + dp[i - 1][j]) % MOD;
            if (j > 0) dp[i][j] = (dp[i][j] + dp[i][j - 1]) % MOD;
        }

    cout << dp[r - 1][c - 1] << '\n';
    return 0;
}
```

**Input**

```text
3 3
...
.#.
...
```

**Output**

```text
2
```

Without the blocked centre there would be $\binom{4}{2} = 6$ routes; four of them pass through the centre, leaving two that go around it.

**Complexity.** $O(rc)$ time and memory.

### The trap

The start cell. If you set `dp[0][0] = 1` before the loop *and* also add `dp[i-1][j]`/`dp[i][j-1]` on the first row's cells, you double count one boundary cell - which is invisible at $3 \times 3$ and wrong at $1000 \times 1000$. The `continue` after initialising the start is what makes the general recurrence safe.

A second snag: if the start is a skylight the answer is $0$, and the loop as written gives $0$ correctly only because the `#` check comes first. Order of conditions matters.

</details>

---

## 11. Solved Problem 5 — The Dice Tally

> **Statement.** A game master rolls $n$ standard six-sided dice and records the total. She wants a table of how many distinct rolls produce each total, for the purposes of balancing an encounter. How many rolls produce the total $S$? Both $n$ and $S$ can be up to $5000$, only totals between $n$ and $6n$ are possible, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

The state needs two pieces of information: how many dice have been rolled, and what they sum to. Let `dp[i][s]` be the number of rolls of $i$ dice totalling $s$. The last die shows some face $f \in [1,6]$, so

$$dp[i][s] = \sum_{f=1}^{6} dp[i-1][s-f].$$

The state space is $nS \le 25 \times 10^{6}$ and each transition is six additions - comfortably inside the limit, and a good illustration of the "states × work" budget.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int n, S;
    cin >> n >> S;

    vector<vector<long long>> dp(n + 1, vector<long long>(S + 1, 0));
    dp[0][0] = 1;

    for (int i = 1; i <= n; ++i)
        for (int s = i; s <= S; ++s) {
            long long ways = 0;
            for (int f = 1; f <= 6 && f <= s; ++f)
                ways += dp[i - 1][s - f];
            dp[i][s] = ways % MOD;
        }

    cout << dp[n][S] << '\n';
    return 0;
}
```

**Input**

```text
3 8
```

**Output**

```text
21
```

Triples from $\{1,\dots,6\}$ summing to eight: enumerating the first die gives $6 + 5 + 4 + 3 + 2 + 1 = 21$.

**Complexity.** $O(nS \cdot 6)$ time, $O(nS)$ memory - reducible to $O(S)$ by keeping only the previous layer.

### The trap

The lower bound of the sum loop. Starting at $s = 1$ instead of $s = i$ is harmless (those states are zero) but wastes time; more importantly, starting at $s = 0$ with a careless `dp[i-1][s-f]` index can read negative indices, which in C++ is undefined behaviour rather than a crash. Guard with `f <= s`, as above.

The second trap is accumulation before reduction: summing six values each below $10^9+7$ reaches $6 \times 10^{9}$, which overflows a 32-bit `int`. Accumulate in `long long` and reduce once, or reduce inside the loop.

</details>

---

## 12. Reconstruction: recovering the choices

A DP table tells you the optimal *value*. Usually the statement only asks for the value, but sometimes it asks for the solution itself - which coins, which path, which items. The standard technique is a **parent array**: when a transition improves a state, record which predecessor produced it.

```cpp
// Minimum cost to climb n steps, each stride 1 or 2, with a parent pointer.
vector<int> parent;

long long climb(const vector<long long>& cost) {
    int n = (int)cost.size() - 1;
    vector<long long> dp(n + 1, LLONG_MAX / 4);
    parent.assign(n + 1, -1);
    dp[0] = 0;

    for (int i = 1; i <= n; ++i)
        for (int step = 1; step <= 2 && step <= i; ++step)
            if (dp[i - step] + cost[i] < dp[i]) {
                dp[i] = dp[i - step] + cost[i];
                parent[i] = i - step;
            }
    return dp[n];
}

// Path from n back to 0, then reverse.
vector<int> pathFrom(int n) {
    vector<int> path;
    for (int v = n; v != -1; v = parent[v]) path.push_back(v);
    reverse(path.begin(), path.end());
    return path;
}
```

Two rules keep reconstruction honest:

- Record the parent **at the moment of improvement**, in the same branch as the assignment. Updating the value in one place and the parent in another is how reconstructed answers drift away from the computed optimum.
- Reconstruct **backwards from the final state**, following parents to a base case. Do not try to walk forward: the table does not know the future, and ties have to be resolved consistently (any tie-breaking rule is fine, as long as it is the same one during reconstruction).

When several optimal solutions exist, a judge checking a particular one will reject yours; statements that ask for "the" optimal solution usually accept any, but this is worth verifying before spending an hour on a rejected submission.

---

## 13. Memory: rolling arrays and in-place updates

A DP's memory is the size of the state space. When that is too large, the dependency pattern usually allows a window.

**Two-dimensional to one-dimensional.** If `dp[i][j]` depends only on row $i-1$ and the current row, keep two rows and swap them:

```cpp
// Grid paths with only two rows of memory.
vector<long long> prev(c, 0), cur(c, 0);
for (int i = 0; i < r; ++i) {
    for (int j = 0; j < c; ++j) {
        if (grid[i][j] == '#') { cur[j] = 0; continue; }
        cur[j] = ((i == 0 && j == 0) ? 1 : 0)
               + (i > 0 ? prev[j] : 0)
               + (j > 0 ? cur[j - 1] : 0);
        cur[j] %= MOD;
    }
    swap(prev, cur);
}
// answer in prev[c-1] after the last swap
```

**In-place updates.** If `dp[j]` depends on `dp[j - w]` from the *previous* layer only, iterate `j` downward so the old value is read before being overwritten. That is exactly the knapsack trick of Topic 05, and it is the reason knapsack uses one array while coin-change-counting uses an upward loop.

**Compression by symmetry.** If the table is symmetric (`dp[i][j] == dp[j][i]`), store half. If it is triangular, store a triangular array. These are worth doing only when memory is the stated constraint; premature compression makes debugging harder and introduces index bugs that cost more than the bytes saved.

The guideline: **memory is the second constraint after time**, and only when the state count is above about $10^{7}$ do you start folding arrays. Below that, clarity wins.

---

## 14. Complexity: states times transitions

The cost of a DP is

$$\underbrace{|\text{states}|}_{\text{table size}} \times \underbrace{\text{work per state}}_{\text{transition cost}}.$$

| Problem | States | Work per state | Total |
|---|---|---|---|
| Staircase | $n$ | $2$ | $O(n)$ |
| Coin multisets | $T$ | $1$ per coin, so $O(n)$ | $O(nT)$ |
| Minimum coins | $T$ | $O(n)$ | $O(nT)$ |
| Grid paths | $rc$ | $2$ | $O(rc)$ |
| Dice tally | $nS$ | $6$ | $O(nS)$ |
| Knapsack (Topic 05) | $nW$ | $1$ | $O(nW)$ |
| LIS, naive (Topic 07) | $n$ | $n$ | $O(n^{2})$ |
| LIS, clever (Topic 07) | $n$ | $\log n$ | $O(n \log n)$ |

Two observations that decide contest outcomes:

- **State count is the enemy, not the transition.** Reducing $nS$ to $S$ by rolling rows rarely changes the time, but reducing $n^{2}$ states to $n$ states is the difference between TLE and accepted.
- **When a DP is too slow, the fix is a better state or a smarter transition, not micro-optimisation.** Prefix minima, monotone queues, and divide-and-conquer optimisation all attack the same product from the "work per state" side; they are Level 4-5 material.

---

## 15. When DP is the wrong tool

DP is powerful enough to tempt you into using it everywhere. Four situations where it is a mistake:

1. **Greedy is provably optimal.** Interval scheduling, the coin system $\{1, 5, 10, 25\}$, and Huffman coding all admit greedy proofs and run in $O(n \log n)$; a DP over them is correct but slower, and usually memory-heavy.
2. **The state space is too large.** "Number of subsets with sum $S$" for $n = 100$, $S = 10^{9}$ has $10^{11}$ states. No compression saves it; the intended solution uses meet-in-the-middle (Level 6) or number theory.
3. **There is no optimal substructure.** If the best solution to the whole uses a *suboptimal* solution to a part - which happens with path constraints like "no three consecutive" combined with another global condition - local optimality is not enough, and the state must be enriched before it becomes DP.
4. **The dependency graph has cycles.** A DP needs an acyclic order. Grids with all four directions, or states that can transition back into themselves with positive cost, need shortest-path algorithms (Dijkstra, Bellman-Ford) instead - which are DP over a graph with a different evaluation order.

A useful heuristic: if the transition graph is a DAG and each state's answer is a function of its predecessors, it is DP. If you cannot exhibit the order, you probably have a graph problem in disguise.

---

## 16. Unsolved problems

Five problems. The first two are one-dimensional tables and test your loop order and your base cases; the last three need a second dimension in the state, and the last one needs you to invent the state yourself rather than copy this topic's examples.

### 16.1 The Broken Staircase

> A tower has $n$ steps, and the caretaker climbs one or two steps at a time. This year several steps are under repair and cannot be stepped on, so any route touching them is invalid. Given the list of broken steps, how many valid routes are there from the ground to the top? The tower has at most $10^{6}$ steps, at most $10^{5}$ of them broken, and the count is printed modulo $10^9+7$. The ground and the top are never broken.

### 16.2 The Toll Road

> A road has $n+1$ toll booths numbered $0$ to $n$. A traveller starts at booth $0$ and must reach booth $n$, paying the toll at every booth they stop at; booth $0$ is free. From booth $i$ the traveller may jump to any booth $i+1, \dots, i+K$. Find the minimum total toll. There can be $2 \times 10^{5}$ booths, $K \le 100$, and tolls up to $10^{9}$.

### 16.3 The Scribe's Correction

> A scribe has copied a word from a damaged manuscript and knows that the original differs from the copy only by single-character mistakes: a character may be missing, extra, or written as another character. The head archivist wants the smallest number of single-character edits that turns the copy into the original - insertions, deletions and substitutions each count as one. Both words are at most $5000$ characters long and use only lowercase letters.

### 16.4 The Subset Ledger

> An accountant has $n$ receipts with whole-rupee amounts, and must determine whether some subset of them sums exactly to a target $S$. Each receipt may be used at most once. There are at most $200$ receipts, amounts up to $10^{5}$, and $S$ up to $2 \times 10^{5}$.

### 16.5 The Domino Wall

> A wall is $3$ cells high and $n$ cells wide, and the mason tiles it completely with $2 \times 1$ and $1 \times 2$ dominoes, no overlaps and no gaps. How many tilings are there, modulo $10^9+7$? The wall can be up to $1000$ cells wide.

Problem 16.5 is the one that requires a new state: a single index is not enough, because a tiling's remaining shape depends on which cells of the current column are already covered. If you need a hint, think about the *profile* of a column.

---

## Key Takeaways

1. DP needs overlapping subproblems and optimal substructure; if either is missing, you are doing something else - and it will be slower or wrong.
2. Derive recursively, submit iteratively, compress memory last. The order matters: the recursion makes the state obvious, the table makes the order obvious, the rolling array makes the memory obvious.
3. The four questions - state meaning, transitions, base cases, evaluation order - should be answered on paper before any code is written.
4. Cost is (number of states) × (work per state). When a DP is too slow, change the state or the transition; do not micro-optimise.
5. Loop order is semantics: coins outside with an ascending sum counts multisets, sums outside with coins inside counts sequences.
6. `dp[0] = 1` for counting and `dp[0] = 0` for minimisation are base cases with meanings, not conventions. Derive them from "what does the empty case mean".
7. A sentinel infinity must be small enough that `INF + 1` does not overflow, and must be distinguishable from a legitimate answer.
8. Two-dimensional tables usually compress to one or two rows; iterate downward when a state reads its own previous value, upward when it reads the current one.
9. Reconstruction uses parent pointers written at the moment of improvement, then a backward walk from the final state.
10. If you cannot name the order in which states are finalised, the problem is probably a graph problem - reach for Dijkstra or Bellman-Ford, not for a table.

---

## Common Mistakes & Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| base case from a remembered formula | off by one on $n = 0, 1$ | derive from the meaning of the empty input |
| `dp[0] = 0` for a counting DP | all counts zero | counting base cases are $1$ |
| wrong loop order for multisets | counts permutations instead | coins outside, sums ascending |
| ignoring unreachable states in min DP | `INF + 1` overflow, garbage answer | guard with `if (dp[s-c] < INF)` |
| `int` accumulation of residues | negative or absurd answers at large $n$ | `long long`, reduce each step |
| reading a state before it is final | silent zeros, half the paths missing | derive the order from the transitions |
| memory blow-up on $10^{3} \times 10^{3}$ with `long long` | MLE at 256 MB | two rows, or `int` with mod |
| reconstructing forward from the base | impossible path or wrong route | walk parents backwards |
| updating value and parent in different branches | reconstructed optimum differs from computed | record both at the improvement point |
| recursion depth $10^{6}$ | stack overflow | bottom-up table |

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Staircase of Steps | one-dimensional recurrence, rolling memory | §4 |
| 2 | The Café's Token Jar | coin change counting multisets | §6 |
| 3 | The Arcade's Refund | minimum coins, unreachable states | §8 |
| 4 | The Rooftop Courier | two-dimensional grid DP | §10 |
| 5 | The Dice Tally | two-dimensional DP with a small transition | §11 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Broken Staircase | $n \le 10^{6}$ | recurrence with forbidden states |
| 2 | The Toll Road | $n \le 2 \cdot 10^{5}$, $K \le 100$ | sliding-window minimum over predecessors |
| 3 | The Courier's Grid, Again | $1000 \times 1000$ | grid DP with obstacle masking |
| 4 | The Subset Ledger | $n \le 200$, $S \le 2 \cdot 10^{5}$ | boolean knapsack over a prefix |
| 5 | The Domino Wall | $n \le 1000$ | profile DP over columns |

Problem 2 is the first place where the naive transition is $O(K)$ per state and the constraints make $O(nK)$ fine - but it is also the doorway to monotonic-queue optimisation, which turns it into $O(n)$. Problem 5 is the doorway to bitmask DP, Topic 08.

---

## Next Up

Topic 05 takes the two-dimensional table and applies it to the most rehearsed DP in existence: **knapsack**. You will see why the capacity dimension exists at all, why the one-dimensional version iterates backwards, and why "at most" and "exactly" are different problems that differ by one line. If you can already see that §16.4 is knapsack with boolean values, you have the intuition the next topic formalises.

