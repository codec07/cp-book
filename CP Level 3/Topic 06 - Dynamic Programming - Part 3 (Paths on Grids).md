# Topic 06 — Dynamic Programming Part 3: Paths on Grids

A grid is the friendliest place in the world to learn DP, because the state does not have to be invented: it is the cell you are standing on. Everything else in this topic is a variation on one question - *what else does the walker need to remember besides the cell?*

Answer that question and you have the state dimension. Obstacles add nothing (a blocked cell is simply unreachable). Costs add nothing (the answer becomes a minimum instead of a count). But turns, jokers, extra travellers, and "at most $k$ of these special cells" each add one dimension, and the discipline of adding exactly one dimension - no more, no fewer - is what separates a passing DP from a table that is either too small or too large to run.

> **Prerequisites:** Topic 04 (states, order, rolling arrays), Topic 05 (the habit of reading the constraint to choose the table).
>
> **You will be able to:** count paths under obstacles; minimise and maximise path costs; build the synchronised-walk DP for two travellers at once; add a small extra dimension (turns, mines, jokers) without doubling the state space unnecessarily; and recognise when a grid problem is a graph problem in disguise.

---

## 1. Grid DP: the state is a cell

Take a walker at the top-left cell who may move only right or down, and count routes to the bottom-right cell. Define

$$dp[i][j] = \text{number of routes from } (0,0) \text{ to } (i,j).$$

Every route to $(i,j)$ arrives either from above or from the left, so

$$dp[i][j] = dp[i-1][j] + dp[i][j-1],$$

with $dp[0][0] = 1$ and out-of-grid cells treated as $0$. That is the whole algorithm. Three properties make it work, and each of them can fail in harder problems:

- **Monotone movement** gives a topological order: rows and columns both increase, so no cell depends on a later one.
- **Local transitions** mean each cell needs only its immediate neighbours, so no global information is stored.
- **Additive aggregation** means the answers combine by addition (or by `min`/`max`).

The loop order follows directly: rows outward from the top, columns outward from the left. Writing the loops in the other order reads uninitialised cells, and the result is not a crash but a table full of zeros - the most expensive kind of bug.

---

## 2. Obstacles: the state that cannot be reached

A blocked cell is not a special case in the recurrence; it is a cell whose value is forced to $0$. Write

```cpp
if (grid[i][j] == '#') { dp[i][j] = 0; continue; }
```

before the two additions, and everything else stays the same. The subtle part is the *start*: if the start is blocked, no route exists, and the `continue` must come before the `dp[0][0] = 1` assignment, or you invent a route out of a wall.

The second subtlety is the boundary. A blocked cell in the first row makes every cell to its right unreachable, and the recurrence handles that automatically - the zero propagates. This is the beauty of grid DP: constraints on individual cells compose without extra bookkeeping.

---

## 3. Solved Problem 1 — The Rooftop Revisited

> **Statement.** A courier crosses a rectangular rooftop of $r$ rows and $c$ columns, moving only one cell east or one cell south per step, starting at the north-west corner and finishing at the south-east corner. Some cells are skylights and cannot be crossed. How many routes are there, modulo $10^9+7$? The rooftop is at most $1000 \times 1000$, and the start and finish are guaranteed to be walkable.

<details>
<summary>Solution</summary>

### Why this approach

The count is a sum over predecessors with obstacles forced to zero:

$$dp[i][j] = \begin{cases} 0 & \text{skylight} \\ dp[i-1][j] + dp[i][j-1] & \text{otherwise} \end{cases}$$

with $dp[0][0] = 1$. The state count is $rc \le 10^{6}$ and each transition is two additions, so the table is both time- and memory-feasible.

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

Six monotone routes exist on an empty $3 \times 3$ grid; four of them pass through the centre, leaving two that go around it.

**Complexity.** $O(rc)$ time and memory; $O(c)$ memory with two rows.

### The trap

The order of the two conditions. If the `dp[0][0] = 1` assignment comes before the skylight check, a blocked start still gets a count, and every path from it inherits the error. Judges include "start is a wall" tests precisely because this is the first bug everybody writes.

The second trap is `int` accumulation: two residues below $10^9+7$ sum past $2 \times 10^{9}$, which overflows a signed 32-bit integer. Even a single `int` cell in the table corrupts everything downstream of it.

</details>

---

## 4. Costs instead of counts

Replace `+` with `min` (or `max`) and the counting DP becomes an optimisation DP:

$$dp[i][j] = \text{cost}(i,j) + \min\big(dp[i-1][j],\ dp[i][j-1]\big).$$

The base case changes meaning too. For counting, the empty prefix had one route; for optimisation, the start cell has cost equal to its own value, and cells in the first row or column have exactly one predecessor. The standard implementation uses a sentinel for "outside the grid":

```cpp
const long long INF = (long long)4e18;   // 4e18 fits, 9e18 does not add safely

vector<vector<long long>> dp(r, vector<long long>(c, INF));
for (int i = 0; i < r; ++i)
    for (int j = 0; j < c; ++j) {
        if (i == 0 && j == 0) { dp[i][j] = cost[i][j]; continue; }
        long long best = INF;
        if (i > 0) best = min(best, dp[i - 1][j]);
        if (j > 0) best = min(best, dp[i][j - 1]);
        dp[i][j] = best + cost[i][j];
    }
```

A sentinel of $4 \times 10^{18}$ works because adding a cost below $10^{9}$ cannot overflow; `LLONG_MAX` would overflow on the addition and produce a negative "cost" that `min` happily selects.

---

## 5. Solved Problem 2 — The Toll Tunnel

> **Statement.** A tunnel is a rectangular grid of $r$ rows and $c$ columns, and a vehicle enters at the north-west cell and leaves at the south-east cell, moving only east or south. Each cell has a toll, charged on entry, including the entry and exit cells. Find the route with the smallest total toll. Both dimensions are at most $1000$, and each toll is at most $10^{9}$.

<details>
<summary>Solution</summary>

### Why this approach

Minimum path sum on a DAG: every cell's best route comes from the better of its two predecessors, plus its own toll. There are no choices to revisit, because movement is monotone - a fact that distinguishes this from the general shortest-path problem, where a cell could be reached more cheaply by a path that came from the right.

The answer can reach $2 \times 10^{12}$ in the worst case ($2000$ cells at $10^{9}$ each), so the table must be `long long`.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int r, c;
    cin >> r >> c;

    vector<vector<long long>> cost(r, vector<long long>(c));
    for (int i = 0; i < r; ++i)
        for (int j = 0; j < c; ++j) cin >> cost[i][j];

    const long long INF = (long long)4e18;
    vector<vector<long long>> dp(r, vector<long long>(c, INF));

    for (int i = 0; i < r; ++i)
        for (int j = 0; j < c; ++j) {
            if (i == 0 && j == 0) { dp[i][j] = cost[i][j]; continue; }
            long long best = INF;
            if (i > 0) best = min(best, dp[i - 1][j]);
            if (j > 0) best = min(best, dp[i][j - 1]);
            dp[i][j] = best + cost[i][j];
        }

    cout << dp[r - 1][c - 1] << '\n';
    return 0;
}
```

**Input**

```text
3 3
1 3 1
1 5 1
4 2 1
```

**Output**

```text
7
```

The cheapest route is $1 \to 3 \to 1 \to 1 \to 1$, total $7$; the route through the centre costs $1+1+5+1+1 = 9$.

**Complexity.** $O(rc)$ time and memory.

### The trap

The sentinel, as in §4: `LLONG_MAX` plus a positive cost overflows to a negative number, and `min` then chooses the overflowed path. This failure is invisible for small grids and appears as a negative answer only at the boundary, which is exactly where judges look.

The second trap is treating the toll as charged on *leaving* rather than entering. With entry-charging, the start cell's cost is included; with exit-charging it is not. The statement's word "on entry" is the whole specification, and skipping it produces an answer that differs by exactly `cost[0][0]`.

</details>

---

## 6. Triangular grids and other domains

The rectangular recurrence is not special. Any domain whose cells can be ordered so that each depends on a proper subset of earlier cells supports the same technique. The triangle is the standard second example:

```text
      2
     3 4
    6 5 7
   4 1 8 3
```

A walker starts at the apex and descends, moving from a cell to either of the two cells below it, and wants the smallest total. Number rows from the top: cell $j$ of row $i$ has children $j$ and $j+1$ of row $i+1$. Processing from the bottom up,

$$dp[i][j] = a[i][j] + \min\big(dp[i+1][j],\ dp[i+1][j+1]\big),$$

and the answer is $dp[0][0]$. Bottom-up is the natural direction here because each cell has two children and one parent - the opposite of the rectangular case, where the fan-out goes downward.

That asymmetry is worth internalising: **choose the direction in which each cell has few dependencies**, not the direction in which the story is told.

---

## 7. Solved Problem 3 — The Pyramid Descent

> **Statement.** A board is shaped like a triangle of $n$ rows: row $i$ contains $i$ numbers. A token starts at the single number in the top row and moves downward, each move going to one of the two numbers directly below it, until it reaches the bottom row. The token collects every number it visits. Find the smallest collectable total. There are at most $1000$ rows, and each number is between $0$ and $10^{6}$.

<details>
<summary>Solution</summary>

### Why this approach

Bottom-up triangle DP: `dp[i][j]` is the minimum total from cell $(i,j)$ to the bottom. Each cell depends on the two cells below it, so filling from the last row upward guarantees readiness. The state count is $n(n+1)/2 \le 5 \times 10^{5}$ for $n = 1000$.

Working top-down would need the same table plus care with two parents, which is strictly more work for the same result.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<vector<long long>> a(n);
    for (int i = 0; i < n; ++i) {
        a[i].resize(i + 1);
        for (int j = 0; j <= i; ++j) cin >> a[i][j];
    }

    vector<vector<long long>> dp(n);
    for (int i = 0; i < n; ++i) dp[i].resize(i + 1);

    for (int j = 0; j < n; ++j) dp[n - 1][j] = a[n - 1][j];

    for (int i = n - 2; i >= 0; --i)
        for (int j = 0; j <= i; ++j)
            dp[i][j] = a[i][j] + min(dp[i + 1][j], dp[i + 1][j + 1]);

    cout << dp[0][0] << '\n';
    return 0;
}
```

**Input**

```text
4
2
3 4
6 5 7
4 1 8 3
```

**Output**

```text
11
```

The path is $2 \to 3 \to 5 \to 1$.

**Complexity.** $O(n^{2})$ time and memory, which for $n = 1000$ is half a million cells.

### The trap

The bottom row initialisation. Setting `dp[n-1][j] = a[n-1][j]` is correct; leaving it at zero silently allows a free exit from any bottom cell, and the answer becomes just the top value plus the cheapest descent to a zero. This is the single most common bug in triangle problems.

The second trap is the input shape: rows have $i+1$ entries, so a flat array indexed by `i * n + j` wastes half its space and risks reading the wrong row when $n$ varies. Either read rows as rows, as above, or store `(i, j)` as an offset into a flat vector of length $n(n+1)/2$.

</details>

---

## 8. Two travellers at once

Suppose two walkers move simultaneously from the top-left to the bottom-right, both monotone, and you want to maximise the total value collected, counting each cell once even if both visit it. The naive state is $(r_1, c_1, r_2, c_2)$: four dimensions, $10^{12}$ states for a $1000 \times 1000$ grid.

The key observation is that both walkers move at the same speed, so after $t$ steps each is on the diagonal $r + c = t$. That collapses two coordinates into one:

$$dp[t][c_1][c_2], \qquad r_1 = t - c_1,\quad r_2 = t - c_2 .$$

The state count is now $(r + c) \cdot c^{2}$, which for a $50 \times 50$ grid is about $100 \cdot 2500 = 2.5 \times 10^{5}$ - entirely feasible.

The transition tries all four combinations of "did each walker come from above or from the left", and the gain is the sum of the two current cells, minus one copy when they coincide:

```cpp
long long gain = grid[r1][c1] + (c1 != c2 ? grid[r2][c2] : 0);
```

That `c1 != c2` check is the whole difficulty: forgetting it double counts the meeting cell, and the error is largest exactly when the walkers are forced together.

---

## 9. Solved Problem 4 — The Two Harvesters

> **Statement.** Two harvesting machines start together at the north-west cell of an $r \times c$ field and both travel to the south-east cell, each moving one cell east or south per step, at the same time and with the same number of steps. Each cell holds a crop yield, and a cell's yield is collected once even if both machines pass over it. Maximise the total collected. The field is at most $50 \times 50$, and yields are between $0$ and $100$.

<details>
<summary>Solution</summary>

### Why this approach

Both machines take exactly $r + c - 2$ steps, so at step $t$ each is somewhere on the anti-diagonal $r + c = t$. Indexing each machine by its column alone therefore determines its row, reducing a four-dimensional problem to three dimensions.

State: `dp[c1][c2]` for the current step, where machine 1 is at column `c1` and machine 2 at column `c2`. Transition: each machine arrived either from the cell above (same column, previous step) or from the left (column minus one). Four combinations, take the maximum.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int r, c;
    cin >> r >> c;

    vector<vector<long long>> grid(r, vector<long long>(c));
    for (int i = 0; i < r; ++i)
        for (int j = 0; j < c; ++j) cin >> grid[i][j];

    const long long NEG = -(long long)4e18;
    vector<vector<long long>> cur(c, vector<long long>(c, NEG));
    vector<vector<long long>> nxt(c, vector<long long>(c, NEG));

    cur[0][0] = grid[0][0];

    for (int t = 1; t <= r + c - 2; ++t) {
        for (int c1 = 0; c1 < c; ++c1)
            for (int c2 = 0; c2 < c; ++c2) nxt[c1][c2] = NEG;

        for (int c1 = 0; c1 < c; ++c1) {
            int r1 = t - c1;
            if (r1 < 0 || r1 >= r) continue;
            for (int c2 = 0; c2 < c; ++c2) {
                int r2 = t - c2;
                if (r2 < 0 || r2 >= r) continue;

                long long best = NEG;
                for (int d1 = 0; d1 <= 1; ++d1)
                    for (int d2 = 0; d2 <= 1; ++d2) {
                        int p1 = c1 - d1, p2 = c2 - d2;      // previous columns
                        if (p1 < 0 || p2 < 0) continue;
                        int pr1 = (t - 1) - p1, pr2 = (t - 1) - p2;
                        if (pr1 < 0 || pr1 >= r || pr2 < 0 || pr2 >= r) continue;
                        best = max(best, cur[p1][p2]);
                    }

                if (best == NEG) continue;
                long long gain = grid[r1][c1] + (c1 == c2 ? 0 : grid[r2][c2]);
                nxt[c1][c2] = max(nxt[c1][c2], best + gain);
            }
        }
        swap(cur, nxt);
    }

    cout << cur[c - 1][c - 1] << '\n';
    return 0;
}
```

**Input**

```text
3 3
1 1 1
1 0 1
1 1 1
```

**Output**

```text
8
```

One machine goes along the top row and down the right column; the other goes down the left column and along the bottom row. Between them they cover all eight cells that contain crop, and the centre cell contributes nothing because it is empty.

**Complexity.** $O((r + c) \, c^{2})$ time, $O(c^{2})$ memory.

### The trap

The double-count rule. `gain` must add the second machine's cell only when `c1 != c2`, and it is easy to write the check as `r1 != r2` (which is equivalent but breaks when you later change the movement rules) or to forget it entirely. With the check missing, this sample returns $9$ instead of $8$ - the empty centre is not enough to expose the bug, because the extra count comes from the *start* and *end* cells, which both machines always share.

The second trap is `NEG` handling: a state that is unreachable must not contribute. Adding `gain` to `NEG` produces an enormous negative number that is still larger than `NEG`, so a careless implementation reports "reachable" for impossible states and the maximum is polluted at the edges.

</details>

---

## 10. Adding a small extra dimension

Many grid problems carry one piece of "memory" beyond the cell: how many times a special move has been used, whether a joker is still available, how many turns have been taken. The pattern is always

$$dp[i][j][s] = \text{best over routes to } (i,j) \text{ with state } s,$$

where $s$ ranges over something small - two or three values, occasionally up to ten. The cost multiplies the table by $|s|$, which is fine when $|s| \le 10$ and fatal when $|s| = 1000$.

Two rules keep this disciplined:

- **Smallest sufficient $s$.** If the statement says "at most one", use $s \in \{0, 1\}$ and *saturate*: once the budget is exhausted, stay at the boundary value. Do not model "how many remain" when "have I used it" is enough.
- **Check reachability.** With an extra dimension, many states are impossible, and impossible states must be explicitly excluded - usually by initialising them to a sentinel and skipping them, not by letting them contribute.

The transition has two flavours: a move that keeps $s$, and a move that advances $s$. Both are ordinary additions or `max`/`min`; nothing about the DP changes except the extra index.

---

## 11. Solved Problem 5 — The Minefield Pass

> **Statement.** A demining crew must cross a rectangular field from the north-west cell to the south-east cell, moving only east or south. Most cells are safe, but some contain mines, and the crew carries exactly one mine-detector charge: it can survive crossing a single mined cell, but a second one would be fatal. How many routes can the crew take? The field is at most $1000 \times 1000$, and the count is printed modulo $10^9+7$.

<details>
<summary>Solution</summary>

### Why this approach

The memory the walker needs is one bit: whether the charge has already been spent. So the state is `dp[i][j][used]`, where `used` is $0$ or $1$.

- Entering a safe cell keeps `used`.
- Entering a mined cell moves `used` from $0$ to $1$, and is forbidden when `used` is already $1$.

The answer is `dp[r-1][c-1][0] + dp[r-1][c-1][1]`.

The table has $2rc \le 2 \times 10^{6}$ cells, so the extra dimension costs a factor of two - exactly the point of using a bit rather than a count.

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

    vector<vector<array<long long, 2>>> dp(r, vector<array<long long, 2>>(c));

    for (int i = 0; i < r; ++i)
        for (int j = 0; j < c; ++j) {
            int mine = (grid[i][j] == '*');

            if (i == 0 && j == 0) { dp[i][j][mine] = 1; continue; }

            for (int used = 0; used <= 1; ++used) {
                long long ways = 0;
                if (i > 0) ways += dp[i - 1][j][used];
                if (j > 0) ways += dp[i][j - 1][used];
                ways %= MOD;

                if (mine == 0) dp[i][j][used] = ways;              // safe cell: keep the state
                else if (used == 0) dp[i][j][1] = (dp[i][j][1] + ways) % MOD;  // spend the charge
            }
        }

    cout << (dp[r - 1][c - 1][0] + dp[r - 1][c - 1][1]) % MOD << '\n';
    return 0;
}
```

**Input**

```text
3 4
....
.*..
..*.
```

**Output**

```text
6
```

There are ten monotone routes in total. Four of them cross both mines - two routes reach the first mine and two of those continue to the second - so those four are fatal, leaving six.

**Complexity.** $O(rc)$ time with a constant factor of two, $O(rc)$ memory.

### The trap

The transition for a mined cell. It is tempting to write `dp[i][j][1] += dp[i-1][j][0] + dp[i-1][j][1] + ...` - summing *both* layers into the spent layer - because "either the charge was already gone or I just spent it". That version counts routes that cross two mines: a route in layer $1$ is allowed to enter another mine and stay in layer $1$, so the constraint "at most one" quietly disappears. The mined-cell transition must take contributions from `used = 0` only, while safe cells carry both layers forward unchanged. On this sample the buggy version prints `10`, the total number of routes, because every path survives.

The second trap is the start cell: if it is mined, the charge is spent immediately, and the initial assignment `dp[0][0][mine] = 1` handles it - but only because `mine` is used as the index. Writing `dp[0][0][0] = 1` unconditionally counts routes that begin by ignoring a mine.

</details>

---

## 12. Rolling arrays and memory

A grid DP rarely needs the whole table. During row $i$, the only rows referenced are $i-1$ (fully) and $i$ (partially). Two rows suffice:

```cpp
vector<long long> prev(c, 0), cur(c, 0);
for (int i = 0; i < r; ++i) {
    for (int j = 0; j < c; ++j) {
        if (grid[i][j] == '#') { cur[j] = 0; continue; }
        if (i == 0 && j == 0) { cur[j] = 1; continue; }
        cur[j] = ((i > 0 ? prev[j] : 0) + (j > 0 ? cur[j - 1] : 0)) % MOD;
    }
    swap(prev, cur);
}
// answer: prev[c-1]
```

Three details:

- `cur[j-1]` must be the *current* row's value, so it must not be swapped or cleared mid-row.
- After the final `swap`, the answer lives in `prev`, not `cur`. Getting this backwards returns the second-to-last row's value, which is zero for $r=1$ and plausible-but-wrong otherwise.
- When rows are very wide and columns very few, transpose the grid so that the rolling dimension is the smaller one; the recurrence is symmetric under transposition.

For the two-walker DP of §8 the rolling dimension is time, and the table is $c^{2}$ rather than $2c$ - the same idea with a bigger cell.

---

## 13. When grid DP does not apply

The technique depends on three things that a statement can quietly take away:

1. **Monotone movement.** If the walker may move in all four directions, cells can depend on each other cyclically, and there is no order. That is a shortest-path problem: BFS for unit weights, Dijkstra for non-negative weights, 0-1 BFS for weights in $\{0,1\}$.
2. **No negative cycles.** Even with a topological order, "maximise total collected with revisits allowed" is unbounded unless the statement forbids it. If the graph has cycles and a positive-value cycle exists, the answer is infinite.
3. **Local transitions only.** "The route must not pass through the same column twice" is global; a cell-by-cell DP cannot enforce it without remembering the whole column history, at which point the state is $2^{c}$ and you are in profile DP or bitmask DP territory.

The giveaway is in the constraint list: grids up to $1000 \times 1000$ with four-direction movement are BFS problems with $10^{6}$ nodes; grids up to $50 \times 50$ with two walkers are the synchronised DP. The bound tells you which.

---

## 14. Complexity at a glance

| Problem | State | Transition | Total |
|---|---|---|---|
| Count paths, obstacles | $rc$ | $2$ | $O(rc)$ |
| Minimum path sum | $rc$ | $2$ | $O(rc)$ |
| Triangle descent | $n^{2}/2$ | $2$ | $O(n^{2})$ |
| Two walkers | $(r+c)c^{2}$ | $4$ | $O((r+c)c^{2})$ |
| Three walkers | $(r+c)c^{3}$ | $8$ | $O((r+c)c^{3})$ |
| At most $k$ special cells | $krc$ | $2k$ | $O(krc)$ |
| Four-direction shortest path | $rc$ nodes | edges | $O(rc \log(rc))$ Dijkstra |

The third row is why "two walkers" problems are always small: $c^{3}$ at $c = 1000$ is $10^{9}$, which is why the statement says $50$ and not $1000$. Whenever you see a surprising constraint like that, it is the setter telling you the shape of the intended state.

---

## 15. Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| zeros everywhere | loops transposed relative to dependencies | rows downward, columns rightward |
| paths through walls | skylight check after the start assignment | check first, then `continue` |
| negative minimum cost | `LLONG_MAX` sentinel overflowed | use $4 \times 10^{18}$ |
| double-counted meeting cell | two-walker gain missing `c1 != c2` | add the second cell only when distinct |
| wrong answer for $r = 1$ or $c = 1$ | answer read from the wrong buffer after `swap` | trace the single-row case by hand |
| memory limit exceeded at $1000 \times 1000$ | full `long long` table (8 MB) times layers | two rows, or `int` with modulus |
| impossible states contributing | sentinel value used in arithmetic | skip states equal to the sentinel |
| count overflows | `int` accumulation of residues | `long long`, reduce each step |
| four-direction walk loops | DP used where shortest path is needed | BFS / Dijkstra |
| joker dimension too large | modelling "how many uses remain" when a bit suffices | saturate at the boundary |

---

## 16. Unsolved problems

Five problems. The first is the plain count at a larger size; the second changes the move set; the third adds a third traveller; the fourth reverses the objective; the fifth generalises the joker.

### 16.1 The Wide Rooftop

> A courier crosses a rooftop of $r$ rows and $c$ columns, moving only east or south. Some cells are skylights and cannot be crossed. Count the monotone routes from the north-west corner to the south-east corner, modulo $10^9+7$. Both dimensions are at most $2000$, and the start and finish are walkable.

### 16.2 The Mine Cart

> A cart travels through a rectangular mine from the north-west cell to the south-east cell. It may move east, south, or diagonally south-east, one cell per step. Every cell has a cost, charged on entry, including the entry and exit cells. Find the cheapest route. Both dimensions are at most $1000$, and costs are up to $10^{9}$.

### 16.3 The Three Couriers

> Three couriers start together at the north-west cell of a field and travel to the south-east cell, each moving one cell east or south per step, all at the same time. Each cell holds a value, collected once no matter how many couriers pass through it. Maximise the total collected. The field is at most $30 \times 30$, and values are between $0$ and $10^{6}$.

### 16.4 The Pyramid Ascent

> A board is shaped like a triangle of $n$ rows, row $i$ containing $i$ numbers. A token starts at the single number in the top row, moves downward to one of the two adjacent numbers below, and continues to the bottom row, collecting every number it visits. Find the largest collectable total. There are at most $1000$ rows, and numbers are between $0$ and $10^{6}$.

### 16.5 The Joker Path

> A courier crosses an $r \times c$ grid from the north-west cell to the south-east cell, moving only east or south. Some cells are traps. The courier carries $k$ joker tokens; entering a trapped cell consumes one token, and entering a trapped cell with no tokens left is impossible. Count the valid routes modulo $10^9+7$. Here $r, c \le 100$ and $k \le 5$.

Problem 16.3 is the one to budget time for: the state is three columns, the transition is eight combinations, and the double-count rule from §9 generalises in a way that is easy to get wrong for the pairwise cases.

---

## Key Takeaways

1. Grid DP works because monotone movement supplies a topological order, transitions are local, and answers combine additively or by `min`/`max`.
2. An obstacle is a cell with value $0$ (or the sentinel), set *before* any other assignment at that cell.
3. For optimisation, the start cell is initialised to its own cost and out-of-grid predecessors to a large sentinel that cannot overflow when a cost is added.
4. When several walkers move in lockstep, collapse their positions onto the anti-diagonal: $(r_1,c_1,r_2,c_2)$ becomes $(t, c_1, c_2)$.
5. Two walkers cost $O((r+c)c^{2})$; three cost $O((r+c)c^{3})$. That is why such statements have small grids.
6. Extra memory of one bit (a joker spent, a turn taken) costs a factor of $|s|$, not an exponential - unless the memory is a set, in which case it is bitmask DP.
7. Choose the direction of the DP by dependency count: triangles are filled bottom-up, rectangles top-down.
8. Rolling arrays need care at the `swap` boundary; the answer lives in `prev` after the last swap.
9. Four-direction movement destroys the topological order; that is BFS or Dijkstra, not DP.
10. A surprising constraint (grids of size $50$ instead of $1000$) is the setter naming your intended state dimension.

---

## Practice Problems

**Solved in this topic**

| # | Problem | What it really asks | Section |
|---|---|---|---|
| 1 | The Rooftop Revisited | counting with obstacles | §3 |
| 2 | The Toll Tunnel | minimum path sum | §5 |
| 3 | The Pyramid Descent | bottom-up triangle DP | §7 |
| 4 | The Two Harvesters | synchronised two-walker DP | §9 |
| 5 | The Minefield Pass | one-bit extra state | §11 |

**Unsolved**

| # | Problem | Bound | Intended shape |
|---|---|---|---|
| 1 | The Wide Rooftop | $2000 \times 2000$ | count with obstacles, rolled rows |
| 2 | The Mine Cart | $1000 \times 1000$ | min path with a diagonal move |
| 3 | The Three Couriers | $30 \times 30$ | three-walker synchronisation |
| 4 | The Pyramid Ascent | $n \le 1000$ | max instead of min, bottom-up |
| 5 | The Joker Path | $100 \times 100$, $k \le 5$ | budgeted special cells |

---

## Next Up

Topic 07 leaves the grid behind for the most-quoted DP in competitive programming: the **longest increasing subsequence**. It is worth its own topic because the $O(n^{2})$ DP is the easy half and the $O(n \log n)$ patience-sorting version is the half that contests actually test - and because LIS is the entry point to a whole family of "best subsequence under a partial order" problems that the rest of this chapter builds on.

