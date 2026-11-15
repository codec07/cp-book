# Topic 05 — Dynamic Programming Part 2: Knapsack

Knapsack is the most rehearsed dynamic program in the world, and for good reason: it is the smallest problem where the DP state has two dimensions that both matter. Once you can write it from scratch - not from memory - you can write a surprising fraction of contest DPs, because most of them are knapsack with a different story.

The story is always some version of this: you have a budget, and a collection of things that cost part of the budget and are worth something. Choose a subset that fits. The interesting part is never the story; it is the four variants (0/1, unbounded, bounded, counting) that differ by a loop direction or a factor, and which are routinely mixed up under time pressure.

> **Prerequisites:** Topic 04, especially loop order and rolling arrays.
>
> **You will be able to:** write 0/1 knapsack in one and two dimensions and explain why the 1D loop runs backwards; tell the unbounded variant from the 0/1 one by the loop direction alone; count subsets with an exact sum; handle item multiplicities with binary splitting; and reconstruct which items were chosen.

---

## 1. The problem, and why greedy fails

You are given $n$ items, item $i$ having weight $w_i$ and value $v_i$, and a capacity $W$. Choose a subset of items with total weight at most $W$, maximising total value.

The obvious greedy - take the best value-per-weight first - is wrong. A famous counterexample: capacity $10$, items $(w,v) = (6,7), (5,5), (5,5)$. The ratio-greedy takes the first item (ratio $7/6$), then cannot fit anything else, total $7$; the optimum is $5 + 5 = 10$ by skipping it. Greedy makes one locally excellent choice and loses the ability to combine.

What makes DP possible is that *the capacity is a small integer*. That is the entire reason knapsack is solvable: the state can index capacity, and the number of states is $nW$, not $2^{n}$. This is called a **pseudo-polynomial** algorithm, and it is why the constraints always give you $W$ explicitly. If $W$ were $10^{18}$, knapsack would be a different (much harder) problem.

So the first thing to check in any knapsack-flavoured statement is the size of the budget. It decides whether you are writing a table or looking for structure.

---

## 2. The two-dimensional table

Let

$$dp[i][w] = \text{maximum value using a subset of items } 1..i \text{ with total weight at most } w.$$

Item $i$ is either excluded - leaving $dp[i-1][w]$ - or included, which requires $w_i \le w$ and leaves $dp[i-1][w - w_i] + v_i$. Taking the maximum:

$$dp[i][w] = \max\big(dp[i-1][w],\ dp[i-1][w - w_i] + v_i\big).$$

Base cases: $dp[0][w] = 0$ for every $w$ (no items, no value), and the answer is $dp[n][W]$.

The table is filled row by row, and each cell reads two cells from the previous row. That is the entire structure, and it is worth writing the four questions explicitly for the first few knapsacks you solve:

- **State:** prefix of items, remaining capacity.
- **Transition:** skip or take.
- **Base:** empty prefix has value $0$ at every capacity.
- **Order:** items increasing, capacity any direction (for the 2D version).

---

## 3. Solved Problem 1 — The Expedition Pack

> **Statement.** An expedition can carry at most $W$ kilograms. The quartermaster has $n$ crates, each with a known weight and a known survival value; a crate cannot be split and cannot be taken twice. The expedition leader wants the largest total value that fits in the pack. The pack holds at most $10^{5}$ kilograms and there are at most $100$ crates, each weighing at most $10^{5}$ kilograms with value up to $10^{9}$.

<details>
<summary>Solution</summary>

### Why this approach

This is 0/1 knapsack in its plainest form: capacity $W$ is small enough to index, $n$ is small, and each crate is either in or out. The value bound $10^{9}$ is a warning about overflow, not about the algorithm: with $100$ crates the total value can reach $10^{11}$, so the DP table must hold `long long`.

State space is $100 \times 10^{5} = 10^{7}$ cells; with eight bytes each that is 80 MB, which is why the one-dimensional version of §4 is the one to submit.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, W;
    cin >> n >> W;

    vector<int> weight(n), value(n);
    for (int i = 0; i < n; ++i) cin >> weight[i] >> value[i];

    vector<long long> dp(W + 1, 0);            // dp[w] = best value with capacity w

    for (int i = 0; i < n; ++i)
        for (int w = W; w >= weight[i]; --w)   // backwards: each item at most once
            dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);

    cout << dp[W] << '\n';
    return 0;
}
```

**Input**

```text
4 7
1 1
3 4
4 5
5 7
```

**Output**

```text
9
```

Take the crates of weight $3$ and $4$: value $4 + 5 = 9$ at exactly $7$ kilograms. Taking the $5$-kilogram crate instead gives $1 + 5 = 6$ or $7$, depending on what else fits, both below $9$.

**Complexity.** $O(nW)$ time, $O(W)$ memory.

### The trap

The loop direction is the entire problem, and §4 is dedicated to it. The second trap is the capacity lower bound: `w >= weight[i]` must compare with the item's weight, not with $0$, or you index `dp` negatively. The third is the value type: `int` holds up to $2.1 \times 10^{9}$, so $10^{11}$ wraps to a negative number and the `max` happily selects it.

</details>

---

## 4. Why the one-dimensional loop runs backwards

The 1D array `dp[w]` is a snapshot of the previous row. To compute the new row, each cell needs the old value at `w` and the old value at `w - w_i`. If the capacity loop runs **upward**, then `dp[w - w_i]` has already been overwritten with a value from the *current* row - which means the item has already been used once, and using it again is allowed implicitly. That is exactly the unbounded knapsack recurrence, not the 0/1 one.

Running **downward** guarantees that `dp[w - w_i]` still holds the previous row's value when it is read, so each item is considered at most once.

```cpp
// 0/1: downwards, each item once
for (int i = 0; i < n; ++i)
    for (int w = W; w >= weight[i]; --w)
        dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);

// Unbounded: upwards, each item any number of times
for (int i = 0; i < n; ++i)
    for (int w = weight[i]; w <= W; ++w)
        dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);
```

This single character - `>=` versus `<=` - is the difference between two different problems, and it appears in the practice tables of every competitive programming site. Do not memorise which is which by the direction; memorise the *reason*: upward reuses the current row, and reusing the current row means reusing the item.

---

## 5. Solved Problem 2 — The Audit Sum

> **Statement.** An auditor must determine whether some sub-collection of the company's receipts adds up to exactly a disputed total $S$. Each receipt has a positive whole-rupee amount and may be counted at most once. There are at most $200$ receipts, each amount is at most $10^{5}$, and $S$ is at most $2 \times 10^{5}$.

<details>
<summary>Solution</summary>

### Why this approach

This is knapsack with values replaced by truth: `dp[s]` is true when some subset of the receipts seen so far sums to $s$. Including a receipt of amount $a$ turns a reachable $s - a$ into a reachable $s$, so

$$dp[s] \leftarrow dp[s] \text{ or } dp[s-a].$$

Because each receipt may be used once, the sum loop runs downward for exactly the reason in §4. The state space is $2 \times 10^{5}$ booleans, and the work is $200 \times 2 \times 10^{5} = 4 \times 10^{7}$ simple operations.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, S;
    cin >> n >> S;

    vector<int> a(n);
    for (int i = 0; i < n; ++i) cin >> a[i];

    vector<char> dp(S + 1, 0);          // char, not vector<bool>: simple and fast
    dp[0] = 1;

    for (int x : a)
        for (int s = S; s >= x; --s)
            if (dp[s - x]) dp[s] = 1;

    cout << (dp[S] ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**

```text
5 11
2 3 5 8 12
```

**Output**

```text
YES
```

$3 + 8 = 11$.

**Complexity.** $O(nS)$ time, $O(S)$ memory.

### The trap

`vector<bool>` is a bitset in disguise: it stores bits, not bytes, and every read is a shift-and-mask. It works here, but the access pattern is slower than `vector<char>` by roughly a factor of two to four, and it cannot be handed to functions expecting a real array. For $4 \times 10^{7}$ accesses that constant matters.

The second trap is the early exit: once `dp[S]` is true you may print and stop, but only if you are certain no later item can *unset* it - which is true here, because reachability only grows. In a counting DP nothing like this is true, and early exits destroy the answer.

</details>

---

## 6. Unbounded knapsack: when "any number" changes the loop

If every item may be taken any number of times, the state is no longer "prefix of items" but "items considered so far, with the current one still available". The 1D recurrence becomes

$$dp[w] = \max_{i:\ w_i \le w}\big(dp[w - w_i] + v_i\big),$$

and it can be written either as "items outside, ascending capacity" or as "capacity outside, items inside". Both are correct; the first is usually faster because it touches memory sequentially.

The same shape gives **minimum coins** and **ways to make change**, which you met in Topic 04. The unification is worth stating: a knapsack item with weight $w_i$ and value $v_i$ is a coin of denomination $w_i$ with cost/value $v_i$.

One warning before the next problem: the unbounded version with a *maximum* value can loop forever if some item has $w_i = 0$ and $v_i > 0$ - the optimum is infinite. Real statements exclude zero weights, but the failure mode is worth recognising, because it is the same one that appears in shortest-path problems with negative cycles.

---

## 7. Solved Problem 3 — The Replicator's Cargo

> **Statement.** A cargo pod has capacity $W$ and can be loaded with up to $n$ kinds of canisters. Each kind has a weight and a yield, and the loading bay can manufacture as many canisters of each kind as the pod has room for - there is no stock limit. Load the pod to maximise total yield. The pod holds at most $10^{5}$ kilograms, there are at most $100$ kinds, and the best total yield fits in a signed 64-bit integer.

<details>
<summary>Solution</summary>

### Why this approach

Unlimited copies of each kind means unbounded knapsack. The 1D table is filled with the capacity loop ascending, so `dp[w - w_i]` may already include canisters of the same kind - which is exactly the reuse we want.

The safe total-yield bound means `long long` for the table; the number of copies can be large (up to $W / \min w_i$), which is another reason not to try enumerating copies.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, W;
    cin >> n >> W;

    vector<int> weight(n), value(n);
    for (int i = 0; i < n; ++i) cin >> weight[i] >> value[i];

    vector<long long> dp(W + 1, 0);

    for (int i = 0; i < n; ++i)
        for (int w = weight[i]; w <= W; ++w)          // upwards: unlimited copies
            dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);

    cout << dp[W] << '\n';
    return 0;
}
```

**Input**

```text
3 8
2 3
3 4
5 6
```

**Output**

```text
12
```

Four canisters of weight $2$ give $12$; the alternatives $2+3+3$ give $11$ and $5+3$ gives $10$. Note that the optimum uses one kind exclusively - a good reminder that "balanced" loadings are not automatically best.

**Complexity.** $O(nW)$ time, $O(W)$ memory.

### The trap

The loop direction, once more, and it is worth internalising through the *effect*: if you write this one downwards you get the 0/1 answer, which is at most as large and usually strictly smaller. On this sample it gives $11$ (the $2+3+3$ loading), which looks plausible - the exact reason such errors survive samples.

The second trap is the "at most $W$" versus "exactly $W$" distinction. This DP answers "at most", because `dp[w]` is non-decreasing in $w$ (the optimum for a smaller capacity is feasible for a larger one). For "exactly", initialise unreachable states to $-\infty$ instead of $0$, and the same loop gives the exact answer.

</details>

---

## 8. Counting knapsacks: how many subsets hit a sum

Replace `max` with addition and the optimisation problem becomes a counting problem:

$$dp[s] = \#\{\text{subsets of items seen so far with total weight } s\}.$$

The transition is `dp[s] += dp[s - a]` downward (each item used once), and `dp[0] = 1` because the empty subset is one way to reach zero. If the statement wants "at most $S$", sum `dp[0..S]` at the end; if it wants "exactly $S$", read `dp[S]`.

Under a modulus this is a counting DP; without one, the counts grow like $2^{n}$ and overflow - another reminder that the modulus is part of the algorithm, not decoration.

The same table answers "how many subsets have sum congruent to $r$ modulo $K$" by reducing indices modulo $K$; that variant appears in the unsolved set of Topic 08.

---

## 9. Solved Problem 4 — The Vault Codes

> **Statement.** A vault's code is the number of distinct sub-collections of a set of weights that total exactly $S$. The security office must publish that number modulo $10^9+7$; the weights are positive integers, each may appear once, and the set has at most $200$ weights, each at most $10^{5}$, with $S$ at most $2 \times 10^{5}$. Two sub-collections are distinct if they use different positions in the input, even when their weights are equal.

<details>
<summary>Solution</summary>

### Why this approach

Counting subsets with an exact sum is the knapsack recurrence with `+` instead of `max`, and the phrase "different positions, even when weights are equal" tells you that duplicate weights are separate items - so the standard downward loop over positions counts them correctly, with no deduplication.

`dp[s]` starts at $1$ for $s = 0$: one way to choose nothing. Every item of weight $a$ then adds, for each reachable $s$, the number of ways to reach $s - a$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1000000007LL;

int main() {
    int n, S;
    cin >> n >> S;

    vector<int> a(n);
    for (int i = 0; i < n; ++i) cin >> a[i];

    vector<long long> dp(S + 1, 0);
    dp[0] = 1;

    for (int x : a)
        for (int s = S; s >= x; --s) {
            dp[s] += dp[s - x];
            if (dp[s] >= MOD) dp[s] -= MOD;
        }

    cout << dp[S] << '\n';
    return 0;
}
```

**Input**

```text
4 5
1 2 3 4
```

**Output**

```text
2
```

The two sub-collections are $\{1,4\}$ and $\{2,3\}$.

**Complexity.** $O(nS)$ time, $O(S)$ memory.

### The trap

Count *multisets* instead of subsets and the loop runs **upward**, and on this input the answer changes: the upward version also counts $\{1,1,1,1,1\}$ and $\{1,1,3\}$, which use the weight $1$ more times than the input provides. The loop direction encodes whether an item may be reused, and the statement's phrase "each may appear once" is what fixes it.

The second trap is `dp[s] += dp[s-x]` in `int`: two residues below $10^9+7$ fit, but the sum reaches $2 \times 10^9+14$ and overflows. The conditional subtraction above is the standard cheap fix.

</details>

---

## 10. Bounded knapsack: many copies, but not unlimited

Between "one copy" and "any number" lies the real-world case: item $i$ is available $c_i$ times. The naive fix is to repeat the item $c_i$ times and run 0/1 knapsack, which costs $O(W \sum c_i)$ - hopeless when $c_i$ is $10^{9}$.

**Binary splitting** reduces any count to a logarithmic number of 0/1 items. Write $c = 1 + 2 + 4 + \dots + 2^{k-1} + r$ with $r < 2^{k}$; then any number of copies from $0$ to $c$ can be formed as a sum of a subset of $\{1, 2, 4, \dots, r\}$. So replace item $i$ by $O(\log c_i)$ pseudo-items of weights $w_i, 2w_i, 4w_i, \dots$ and values $v_i, 2v_i, 4v_i, \dots$, then run the ordinary 0/1 DP.

```cpp
// Expand a bounded item into O(log c) 0/1 items.
vector<pair<int,long long>> expand(int w, long long v, long long c) {
    vector<pair<int,long long>> items;
    for (long long k = 1; c > 0; k <<= 1) {
        long long take = min(k, c);
        items.push_back({(int)(take * w), take * v});
        c -= take;
    }
    return items;
}
```

The cost becomes $O(W \sum \log c_i)$, which for $100$ item types and $W = 10^{5}$ is about $10^{5} \times 100 \times 30$ - still large, so the usual contest bound pairs binary splitting with small $n$. When even that is too slow, the monotone-queue optimisation gives an exact $O(nW)$; it is Level 4 material, but it is worth knowing it exists, because "bounded knapsack" with $n = 1000$, $W = 10^{5}$ is not solvable by splitting.

---

## 11. Solved Problem 5 — The Depot Manifest

> **Statement.** A depot stocks $n$ kinds of crates. Kind $i$ has weight $w_i$, value $v_i$, and exactly $c_i$ crates in stock. A truck with capacity $W$ is loaded from the depot, and crates cannot be split. Maximise the value on the truck. There are at most $100$ kinds, the capacity is at most $10^{5}$, and a kind may have up to $10^{9}$ crates in stock.

<details>
<summary>Solution</summary>

### Why this approach

This is bounded knapsack. The count $c_i$ rules out repeating the item literally, and the bound $c_i \le 10^{9}$ rules out even the $O(W \sum c_i)$ version. Binary splitting turns each kind into at most $30$ pseudo-items, after which the problem is plain 0/1 knapsack - and the loop direction from §4 applies unchanged.

Total pseudo-items: at most $100 \times 30 = 3000$, times $W = 10^{5}$ gives $3 \times 10^{8}$ operations in the worst case - tight, which is why the constant matters: use `long long` for values, `int` for weights, and a flat array.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, W;
    cin >> n >> W;

    vector<int> weight;
    vector<long long> value;

    for (int i = 0; i < n; ++i) {
        int w; long long v, c;
        cin >> w >> v >> c;
        for (long long k = 1; c > 0; k <<= 1) {
            long long take = min(k, c);
            weight.push_back((int)(take * w));
            value.push_back(take * v);
            c -= take;
        }
    }

    vector<long long> dp(W + 1, 0);
    for (size_t i = 0; i < weight.size(); ++i) {
        if (weight[i] > W) continue;
        for (int w = W; w >= weight[i]; --w)
            dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);
    }

    cout << dp[W] << '\n';
    return 0;
}
```

**Input**

```text
2 7
2 3 3
3 5 2
```

**Output**

```text
11
```

Two crates of the first kind and one of the second: weight $4 + 3 = 7$, value $6 + 5 = 11$.

**Complexity.** $O(W \sum \log c_i)$ time, $O(W)$ memory.

### The trap

The expansion loop. `for (long long k = 1; c > 0; k <<= 1)` with `take = min(k, c)` produces $1, 2, 4, \dots$ and a final remainder, and the sum of the `take`s is exactly $c$. Writing the more obvious "while c >= k" version drops the remainder when $c$ is not one less than a power of two - for example $c = 6$ becomes $1 + 2$ with the remainder $3$ lost if you are careless, and the DP then cannot use all six crates.

The second trap is weight overflow in the pseudo-item: `take * w` with $c = 10^{9}$ and $w = 10^{5}$ gives $10^{14}$, which is fine in `long long` but does not fit an `int`; since the pseudo-item is useless above $W$, compare in `long long` before casting, or skip it as the code does.

</details>

---

## 12. Reconstruction: which items were taken

For "print the chosen items", the 1D array is not enough - it does not remember decisions. Two standard approaches:

**Keep the 2D table.** `dp[i][w]` as in §2, then walk backwards: if `dp[i][w] != dp[i-1][w]`, item $i$ was taken, set $w -= w_i$, and continue with $i-1$. This costs $O(nW)$ memory but the reconstruction is trivial and always correct.

**Keep a decision array.** For 1D, store `choice[i][w]` as a `bool` per item and capacity - the same memory as the 2D table, so no saving.

The reconstruction walk is short enough to memorise:

```cpp
vector<int> chosen(n, 0);
int w = W;
for (int i = n; i >= 1; --i)
    if (dp[i][w] != dp[i - 1][w]) {
        chosen[i - 1] = 1;
        w -= weight[i - 1];
    }
```

Two notes. Ties are resolved arbitrarily by this rule (it prefers not to take the item when the value is equal), which is fine unless the statement demands a specific one. And if the DP was monotone in capacity - "at most $W$" - the final `w` may be less than $W$ at the end; that is fine, but do not assert that the chosen weights sum to $W$.

---

## 13. Pseudo-polynomial complexity, and what to do when $W$ is huge

Knapsack is polynomial in $n$ and $W$, but $W$ is a *number*, not a bit-length. Written in binary, $W$ has $\log W$ bits, so an algorithm that is linear in $W$ is exponential in the input size. This is why knapsack is NP-hard in general and why the constraints matter so much: the problem is only easy when the budget is small.

Three escapes when it is not:

1. **Small $n$, huge $W$: meet in the middle.** Split the items into two halves, enumerate all $2^{n/2}$ subsets of each (feasible for $n \le 40$), sort one half's (weight, value) pairs, and for each subset of the other half binary search the best complement. $O(2^{n/2} \log 2^{n/2})$.
2. **Small total value, huge $W$: invert the table.** Let `dp[v]` be the minimum weight needed to achieve value $v$; the table is indexed by value, so its size is the sum of values, not the capacity.
3. **Special structure.** Divisible weights, few distinct weights, or weights forming a matroid all admit polynomial algorithms; recognising that is a skill for later levels.

The lesson: **always check which of $n$, $W$, and total value is small**, and index the DP by the small one. The statement's bounds tell you which table the setter intends.

---

## 14. The knapsack family at a glance

| Variant | Recurrence | Loop direction | Typical statement phrase |
|---|---|---|---|
| 0/1 maximum value | `dp[w] = max(dp[w], dp[w-w_i]+v_i)` | capacity down | "each item at most once" |
| Unbounded maximum | same | capacity up | "unlimited supply" |
| Bounded copies | split into $O(\log c)$ 0/1 items | down | "up to $c_i$ available" |
| Subset sum (feasibility) | `dp[s] = dp[s] or dp[s-a]` | down | "is there a subset summing to" |
| Count subsets, exact sum | `dp[s] += dp[s-a]` | down | "how many subsets" |
| Count multisets, exact sum | `dp[s] += dp[s-a]` | up | "how many ways to make change" |
| Minimum coins | `dp[s] = 1 + min dp[s-c]` | either | "fewest coins" |
| At most $S$ | sum `dp[0..S]` at the end | - | "total weight at most" |
| Exactly $S$ | read `dp[S]`, unreachable $-\infty$ | - | "total weight exactly" |

The two columns that decide correctness are the recurrence and the loop direction; the phrase column is how you choose them. Notice that "at most" and "exactly" are not a loop difference but a *base-case* difference - a distinction people miss because they focus on the loops.

---

## 15. Pitfalls worth a checklist

| Symptom | Cause | Fix |
|---|---|---|
| unbounded answers from a 0/1 statement | capacity loop ascending | run it downwards |
| 0/1 answers from an unbounded statement | capacity loop descending | run it upwards |
| negative table values at large inputs | `int` overflow of values up to $10^{9}$ | `long long` |
| wrong counts for repeated weights | deduplicating items that the statement treats as distinct | count by position, not by value |
| bounded knapsack TLE | repeating the item $c_i$ times | binary splitting |
| reconstruction mismatches the value | parents recorded only on strict improvement | record at the same branch as the value |
| exact-sum answer wrong at small $S$ | initialised to $0$ instead of $-\infty$ | distinguish unreachable |
| MLE at $n = 100$, $W = 10^{5}$ | 2D table of `long long` (80 MB) | one row |
| infinite loop with zero weights | item of weight $0$ and positive value | strip weight-$0$ items first |
| answer for "at most $W$" read from `dp[W]` only | unreachable states treated as zero | sum or use monotonicity |

---

## 16. Unsolved problems

Five problems. All of them are knapsacks, but only one says so; the others need you to identify the capacity, the item, and the meaning of `dp`'s value.

### 16.1 The Survey Kit

> A field survey can carry a total instrument weight of at most $W$ kilograms. There are $n$ instruments, each with a weight and a scientific value, and each may be taken or left behind. What is the largest total value the team can carry? There are at most $100$ instruments, the limit is at most $10^{5}$ kilograms, and values are up to $10^{9}$.

### 16.2 The Fair Division

> Two siblings must divide a collection of $n$ heirlooms between them. Each heirloom has a whole-number value and cannot be split. The siblings want the two shares to be as close as possible, so they need to know whether the collection can be divided into two parts of equal total value. Answer YES or NO. There are at most $200$ heirlooms, each worth at most $1000$.

### 16.3 The Stamp Album

> A collector fills an album page of total value exactly $S$ using stamps drawn from a set of denominations, with an unlimited supply of each denomination. Every stamp counts the same, and the collector wants the fewest stamps that reach $S$ exactly. If $S$ cannot be reached, report $-1$. The target is at most $10^{5}$ and there are at most $200$ denominations.

### 16.4 The Warehouse Order

> A warehouse must ship a truckload of at most $W$ kilograms. There are $n$ product lines; line $i$ has a unit weight, a unit value, and a stock of $c_i$ units. Units are indivisible, and a line may be used partially. Maximise the value shipped. Here $n \le 100$, $W \le 10^{5}$, $c_i \le 10^{9}$, and values are up to $10^{6}$.

### 16.5 The Museum Vault

> A museum stores $n$ artefacts in a vault with a weight limit $W$. Each artefact has a weight, and the curator wants to count the subsets whose total weight is at most $W$, modulo $10^9+7$. Two subsets are different if they contain different artefacts, even when weights coincide. There are at most $200$ artefacts and $W$ is at most $10^{5}$.

16.2 is the one to think about: it is subset sum in disguise, and the target is not given to you - it is half the total, which may not be an integer.

---

## Key Takeaways

1. Knapsack is solvable because the budget is a small integer; the state is (prefix of items, capacity), and the algorithm is pseudo-polynomial.
2. Greedy by value-per-weight is wrong; the counterexample is three items and a capacity of ten, and it is worth remembering.
3. The 1D loop direction *is* the multiplicity rule: downward means each item once, upward means unlimited copies.
4. `dp[0] = 1` for counting, `dp[0] = 0` for maximising, `dp[0] = -∞` for exact-sum feasibility - base cases carry meaning.
5. Bounded knapsack becomes 0/1 knapsack by binary splitting: $O(\log c)$ pseudo-items per kind.
6. "At most" versus "exactly" is a base-case difference, not a loop difference.
7. Reconstruction needs the 2D table (or a decision array); the 1D array has no memory of choices.
8. When $W$ is huge, index the table by whichever of $n$, $W$, or total value is small - or use meet in the middle for $n \le 40$.
9. Values up to $10^{9}$ with $n$ up to $100$ mean totals up to $10^{11}$: `long long` in the table, always.
10. Every knapsack variant in the table of §14 differs by one of three things: the recurrence's combining operator, the loop direction, or the base case. Learn the three, not the nine rows.

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Expedition Pack | 0/1 knapsack, one row | §3 |
| 2 | The Audit Sum | subset sum feasibility | §5 |
| 3 | The Replicator's Cargo | unbounded knapsack | §7 |
| 4 | The Vault Codes | counting subsets with an exact sum | §9 |
| 5 | The Depot Manifest | bounded items by binary splitting | §11 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Survey Kit | $n \le 100$, $W \le 10^{5}$ | 0/1 maximum value |
| 2 | The Fair Division | $n \le 200$, values $\le 1000$ | subset sum at half the total |
| 3 | The Stamp Album | $S \le 10^{5}$ | unbounded minimum coins |
| 4 | The Warehouse Order | $c_i \le 10^{9}$ | binary splitting |
| 5 | The Museum Vault | $n \le 200$, $W \le 10^{5}$ | counting with a prefix sum at the end |

Problem 5 hides its difficulty in the last step: the DP fills `dp[w]` for exact weights, and the answer is a sum over a range. Problem 2 hides it in the first step: check whether the total is even before doing anything.

---

## Next Up

Topic 06 replaces the capacity dimension with **grid coordinates** and counts paths under local constraints - the same table shape you met in Topic 04, but with obstacles, barriers, and costs, and with the first appearance of the "number of ways modulo $p$" problems that dominate Div. 2 contests. The knapsack habit that transfers is the one worth keeping: **name the state, justify the order, then write the loop.**

