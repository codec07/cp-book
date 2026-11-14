# Topic 08 — Problems on Greedy Algorithms and Graphs

This topic is the working set for the two ideas introduced in Topic 07. Ten problems are solved in full — **five greedy, five graph** — and each comes with the argument that justifies it, because in both areas the code is the easy half.

Every greedy solution here is paired with **why the obvious alternative fails**. That is deliberate. The skill this topic builds is not "recognise a greedy problem" — it is "prove the greedy before submitting", which is the difference between a 1700 and a 2100 rating.

> **Prerequisites:** [Topic 07 — Introduction to Greedy Algorithms and Graphs](Topic%2007%20-%20Introduction%20to%20Greedy%20Algorithms%20and%20Graphs.md), [Topic 05 — Introduction to Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Solve five standard greedy problems and state the exchange argument for each.
> - Solve five standard graph problems with BFS or DFS, chosen for a reason.
> - Recognise the four greedy-proof shapes: exchange, stay-ahead, earliest-deadline, density.
> - Implement multi-source BFS and path reconstruction without a reference.
> - Place 20 further problems into the right technique within a minute of reading them.

---

## 1. How the ten solutions are organised

| Part | Problems | Core technique |
|---|---|---|
| Greedy | G1–G5 | Sort by a key, then one pass (with a heap or a set) |
| Graph | R1–R5 | BFS or DFS over an explicit or implicit graph |

For every problem the same four things are written out:

1. **The greedy key or traversal invariant** — stated before any code.
2. **The proof** — an exchange argument, or a BFS/DFS correctness argument.
3. **The code** — complete, compilable, $C++17$.
4. **The trap** — the specific wrong approach that looks right.

---

## 2. The four shapes of greedy proof

Before the problems, the vocabulary. Almost every greedy proof you will write is one of these.

| Shape | Statement | Used when |
|---|---|---|
| **Exchange** | Any optimal solution can swap in the greedy choice without getting worse | Selecting items, ordering jobs |
| **Stay-ahead** | After $k$ steps, greedy's partial solution is at least as good as any other | Interval scheduling, covering |
| **Earliest deadline** | Committing to the most urgent item first is never harmful | Deadlines, scheduling |
| **Density** | Taking the best ratio first is optimal when items are divisible or independent | Knapsack, ratios |

**Exchange** is the workhorse and the one worth mastering. The others are specialisations or restatements of it.

---

## 3. Solved Problem G1 — Maximum Simultaneous Intervals

> **Statement.** Given $n$ intervals $[l_i, r_i)$, find the largest number that overlap at any single point in time.

### The idea

This is not really a scheduling problem; it is a **counting** problem. Each interval contributes $+1$ at $l_i$ and $-1$ at $r_i$. Sweep the timeline in order and the running sum is the number of active intervals.

### Why the sweep is correct

Between consecutive event times the active set does not change, so the maximum of the running sum over events is the maximum over all real positions. Nothing is missed. This is a **stay-ahead** argument: at every event, the counter equals the true number of active intervals, so its maximum equals the true maximum.

**Tie-breaking matters.** At a shared time, process $-1$ before $+1$. Otherwise an interval ending exactly where another begins is counted as overlapping, which is wrong for half-open intervals $[l, r)$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<int,int>> ev;          // (time, delta)
    for (int i = 0; i < n; i++) {
        int l, r;
        cin >> l >> r;
        ev.push_back({l, +1});
        ev.push_back({r, -1});
    }

    sort(ev.begin(), ev.end(), [](const auto &a, const auto &b) {
        if (a.first != b.first) return a.first < b.first;
        return a.second < b.second;     // -1 before +1: half-open intervals
    });

    int cur = 0, best = 0;
    for (auto &e : ev) {
        cur += e.second;
        best = max(best, cur);
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
3
1 5
2 6
4 8
```

**Output**
```text
3
```

At $t = 4$ all three are active: $1 \le 4 < 5$, $2 \le 4 < 6$ and $4 \le 4 < 8$.

### Complexity

$O(n \log n)$ for sorting $2n$ events, $O(1)$ beyond the event array.

### The trap

Sorting by start and counting "how many started before this one ended" is $O(n^2)$ and easy to get wrong on ties. The event sweep is both faster and less error-prone.

---

## 4. Solved Problem G2 — Fractional Knapsack

> **Statement.** A knapsack holds weight $W$. There are $n$ items; item $i$ has value $v_i$ and weight $w_i$. Items may be split. Maximise total value.

### The greedy key

Take items in decreasing order of **value per unit weight**, $v_i / w_i$.

### The exchange argument

Let greedy take item $A$ (highest density) and suppose an optimal solution $O$ takes a fraction of some item $B$ with $v_B/w_B < v_A/w_A$ instead. Move $\delta$ weight from $B$ to $A$. The total weight is unchanged, so feasibility is preserved. The value changes by

$$\delta \cdot \left(\frac{v_A}{w_A} - \frac{v_B}{w_B}\right) > 0.$$

So $O$ was not optimal — contradiction. Therefore an optimal solution takes as much of the highest-density item as possible, and the argument repeats.

**This proof breaks the moment items are indivisible**, because "move $\delta$ weight" is no longer available. That is exactly why 0/1 knapsack needs DP (Chapter 5) and why fractional knapsack is greedy.

### Comparing ratios without floating point

`v/w` as a `double` can tie incorrectly on nearly equal ratios. Cross-multiply:

```cpp
sort(items.begin(), items.end(), [](const auto &a, const auto &b) {
    return (long long)a.first * b.second > (long long)b.first * a.second;
});
// a.first = value, a.second = weight
```

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long W;
    cin >> n >> W;

    vector<pair<long long,long long>> it(n);     // (value, weight)
    for (auto &p : it) cin >> p.first >> p.second;

    sort(it.begin(), it.end(), [](const auto &a, const auto &b) {
        return (long long)a.first * b.second > (long long)b.first * a.second;
    });

    double total = 0.0;
    for (auto &p : it) {
        if (W == 0) break;
        long long take = min(W, p.second);
        total += (double)take * p.first / p.second;
        W -= take;
    }

    cout << fixed << setprecision(4) << total << '\n';
    return 0;
}
```

**Input**
```text
3 50
60 10
100 20
120 30
```

**Output**
```text
240.0000
```

All three fit exactly: $10 + 20 + 30 = 60 > 50$. Densities are 6, 5, 4, so take all 10 of the first (60), all 20 of the second (100), then 20 of the third (80). Total $240$.

### Complexity

$O(n \log n)$ for the sort, $O(n)$ for the pass.

### The trap

Using integer division `p.first / p.second` for the value, which truncates. Accumulate in `double`, but compare in exact integer arithmetic.

---

## 5. Solved Problem G3 — Minimising Total Waiting Time

> **Statement.** $n$ jobs must run one after another on a single machine. Job $i$ takes $t_i$ time. Minimise the **sum of completion times**.

### The greedy key

Run the **shortest job first** (SPT): sort $t_i$ ascending.

### The exchange argument

Take any schedule and two adjacent jobs with $t_i > t_j$ (the longer one first). Let $S$ be the time before them. Their contributions to the total are

$$\text{before}: (S + t_i) + (S + t_i + t_j), \qquad \text{after}: (S + t_j) + (S + t_j + t_i).$$

The difference is $t_i - t_j > 0$, so swapping to the shorter-first order strictly improves the total. Repeating this bubble-sort-like swap removes every inversion, so the sorted order is optimal.

**Note the difference from makespan.** If the objective were "finish everything as early as possible", the order would not matter at all — the last completion time is $\sum t_i$ regardless. The objective must be *sum of completion times* for SPT to be the answer. Read what is being minimised.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> t(n);
    for (auto &x : t) cin >> x;

    sort(t.begin(), t.end());

    long long clock = 0, total = 0;
    for (long long x : t) {
        clock += x;
        total += clock;          // this job's completion time
    }

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
4
8 3 1 5
```

**Output**
```text
31
```

Sorted: 1, 3, 5, 8. Completion times: 1, 4, 9, 17. Sum $= 31$.

### Complexity

$O(n \log n)$.

### The trap

Computing the *average* and forgetting to divide, or accumulating waiting time instead of completion time. Waiting time for job $i$ is the total time of jobs before it; completion time includes its own duration. The two differ by $\sum t_i$, which is a constant — so both are minimised by the same order, but the numeric answer differs.

---

## 6. Solved Problem G4 — Assigning Cookies

> **Statement.** $n$ children have greed factors $g_i$; $m$ cookies have sizes $s_j$. A child is satisfied by a cookie with size $\ge g_i$, and each cookie is used at most once. Maximise the number of satisfied children.

### The greedy key

Sort both. Walk with two pointers; give the smallest cookie that satisfies the current least-greedy child.

### The stay-ahead argument

Claim: if any assignment satisfies $k$ children, greedy satisfies at least $k$.

Consider the $k$ satisfied children in an optimal solution, in increasing order of greed, paired with cookies. Greedy processes children from least greedy upward and always uses the smallest cookie that works. By induction, after handling the $j$-th satisfied child, greedy's pointer is no further along than the optimal solution's — so whenever optimal can satisfy one more child, greedy can too.

The structure is: *being cheap now never hurts later*, because a smaller cookie spent can only leave larger cookies available.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;

    vector<int> g(n), s(m);
    for (auto &x : g) cin >> x;
    for (auto &x : s) cin >> x;

    sort(g.begin(), g.end());
    sort(s.begin(), s.end());

    int i = 0, j = 0, satisfied = 0;
    while (i < n && j < m) {
        if (s[j] >= g[i]) {     // this cookie satisfies the least greedy child
            satisfied++;
            i++;
        }
        j++;                    // cookie is used either way
    }

    cout << satisfied << '\n';
    return 0;
}
```

**Input**
```text
3 4
2 4 6
1 3 5 7
```

**Output**
```text
3
```

Cookie 3 satisfies greed 2, cookie 5 satisfies greed 4, cookie 7 satisfies greed 6. All three children are satisfied; the wasted cookie is the 1.

### Complexity

$O(n \log n + m \log m)$.

### The trap

Looping over cookies and giving each to the *most* greedy satisfiable child. That wastes large cookies on cheap children and is not optimal. Always match the least demanding requirement with the smallest sufficient resource.

---

## 7. Solved Problem G5 — Job Sequencing With Deadlines

> **Statement.** There are $n$ jobs. Job $i$ takes one unit of time, must finish by deadline $d_i$, and earns profit $p_i$ if completed. Each time slot holds one job. Maximise total profit.

### The greedy key

Process jobs in **decreasing profit**. For each job, book the **latest free slot** at or before its deadline.

### The exchange argument

Let $A$ be the highest-profit remaining job. Take any optimal schedule $O$.

- If $O$ schedules $A$, done.
- If $O$ leaves $A$ out, then $O$ either has a free slot at or before $d_A$ (insert $A$ — profit increases, contradiction with optimality), or every slot up to $d_A$ is full. In that case some job $B$ with $p_B \le p_A$ occupies a slot $\le d_A$. Replace $B$ with $A$: still feasible, profit does not decrease.

So an optimal schedule containing $A$ exists. Induction gives the greedy.

**Why the latest free slot, not the earliest?** Booking early blocks other jobs with earlier deadlines. Booking as late as possible leaves the early slots free for the jobs that need them.

### Implementation with a set of free slots

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<pair<long long,int>> job(n);      // (profit, deadline)
    int maxD = 0;
    for (auto &p : job) {
        cin >> p.first >> p.second;
        maxD = max(maxD, p.second);
    }

    sort(job.begin(), job.end(), greater<>());   // by profit, descending

    set<int> freeSlot;                            // 1 .. maxD
    for (int s = 1; s <= maxD; s++) freeSlot.insert(s);

    long long profit = 0;
    int taken = 0;

    for (auto &j : job) {
        int d = j.second;
        auto it = freeSlot.upper_bound(d);        // first slot > d
        if (it == freeSlot.begin()) continue;     // no slot at or before d
        --it;                                     // latest slot <= d
        profit += j.first;
        taken++;
        freeSlot.erase(it);
    }

    cout << taken << ' ' << profit << '\n';
    return 0;
}
```

**Input**
```text
4
100 2
19 1
27 2
25 1
```

**Output**
```text
2 127
```

Sorted by profit: $(100,2)$, $(27,2)$, $(25,1)$, $(19,1)$. Slots $\{1,2\}$. Job 100 takes slot 2, job 27 takes slot 1, and the remaining two find no free slot at or before their deadlines. Two jobs, profit $127$.

### Complexity

$O(n \log n + D)$ to build the slot set, then $O(n \log D)$.

### The trap

A greedy of "sort by deadline, take everything that fits" fails because it ignores profit — it can fill a slot with a low-value job and lose a high-value one. Profit must be the primary key; the deadline only decides *where* the job goes.

---

## 8. Solved Problem R1 — BFS With Path Reconstruction

> **Statement.** Given an unweighted graph with $n$ vertices and $m$ edges, find the shortest path (fewest edges) from $s$ to $t$, and output the vertices on it.

### The traversal invariant

BFS dequeues vertices in non-decreasing order of distance. The first time a vertex is reached, its distance is final and minimal.

**Proof.** Suppose a vertex $v$ is first reached at distance $d$ via $u$. Then $u$ was dequeued at distance $d - 1$, and at that moment every vertex at distance $< d$ had already been dequeued (the induction hypothesis). If a path of length $< d$ existed, its penultimate vertex would have been dequeued earlier and would have reached $v$, contradiction.

### Reconstructing the path

Keep a `parent[]` array, set when a vertex is first discovered. Walking parents from $t$ back to $s$ and reversing gives the path. This works only because BFS sets `parent` exactly once per vertex — the first parent is on a shortest path.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m, s, t;
    cin >> n >> m >> s >> t;
    --s; --t;

    vector<vector<int>> adj(n);
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        --u; --v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    vector<int> dist(n, -1), parent(n, -1);
    queue<int> q;
    dist[s] = 0;
    q.push(s);

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        for (int v : adj[u]) {
            if (dist[v] != -1) continue;
            dist[v] = dist[u] + 1;
            parent[v] = u;
            q.push(v);
        }
    }

    if (dist[t] == -1) { cout << "No path\n"; return 0; }

    vector<int> path;
    for (int v = t; v != -1; v = parent[v]) path.push_back(v);
    reverse(path.begin(), path.end());

    cout << dist[t] << '\n';
    for (int i = 0; i < (int)path.size(); i++)
        cout << path[i] + 1 << " \n"[i + 1 == (int)path.size()];

    return 0;
}
```

**Input**
```text
6 6 1 6
1 2
1 3
2 4
3 4
4 5
5 6
```

**Output**
```text
4
1 2 4 5 6
```

Distance 4: $1 \to 2 \to 4 \to 5 \to 6$. The alternative $1 \to 3 \to 4 \to 5 \to 6$ is equally short; BFS returns whichever parent was assigned first, which is deterministic for a fixed input order.

### Complexity

$O(n + m)$ time, $O(n + m)$ memory.

### The trap

Storing the path inside the queue (e.g. `queue<vector<int>>`) is correct but costs $O(n \cdot \text{path length})$ memory in the worst case. Store the parent array instead.

---

## 9. Solved Problem R2 — Largest Connected Region

> **Statement.** Given an $R \times C$ grid of `.` and `#`, find the size of the largest 4-connected region of `.`.

### The invariant

Every flood fill visits exactly the set of cells reachable from its seed using 4-neighbour moves, and visits each cell at most once. So the set of cells reached is precisely one connected component.

### Iterative implementation

Recursion is the textbook form but overflows the stack on large grids. Use an explicit queue.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int R, C;
    cin >> R >> C;
    vector<string> g(R);
    for (auto &row : g) cin >> row;

    const int dr[4] = {-1, 1, 0, 0};
    const int dc[4] = {0, 0, -1, 1};

    vector<vector<bool>> seen(R, vector<bool>(C, false));
    int best = 0;

    for (int r = 0; r < R; r++) {
        for (int c = 0; c < C; c++) {
            if (g[r][c] != '.' || seen[r][c]) continue;

            int size = 0;
            queue<pair<int,int>> q;
            seen[r][c] = true;
            q.push({r, c});

            while (!q.empty()) {
                auto [x, y] = q.front();
                q.pop();
                size++;

                for (int d = 0; d < 4; d++) {
                    int nx = x + dr[d], ny = y + dc[d];
                    if (nx < 0 || nx >= R || ny < 0 || ny >= C) continue;
                    if (seen[nx][ny] || g[nx][ny] != '.') continue;
                    seen[nx][ny] = true;
                    q.push({nx, ny});
                }
            }

            best = max(best, size);
        }
    }

    cout << best << '\n';
    return 0;
}
```

**Input**
```text
4 5
.#...
..##.
..#..
.....
```

**Output**
```text
16
```

There are 20 cells and 4 are `#`, so 16 dots. They all belong to one region: the `#` block is walled off from the top-right only by a single diagonal contact, and the dots wrap around it. Column 4 links the top-right group down to row 2, row 2 links rightward to column 3, and the bottom row links everything back to the left column.

### Complexity

$O(RC)$ time and $O(RC)$ memory.

### The trap

Using 8-neighbour adjacency when the statement says "sharing a side". Diagonal moves let regions leak through a diagonal gap, which changes the answer on almost every test.

---

## 10. Solved Problem R3 — Bipartiteness

> **Statement.** Decide whether an undirected graph is bipartite, i.e. whether its vertices can be split into two groups with every edge crossing between them.

### The invariant

Run BFS and colour each vertex $1$ or $2$. A vertex gets the opposite colour of the vertex that discovered it. A graph is bipartite exactly when no edge joins two same-coloured vertices.

**Proof.** If a proper 2-colouring exists, every edge crosses, so BFS's colouring — which is forced up to a global swap — agrees with it on each component. Conversely, if BFS finds a same-coloured edge $(u,v)$, the path from the component root to $u$, plus the edge, plus the path back to the root, forms an odd cycle, and odd cycles make 2-colouring impossible.

### Two structural facts worth remembering

- A graph is bipartite $\iff$ it contains **no odd cycle**.
- A tree is always bipartite (it has no cycles at all).

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;

    vector<vector<int>> adj(n);
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        --u; --v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    vector<int> colour(n, 0);          // 0 = unvisited, 1 and 2 = the two sides
    bool ok = true;

    for (int s = 0; s < n && ok; s++) {
        if (colour[s] != 0) continue;
        queue<int> q;
        colour[s] = 1;
        q.push(s);

        while (!q.empty() && ok) {
            int u = q.front();
            q.pop();
            for (int v : adj[u]) {
                if (colour[v] == 0) {
                    colour[v] = 3 - colour[u];   // the other colour
                    q.push(v);
                } else if (colour[v] == colour[u]) {
                    ok = false;
                    break;
                }
            }
        }
    }

    cout << (ok ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
4 4
1 2
2 3
3 4
4 1
```

**Output**
```text
YES
```

The 4-cycle colours $1,2,1,2$ and every edge crosses. A 3-cycle (`1 2`, `2 3`, `3 1`) would print `NO`.

### Complexity

$O(n + m)$.

### The trap

Starting BFS from only vertex $0$. A disconnected graph needs a fresh colouring per component, which is why the outer loop exists. Forgetting it makes a bipartite graph report `NO` if any later component contains an odd cycle, or miss one entirely.

---

## 11. Solved Problem R4 — Multi-Source BFS

> **Statement.** A grid contains empty cells `.` and $k$ sources `S`. For every cell, find the distance to the **nearest** source (4-neighbour moves).

### The invariant

Seeding the queue with **all** sources at distance $0$ simulates one BFS in a graph with a virtual super-source connected to every `S` by a zero-length edge. BFS still reaches each vertex by a shortest path — the shortest path from *some* source.

**Proof.** The single-source BFS correctness argument never used the fact that there is one start. All it needs is that every enqueued vertex has a correct distance. Seeding $k$ vertices with distance $0$ satisfies that.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int R, C;
    cin >> R >> C;
    vector<string> g(R);
    for (auto &row : g) cin >> row;

    const int dr[4] = {-1, 1, 0, 0};
    const int dc[4] = {0, 0, -1, 1};

    vector<vector<int>> dist(R, vector<int>(C, -1));
    queue<pair<int,int>> q;

    for (int r = 0; r < R; r++)
        for (int c = 0; c < C; c++)
            if (g[r][c] == 'S') {
                dist[r][c] = 0;
                q.push({r, c});
            }

    while (!q.empty()) {
        auto [x, y] = q.front();
        q.pop();
        for (int d = 0; d < 4; d++) {
            int nx = x + dr[d], ny = y + dc[d];
            if (nx < 0 || nx >= R || ny < 0 || ny >= C) continue;
            if (g[nx][ny] == '#' || dist[nx][ny] != -1) continue;
            dist[nx][ny] = dist[x][y] + 1;
            q.push({nx, ny});
        }
    }

    for (int r = 0; r < R; r++) {
        for (int c = 0; c < C; c++) cout << dist[r][c] << ' ';
        cout << '\n';
    }
    return 0;
}
```

**Input**
```text
3 5
S...S
.....
..#..
```

**Output**
```text
0 1 2 1 0
1 2 3 2 1
2 3 -1 3 2
```

Cell $(1,2)$ is distance 3 — its only open neighbours are the three cells around it at distance 2, and the cell below is a wall. The `#` at $(2,2)$ stays $-1$.

### Complexity

$O(RC)$ — each cell is enqueued at most once, regardless of how many sources there are.

### The trap

Running BFS once per source and taking the minimum is $O(k \cdot RC)$ and TLEs when $k$ is large. Seeding all sources at once is the whole trick.

---

## 12. Solved Problem R5 — Cycle Detection in an Undirected Graph

> **Statement.** Decide whether an undirected graph contains a cycle.

### The invariant

Run DFS from every unvisited vertex, passing the parent. If you meet a visited vertex that is **not** the parent, you have found a back edge, and a back edge closes a cycle.

**Proof.** In a DFS forest, every edge is either a tree edge (to an unvisited vertex) or connects a vertex to an ancestor — there is no third case in an undirected graph. An edge to an ancestor is a back edge, and the tree path from the ancestor down to the current vertex plus the back edge is a cycle. Conversely, every cycle contains at least one such edge.

**Why the parent check is essential.** Without it, the tree edge you just came down looks like a back edge, and every edge is reported as a cycle.

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> adj;
vector<bool> visited;

bool dfs(int u, int parent) {
    visited[u] = true;
    for (int v : adj[u]) {
        if (v == parent) continue;              // the edge we came down
        if (visited[v]) return true;            // back edge -> cycle
        if (dfs(v, u)) return true;
    }
    return false;
}

int main() {
    int m;
    cin >> n >> m;

    adj.assign(n, {});
    visited.assign(n, false);

    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        --u; --v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    for (int i = 0; i < n; i++) {
        if (visited[i]) continue;
        if (dfs(i, -1)) { cout << "YES\n"; return 0; }
    }

    cout << "NO\n";
    return 0;
}
```

**Input**
```text
5 5
1 2
2 3
3 4
4 5
5 3
```

**Output**
```text
YES
```

The edges $3\!-\!4$, $4\!-\!5$, $5\!-\!3$ form a triangle.

### Complexity

$O(n + m)$.

### The trap

Using `v == parent` on a **multigraph** — two parallel edges between the same pair are a cycle, but the parent check skips both. Either deduplicate the edges on input or track edge IDs instead of vertices.

---

## 13. A checklist for the ten patterns

| # | Problem | Key or invariant | Structure |
|---|---|---|---|
| G1 | Max simultaneous intervals | sweep, $-1$ before $+1$ | events |
| G2 | Fractional knapsack | density, cross-multiplied | sort |
| G3 | Sum of completion times | shortest first | sort |
| G4 | Assign cookies | smallest sufficient | two pointers |
| G5 | Job sequencing | profit desc, latest free slot | sort + `set` |
| R1 | Shortest path | BFS distance is final | queue + parent |
| R2 | Largest region | flood fill = component | queue |
| R3 | Bipartite | alternate colour, no same-colour edge | BFS |
| R4 | Nearest source | seed all sources at 0 | multi-source BFS |
| R5 | Cycle detection | back edge to a non-parent | DFS + parent |

---

## 14. Twenty unsolved problems

Olympiad style, with constraints. No solutions and no hints — the point is to classify each one within a minute, then prove or trace it.

### Greedy set

| # | Problem | Constraints |
|---|---|---|
| 1 | **Interval Stabbing.** Given $n$ closed intervals, choose the minimum number of points so every interval contains at least one chosen point. | $n \le 2 \times 10^5$ |
| 2 | **Boat Rescue.** Each boat carries at most two people and a total weight $\le W$. Find the minimum number of boats. | $n \le 2 \times 10^5$, $w_i \le W$ |
| 3 | **Tape Segments.** Points $x_1 < \dots < x_n$ on a line must be covered by pieces of tape of length $L$. Find the minimum number of pieces. | $n \le 2 \times 10^5$ |
| 4 | **Fuel Stops.** A car starts with fuel $F$, consumes one unit per unit distance, and there are $n$ stations with amounts $a_i$ at distances $d_i$. Find the minimum number of refuels to reach distance $D$, or report impossible. | $n \le 2 \times 10^5$ |
| 5 | **Largest Number.** Arrange $n$ integers so their concatenation (in some order) is the largest possible number. | $n \le 10^5$, $a_i \le 10^9$ |
| 6 | **Maximum Jobs Completed.** $n$ unit-time jobs with deadlines; maximise how many can be completed (profit ignored). | $n \le 2 \times 10^5$ |
| 7 | **Merge Ropes.** Merging two ropes of lengths $a$ and $b$ costs $a + b$ and produces one rope of length $a + b$. Find the minimum total cost to merge $n$ ropes into one. | $n \le 2 \times 10^5$ |
| 8 | **Gas Station Circuit.** $n$ stations in a circle; from station $i$ you gain $g_i$ fuel and spend $c_i$ to reach the next. Find a valid starting index, or report none. | $n \le 2 \times 10^5$ |
| 9 | **Water a Garden.** A garden spans $[0, L]$. Taps at positions $p_i$ with range $r_i$ cover $[p_i - r_i, p_i + r_i]$. Find the minimum number of taps to cover the whole garden. | $n \le 10^4$, $L \le 10^4$ |
| 10 | **Skill Matching.** $n$ workers with skills $a_i$ and $m$ tasks requiring $b_j$. A task is done by a worker with skill $\ge b_j$. Maximise completed tasks. | $n, m \le 2 \times 10^5$ |

### Graph set

| # | Problem | Constraints |
|---|---|---|
| 11 | **Maze Escape.** An $R \times C$ maze of `.` and `#`. Find the fewest moves from start to exit, or report impossible. | $R, C \le 1000$ |
| 12 | **Colour Regions.** Count the number of maximal 4-connected regions of identical characters. | $R, C \le 1000$ |
| 13 | **Build Two Teams.** Split a graph into two groups so no edge is inside a group. Output the groups, or report impossible. | $n, m \le 2 \times 10^5$ |
| 14 | **Forest Components.** Given a graph known to be a forest, count its connected components. | $n \le 2 \times 10^5$ |
| 15 | **Find a Cycle.** Output the vertices of any cycle in an undirected simple graph, or report that none exists. | $n, m \le 2 \times 10^5$ |
| 16 | **Rotting Oranges.** Several cells are rotten. Each minute, rot spreads to 4-neighbours. Find the time when all cells rot, or $-1$ if some never do. | $R, C \le 500$ |
| 17 | **Knight's Moves.** On an $8 \times 8$ board, find the minimum number of knight moves between two squares. | fixed $8 \times 8$ |
| 18 | **Tree Diameter.** For a tree, find the number of edges on the longest path between any two vertices. | $n \le 2 \times 10^5$ |
| 19 | **Island Perimeter.** Given a grid of `0` and `1`, find the perimeter of the single island formed by the `1`s. | $R, C \le 100$ |
| 20 | **Nearest Exit.** A grid with multiple exits; find the minimum distance from a given cell to any exit avoiding walls. | $R, C \le 1000$ |

### Classification drill

Cover the right-hand column and decide, for each problem, which of these it is:

**sort + one pass** $\cdot$ **sort + heap/set** $\cdot$ **two pointers** $\cdot$ **sweep** $\cdot$ **BFS** $\cdot$ **multi-source BFS** $\cdot$ **DFS/flood fill** $\cdot$ **not greedy — needs DP**

Three of the twenty above are listed to catch the habit of assuming greedy. Find them before coding.

---

## Key Takeaways

- The event sweep counts simultaneous intervals in $O(n \log n)$; the tie-break ($-1$ before $+1$) is part of the algorithm, not a detail.
- Fractional knapsack is greedy because items are divisible; 0/1 knapsack is not, and that difference is the whole proof.
- Shortest-job-first minimises the **sum of completion times**, and the adjacent-swap argument is two lines. It does not minimise makespan.
- Match the least demanding requirement with the smallest sufficient resource — that single sentence solves every two-pointer matching problem.
- Job sequencing keys on **profit**, not deadline; the deadline only decides which slot. Always book the **latest** free slot.
- Compare ratios by cross-multiplying in `long long`. Floating-point ties silently reorder your sort.
- BFS distance is final at the moment of discovery, which is why `parent` set once is enough to reconstruct a shortest path.
- Multi-source BFS is one normal BFS with several seeds at distance 0 — running BFS per source is the TLE trap.
- Bipartite $\iff$ no odd cycle; the BFS colouring is forced, so a same-coloured edge is a proof of failure.
- Cycle detection needs the parent check, and the parent check breaks on parallel edges.
- Iterative flood fill for anything above roughly $300 \times 300$; recursion depth equals region size.
- Greedy fails silently. Ten minutes of stress-testing against a brute force is worth more than an hour of re-reading your proof.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Sweeping with $+1$ before $-1$ on shared endpoints | Counts touching intervals as overlapping | Sort deltas ascending |
| 2 | Fractional knapsack on indivisible items | Greedy is simply wrong for 0/1 | DP, or split explicitly |
| 3 | Comparing densities as `double` | Near-equal ratios flip | Cross-multiply in `long long` |
| 4 | Using `int` for `value * weight` in the comparator | Overflow reverses the order | Cast to `long long` |
| 5 | Applying SPT when the objective is makespan | The order does not matter there | Read the objective |
| 6 | Matching the most greedy child first | Wastes large resources | Least requirement, smallest fit |
| 7 | Job sequencing keyed on deadline | Fills slots with low-profit jobs | Key on profit |
| 8 | Booking the earliest free slot | Blocks earlier deadlines | Book the latest free slot |
| 9 | `freeSlot.upper_bound(d)` without decrementing | Picks a slot after the deadline | `--it`, after checking `begin()` |
| 10 | Storing paths inside the BFS queue | $O(n \cdot L)$ memory | `parent[]` array |
| 11 | Marking visited on dequeue | Queue blow-up, duplicate writes | Mark on enqueue |
| 12 | 8-neighbour flood fill when sides are required | Regions leak diagonally | 4-neighbour |
| 13 | Recursive flood fill on a large grid | Stack overflow | Iterative queue |
| 14 | Bipartite check from vertex 0 only | Misses or misreports components | Outer loop over all vertices |
| 15 | Running BFS once per source | $O(k \cdot RC)$ | Seed all sources at once |
| 16 | Cycle detection without the parent check | Every tree edge looks like a cycle | Pass and skip the parent |
| 17 | Parent check on a multigraph | Parallel edges are a cycle, but skipped | Track edge IDs |
| 18 | `dist[]` initialised to 0 instead of $-1$ | Unreachable vertices look reachable | Use $-1$ for unseen |
| 19 | Forgetting `--u; --v;` | Silently wrong graph | Convert once, at read time |
| 20 | Assuming monotonic input in a sweep | Gaps and overlaps mis-ordered | Always sort events |

---

## Practice Problems

The twenty unsolved problems in §14 are the practice set. Work them in two passes:

1. **Classify** every problem into a technique without writing code. This should take under 20 minutes for all 20.
2. **Solve** the ten you are least sure about, then compare your approach against the checklist in §13 — if your method is not on the list, either it is a variant worth understanding or it is a wrong greedy.

| Difficulty | Problems |
|---|---|
| Easy | 3, 6, 12, 14, 19 |
| Easy–Medium | 1, 2, 5, 7, 10, 11, 13, 16 |
| Medium | 4, 8, 15, 17, 18, 20 |
| Trap (not greedy) | hidden among the twenty |

---

## Next Up

→ [Topic 09 — Rectangle Geometry and Ad Hoc Problems](Topic%2009%20-%20Rectangle%20Geometry%20and%20Ad%20Hoc%20Problems.md)

Greedy and graphs give you algorithms. The next topic gives you a **bag of tricks** for problems that have no standard algorithm at all: rectangle intersection and area, coordinate handling, orientation tests, and the ad hoc reasoning — parity, invariants, small-case enumeration — that turns a strange statement into a short program.
