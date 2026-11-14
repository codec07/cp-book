# Topic 04 — Complete Searching

Complete search — brute force — is the most reliable technique in competitive programming. You enumerate every candidate solution, test each one, and keep the best. It is never clever, it is almost never wrong, and the constraints tell you when it is the intended solution rather than the naive first attempt.

It is also the foundation of everything else: a brute force is the reference implementation you stress-test your fast solution against, and a backtracking search is the skeleton that dynamic programming later optimises.

> **Prerequisites:** [Topic 02 — Time Complexity](Topic%2002%20-%20Time%20Complexity%20(Basics%20to%20Advanced).md), [Topic 03 — Pairs, Tuples and Simulation](Topic%2003%20-%20Pairs%2C%20Tuples%20and%20Simulation.md)
> **You will be able to:**
> - Recognise the four shapes of search space and count each one.
> - Enumerate subsets with a bitmask and with recursion — and know which is faster.
> - Enumerate permutations and combinations without duplicates or omissions.
> - Write a backtracking template with correct state restoration.
> - Prune a search enough to turn a TLE into an AC.
> - Read a constraint and immediately know whether brute force is intended.

---

## 1. What complete search is

Three steps, always:

1. **Define the candidate set.** What does one candidate solution look like?
2. **Enumerate it.** Visit every candidate exactly once.
3. **Test and keep.** Check the candidate against the objective, remember the best.

```cpp
best = -INF;
for (each candidate c in candidates) {
    if (valid(c)) best = max(best, score(c));
}
```

The technique has one requirement and one cost:

- **Requirement:** the candidate set must be finite, and small enough to enumerate.
- **Cost:** the size of the candidate set, times the cost of testing one.

That is the whole analysis. If the count is $\le 10^8$ and the test is $\mathcal{O}(1)$, you pass. If not, you need a better algorithm or a way to prune.

*Contest story: a Div. 2 C with $n \le 20$ asked for the minimum cost of partitioning items into two groups. Half the field searched for a greedy argument. The constraints said $2^{20} = 10^6$ — one bitmask loop, twenty lines, accepted. The constraint was the hint and it was ignored.*

---

## 2. When brute force is the intended solution

The constraints section is an explicit signal. Learn this table by sight.

| Constraint | Candidates | Intended technique |
|---|---|---|
| $n \le 8$ | $n^n \le 1.7 \times 10^7$ | All tuples, all assignments |
| $n \le 11$ | $n! \le 4 \times 10^7$ | All permutations |
| $n \le 20$ | $2^n \cdot n \approx 2 \times 10^7$ | Bitmask with $\mathcal{O}(n)$ work per mask |
| $n \le 25$ | $2^n \approx 3.4 \times 10^7$ | Subset enumeration, $\mathcal{O}(1)$ per mask |
| $n \le 40$ | $2^{n/2} \approx 10^6$ | Meet in the middle |
| $n \le 100$ | $n^4 = 10^8$ | Four nested loops |
| $n \le 500$ | $n^3 = 1.25 \times 10^8$ | Triple loop, Floyd–Warshall |

The rule: **if the constraints are small, stop looking for the clever solution.** Small $n$ with a large time limit means the setter wants enumeration. This is the single most commonly missed signal in Div. 2 contests — people lose 40 minutes hunting for a pattern that the constraints say does not exist.

> Counter-rule, equally important: **if $n \le 10^5$, brute force is not intended.** Do not spend 20 minutes "optimising" an $\mathcal{O}(n^2)$ loop. The constraint is telling you an $\mathcal{O}(n \log n)$ idea exists.

---

## 3. The four shapes of search space

Every enumeration problem is one of these. Identify the shape, and the code follows.

| Shape | Count | Example |
|---|---|---|
| **Subsets** | $2^n$ | Which items go in the knapsack |
| **Permutations** | $n!$ | Order of tasks on one machine |
| **Combinations** | $\binom{n}{k}$ | Choose $k$ of $n$ items |
| **Tuples** | $n^k$ | All length-$k$ sequences over $n$ symbols |

And one more that appears whenever the alphabet is not the index set:

| Shape | Count | Example |
|---|---|---|
| **Strings** | $a^L$ | All length-$L$ words over an alphabet of size $a$ |

Getting the shape right determines everything: the data structure, the recursion signature, and whether you can use `next_permutation`.

---

## 4. Enumerating tuples: nested loops and the odometer

The simplest case: all $n$-tuples with each coordinate in $[0, m)$.

With a fixed dimension, nested loops are the clearest:

```cpp
for (int x = 0; x < m; x++)
    for (int y = 0; y < m; y++)
        for (int z = 0; z < m; z++)
            check(x, y, z);
```

For a variable dimension, use an odometer — an array of counters that increments like a car's mileage display:

```cpp
int k, m;                       // k coordinates, each in [0, m)
vector<int> digits(k, 0);

while (true) {
    process(digits);            // one candidate

    int pos = k - 1;            // increment the last digit
    while (pos >= 0 && ++digits[pos] == m) {
        digits[pos] = 0;        // carry
        pos--;
    }
    if (pos < 0) break;         // overflowed: all tuples visited
}
```

This is exactly how counting in base $m$ works, and it is the same idea as a recursive generation with an explicit stack. It runs $m^k$ iterations.

| Dimension | Nested loops | Odometer |
|---|---|---|
| Fixed (3) | clearer | fine |
| Variable $k$ | impossible | required |
| Very large $k$ | impossible | recursion depth $k$ |

---

## 5. Enumerating subsets with a bitmask

The canonical technique. A subset of an $n$-element set is an $n$-bit integer: bit $i$ set means "item $i$ is in".

```cpp
for (int mask = 0; mask < (1 << n); mask++) {
    long long sum = 0;
    for (int i = 0; i < n; i++)
        if (mask >> i & 1) sum += a[i];
    if (sum == target) { /* found */ }
}
```

Cost: $2^n$ masks $\times$ $n$ bit tests = $\mathcal{O}(n 2^n)$.

### The $\mathcal{O}(2^n)$ version

The inner loop can be eliminated by reusing a previously computed subset:

```cpp
vector<long long> sum(1 << n, 0);
for (int mask = 1; mask < (1 << n); mask++) {
    int low = mask & -mask;              // lowest set bit
    int i   = __builtin_ctz(mask);       // its index
    sum[mask] = sum[mask ^ low] + a[i];  // remove it, add it back
}
```

Every subset's sum is computed from a smaller subset in $\mathcal{O}(1)$. This halves or better the runtime, and the pattern generalises: *any* value that can be extended by one element can be memoised this way.

### Reading a mask

| Operation | Meaning |
|---|---|
| `mask >> i & 1` | is bit $i$ set? |
| `mask \| (1 << i)` | set bit $i$ |
| `mask & ~(1 << i)` | clear bit $i$ |
| `mask ^ (1 << i)` | toggle bit $i$ |
| `mask & -mask` | isolate the lowest set bit |
| `mask & (mask - 1)` | clear the lowest set bit |
| `__builtin_popcount(mask)` | size of the subset |
| `mask == 0` | empty subset |
| `mask == (1 << n) - 1` | full subset |

### Enumerating submasks of a mask

For each mask, iterate its subsets:

```cpp
for (int mask = 0; mask < (1 << n); mask++)
    for (int sub = mask; ; sub = (sub - 1) & mask) {
        process(sub);
        if (sub == 0) break;
    }
```

Total iterations are $3^n$ — useful up to $n \approx 17$, and the basis of subset-sum-over-subsets DP.

> **Overflow warning:** `1 << n` is `int`. For $n \ge 31$ it is undefined behaviour. If $n$ can reach 31, write `1LL << n`, and remember the loop bound itself is then $2^{31}$ — which is already too large to enumerate, so the real fix is a different algorithm.

---

## 6. Enumerating subsets with recursion

The alternative is an include/exclude decision tree, which generalises to pruning:

```cpp
int n;
vector<int> a, chosen;

void search(int pos) {
    if (pos == n) {
        process(chosen);
        return;
    }
    // branch 1: exclude a[pos]
    search(pos + 1);

    // branch 2: include a[pos]
    chosen.push_back(a[pos]);
    search(pos + 1);
    chosen.pop_back();
}
```

This produces exactly $2^n$ leaves and is structurally identical to the bitmask loop. Prefer it when:

- You need to prune (skip whole branches).
- The state is not a simple bit pattern.
- $n$ is large but most branches are infeasible.

Prefer the bitmask when the state *is* a bit pattern and there is no pruning — it is faster and has no recursion overhead.

---

## 7. Enumerating permutations

### With `next_permutation`

```cpp
vector<int> p = {1, 2, 3};
sort(p.begin(), p.end());          // MUST start sorted
do {
    process(p);
} while (next_permutation(p.begin(), p.end()));
```

Handles duplicate elements correctly: `{1,1,2}` yields 3 permutations, not 6. Cost $\mathcal{O}(n \cdot n!)$.

### With recursion

Useful when you need pruning or need to build the permutation of a multiset by availability:

```cpp
int n;
vector<int> perm;
vector<bool> used;

void search() {
    if ((int)perm.size() == n) {
        process(perm);
        return;
    }
    for (int i = 0; i < n; i++) {
        if (used[i]) continue;
        // skip duplicates if a is sorted: only take the first unused of a run
        if (i > 0 && a[i] == a[i - 1] && !used[i - 1]) continue;

        used[i] = true;
        perm.push_back(a[i]);
        search();
        perm.pop_back();
        used[i] = false;
    }
}
```

The duplicate-skip line is the subtle part: when `a` is sorted, identical values are adjacent, and requiring that the earlier copy is already used ensures each distinct permutation is generated exactly once.

| Approach | Duplicates | Pruning | Speed |
|---|---|---|---|
| `next_permutation` | automatic | no | fastest, simplest |
| Recursion with `used[]` | manual skip line | yes | slower, flexible |

---

## 8. Enumerating combinations

Choose $k$ of $n$, order irrelevant.

```cpp
int n, k;
vector<int> cur;

void gen(int start, int left) {
    if (left == 0) {
        process(cur);
        return;
    }
    for (int i = start; i <= n - left; i++) {   // bound is a pruning
        cur.push_back(i);
        gen(i + 1, left - 1);
        cur.pop_back();
    }
}
```

Two details that matter:

- **`i <= n - left`** — not `i < n`. After choosing `i`, you still need `left - 1` more indices from `i+1..n-1`. If fewer than `left - 1` remain, the branch is dead. This single bound cuts the search to exactly $\binom{n}{k}$ leaves with no wasted descent.
- **`gen(i + 1, ...)`** — passing `i + 1` rather than `start + 1` is what makes combinations unordered (each set generated once) instead of permutations.

Filtering by popcount is the bitmask equivalent:

```cpp
for (int mask = 0; mask < (1 << n); mask++)
    if (__builtin_popcount(mask) == k) process(mask);
```

That costs $2^n$ scans to find $\binom{n}{k}$ hits — wasteful when $k$ is small. Use the recursive version.

---

## 9. The backtracking template

Every recursive complete search is the same five lines with different bodies.

```cpp
void search(State &s, int pos) {
    if (isComplete(s, pos)) {          // 1. base case
        process(s);
        return;
    }
    for (Candidate c : candidates(s, pos)) {   // 2. enumerate choices
        if (!allowed(s, c)) continue;          // 3. prune
        apply(s, c);                           // 4. recurse
        search(s, pos + 1);
        undo(s, c);                            // 5. restore
    }
}
```

**Step 5 is where bugs live.** If `apply` mutates state that `allowed` reads, failing to `undo` corrupts every subsequent branch. The symptom is a wrong answer that depends on the *order* branches are explored in — extremely confusing to debug.

Three ways to get restoration right:

| Approach | When |
|---|---|
| Push/pop a vector | Adding an element to a growing set |
| Set then reset a flag | `used[i] = true; ...; used[i] = false;` |
| Pass by value | Small state; the copy *is* the undo |

Passing state by value is the safest and the slowest. For $n \le 18$ the copy is free; for larger searches, use mutable state with careful undo.

---

## 10. N-Queens: the canonical backtracking problem

Place $n$ queens on an $n \times n$ board so no two attack each other. Count the solutions.

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, solutions = 0;
vector<int> colOf;                  // colOf[row] = column of the queen on that row

bool safe(int row, int col) {
    for (int r = 0; r < row; r++) {
        if (colOf[r] == col) return false;                    // same column
        if (abs(colOf[r] - col) == abs(r - row)) return false; // same diagonal
    }
    return true;
}

void solve(int row) {
    if (row == n) { solutions++; return; }

    for (int col = 0; col < n; col++) {
        if (!safe(row, col)) continue;   // prune: kills most branches early
        colOf[row] = col;
        solve(row + 1);
        // no undo needed: colOf[row] is overwritten before it is read again
    }
}

int main() {
    n = 8;
    colOf.assign(n, -1);
    solve(0);
    cout << solutions << '\n';           // 92
    return 0;
}
```

**Why the prune is everything.** Without pruning, the search is $n^n = 16{,}777{,}216$ candidates for $n = 8$. With pruning it visits about 2,000 nodes and runs in microseconds. For $n = 12$ the unpruned search would be $12^{12} \approx 8.9 \times 10^{12}$; the pruned version finishes in well under a millisecond.

**Why no undo is needed here.** `colOf[row]` is written before the recursive call and read only by rows $< \text{row}$. When the recursion returns, the next iteration overwrites `colOf[row]` before anything reads it again. Recognising when an undo is unnecessary is as important as remembering one when it is.

| $n$ | Solutions | Nodes visited |
|---|---|---|
| 4 | 2 | 17 |
| 8 | 92 | 2,057 |
| 10 | 724 | 35,539 |
| 12 | 14,200 | 856,189 |

The node count grows far slower than $n^n$ — that is the value of a good prune. (Counted as calls to `solve`; the exact figure depends on how you instrument, but the order of magnitude does not.)

---

## 11. Worked problem: subset sum by enumeration

**Problem.** Given $n \le 25$ integers and a target $T$, decide whether some subset sums exactly to $T$.

**Shape:** subsets, $2^n$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long target;
    cin >> n >> target;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    int full = 1 << n;
    vector<long long> sum(full, 0);

    for (int mask = 1; mask < full; mask++) {
        int low = mask & -mask;
        int i = __builtin_ctz(mask);
        sum[mask] = sum[mask ^ low] + a[i];
    }

    bool found = false;
    for (int mask = 0; mask < full; mask++)
        if (sum[mask] == target) { found = true; break; }

    cout << (found ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
5 11
2 3 5 7 11
```

**Output**
```text
YES
```

**Complexity:** $\mathcal{O}(2^n)$ time and $\mathcal{O}(2^n)$ memory. At $n = 25$ that is $3.3 \times 10^7$ `long long`s — 268 MB, which exceeds a 256 MB limit. Use `int` sums when they fit, or drop the table and compute sums on the fly at $\mathcal{O}(n2^n)$.

This memory-versus-time trade is typical of bitmask DP and worth noticing early.

---

## 12. Pruning: turning TLE into AC

Pruning is the difference between "$2^{30}$ is impossible" and "$2^{30}$ with pruning is 10 ms".

Three kinds, each with a distinct justification.

### 12.1 Feasibility pruning

Abandon a branch as soon as it cannot possibly reach a valid solution.

```cpp
// Combination generation: not enough indices left
for (int i = start; i <= n - left; i++) { ... }

// N-Queens: column already occupied
if (!safe(row, col)) continue;
```

**Correctness requirement:** the prune must be sound — it may only cut branches that provably cannot succeed. If it cuts a valid branch, the answer is wrong in a way no test case will reveal reliably.

### 12.2 Optimality pruning (branch and bound)

When minimising, abandon a branch whose best possible completion is already worse than the incumbent.

```cpp
void search(int pos, long long costSoFar, long long optimisticRest) {
    if (costSoFar + optimisticRest >= best) return;   // cannot beat best
    ...
}
```

`optimisticRest` must be a **lower bound** on the remaining cost. Too weak and it prunes nothing; too strong (not actually achievable) and it prunes the optimum.

Ordering the branches matters enormously here: exploring good candidates first raises `best` early, which strengthens every subsequent prune.

### 12.3 Symmetry breaking

If two branches are provably equivalent, explore one.

```cpp
// Choosing 2 of n: only consider i < j — halves the search
for (int i = 0; i < n; i++)
    for (int j = i + 1; j < n; j++) ...

// Identical items: only take duplicates in index order
if (i > 0 && a[i] == a[i-1] && !used[i-1]) continue;
```

| Prune type | Needs | Risk |
|---|---|---|
| Feasibility | A provable impossibility test | Cutting valid branches → WA |
| Optimality | A sound lower bound | A bound that is too optimistic prunes nothing |
| Symmetry | A proof two branches are equivalent | Missing distinct solutions |

---

## 13. Generating all strings of length $L$

A search space that appears constantly: all words of length $L$ over an alphabet of size $a$, totalling $a^L$.

```cpp
string alpha = "abc";
int L = 3;
string cur(L, ' ');

void gen(int pos) {
    if (pos == L) {
        cout << cur << '\n';
        return;
    }
    for (char c : alpha) {
        cur[pos] = c;
        gen(pos + 1);
    }
}

// gen(0) prints aaa, aab, aac, aba, ... , ccc   (27 = 3^3)
```

Note there is no undo: `cur[pos]` is overwritten on the next iteration before it is read. This is the same reasoning as N-Queens.

| Alphabet size $a$ | Length $L$ | Candidates $a^L$ | Feasible? |
|---|---|---|---|
| 2 | 20 | $1.05 \times 10^6$ | yes |
| 2 | 30 | $1.07 \times 10^9$ | no |
| 26 | 4 | $4.6 \times 10^5$ | yes |
| 26 | 5 | $1.2 \times 10^7$ | marginal |
| 10 | 8 | $10^8$ | borderline |

---

## 14. The candidate-set checklist

Before writing the loop, answer these five questions. If any answer is "I'll figure it out", you will get it wrong.

1. **What is one candidate?** Write down its type. A `vector<int>`? A bitmask? A tuple?
2. **How many are there?** Compute the exact count and compare to $10^8$.
3. **Am I generating duplicates?** If yes, either deduplicate or restructure so each candidate appears once.
4. **What does "best" mean?** Minimise, maximise, count, or find any.
5. **Can I prune?** If the count is above $10^8$, pruning is not optional.

---

## 15. Complexity summary

| Search | Count | Feasible to | Notes |
|---|---|---|---|
| All subsets (bitmask) | $2^n$ | $n \le 25$ | $\mathcal{O}(1)$ per mask |
| All subsets, $\mathcal{O}(n)$ each | $n2^n$ | $n \le 20$ | Use the memoised-sum trick to remove the $n$ |
| All submasks, per mask | $3^n$ | $n \le 17$ | Subset DP |
| All permutations | $n!$ | $n \le 11$ | $11! \approx 4 \times 10^7$ |
| All combinations $\binom{n}{k}$ | $\binom{n}{k}$ | $n \le 40$ if $k$ small | Prune with the bound |
| All tuples | $n^k$ | small $n$, small $k$ | Odometer |
| All strings length $L$ | $a^L$ | $a^L \le 10^7$ | Plain recursion |
| Meet in the middle | $2^{n/2}$ | $n \le 40$ | Split, enumerate, combine |
| Backtracking with pruning | data-dependent | huge | The prune is the algorithm |

---

## Key Takeaways

- Complete search = define the candidate set, enumerate it once, test each. The analysis is only ever "how many, and how expensive is the test".
- Small constraints mean brute force is **intended**. $n \le 20$ with a 2-second limit is not a hint to be clever.
- The four shapes are subsets ($2^n$), permutations ($n!$), combinations ($\binom{n}{k}$), tuples ($n^k$). Identify the shape before writing code.
- Bitmask enumeration is $\mathcal{O}(n2^n)$; the memoised-sum trick (`sum[mask] = sum[mask ^ low] + a[i]`) makes it $\mathcal{O}(2^n)$.
- `1 << n` overflows for $n \ge 31$ — use `1LL << n`, and remember $2^{31}$ is too big to enumerate anyway.
- Recursive subset enumeration is the bitmask loop with pruning; use it only when you need to cut branches.
- `next_permutation` is the fastest and simplest permutation enumerator, and it handles duplicates automatically.
- Combinations need `i <= n - left` and `gen(i + 1, ...)`. The bound is both a correctness guard and a prune.
- The backtracking template's step 5 (undo) is where the bugs are. Pass state by value when small; otherwise restore carefully.
- N-Queens at $n = 8$ is $1.7 \times 10^7$ unpruned and 15,720 nodes pruned. A good prune is the algorithm.
- Three prune types: feasibility (impossible), optimality (cannot beat best), symmetry (equivalent). Each needs a soundness argument.
- When an undo is provably unnecessary (the slot is overwritten before being read), skipping it is correct and faster — but prove it.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `1 << n` for $n \ge 31$ | Signed overflow, UB | `1LL << n`, or a different algorithm |
| 2 | `mask >> i & 1` with precedence doubts | `>>` binds tighter than `&`, so it is correct — but unclear | Parenthesise: `((mask >> i) & 1)` |
| 3 | Forgetting `sort` before `next_permutation` | Misses all permutations before the current one | Sort first |
| 4 | `next_permutation` with `do...while` missing | The first permutation is never processed | Use `do { } while (...)` |
| 5 | Recursion without `undo` | Later branches see corrupted state; order-dependent WA | Restore after the call |
| 6 | Undoing by re-initialising the whole state | $\mathcal{O}(n)$ per node instead of $\mathcal{O}(1)$ | Undo only what you changed |
| 7 | Combination bound `i < n` instead of `i <= n - left` | Same answer, far more nodes | Use the tight bound |
| 8 | `gen(start + 1, ...)` instead of `gen(i + 1, ...)` | Generates ordered selections, i.e. duplicates | Pass `i + 1` |
| 9 | Duplicate permutations from a multiset | $6\times$ output for `{1,1,2}` | `sort` + `next_permutation`, or the `used[i-1]` skip |
| 10 | Enumerating $2^n$ with $\mathcal{O}(n)$ work when $n = 25$ | $8 \times 10^8$ operations | Memoise the sum, or drop to $n \le 20$ |
| 11 | Storing $2^{25}$ `long long`s | 268 MB, MLE | `int` sums, or on-the-fly computation |
| 12 | Pruning on a condition that is not provable | Silently wrong answers | Prove the prune is sound |
| 13 | Optimality prune with an unreachable "optimistic" bound | Prunes the optimum | The bound must be achievable-or-better |
| 14 | Recursion depth $n$ with large $n$ | Stack overflow | Iterative bitmask, or raise the stack limit |
| 15 | Testing only the sample | Brute force bugs are edge-case bugs | Compare against a second brute force on random small inputs |
| 16 | `__builtin_popcount` on a 64-bit mask | Truncates to 32 bits | `__builtin_popcountll` |
| 17 | Forgetting that `mask = 0` is the empty subset | Off-by-one in counting | Decide explicitly whether the empty set counts |
| 18 | Assuming the search is fast because it is small | $20! > 10^{18}$ | Always compute the count |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Subset sum decision, $n \le 20$ | CSES "Apple Division" family | Easy |
| 2 | Count subsets with a given property | Codeforces Div. 3 D | Easy–Medium |
| 3 | N-Queens count, $n \le 10$ | Classic / CSES "Chessboard and Queens" | Medium |
| 4 | Generate all permutations of a multiset | AtCoder ABC | Easy–Medium |
| 5 | Choose $k$ of $n$ maximising a function | Codeforces Div. 2 C | Medium |
| 6 | All strings of length $L$ matching a pattern | Codeforces Div. 2 B | Medium |
| 7 | Sudoku solver by backtracking | Classic / UVa | Medium–Hard |
| 8 | Partition into two groups of minimum difference | CSES "Apple Division" | Easy–Medium |
| 9 | TSP by permutation enumeration, $n \le 8$ | Classic | Medium |
| 10 | Bitmask DP over subsets, $n \le 18$ | CSES "Elevator Rides" style | Hard |
| 11 | Meet in the middle, $n \le 40$ | CSES "Meet in the Middle" | Hard |
| 12 | Backtracking with branch and bound | Codeforces Div. 2 D | Hard |

---

## Next Up

→ [Topic 05 — Introduction to Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)

Complete search enumerates everything. The next topic is the first big shortcut: **sorting**, and the two structures built on ordered data. Sorting turns "compare every pair" into "scan once", turns "find duplicates" into "look at neighbours", and turns "answer queries" into binary search. It is the single most common way a naive $\mathcal{O}(n^2)$ becomes an accepted $\mathcal{O}(n \log n)$.