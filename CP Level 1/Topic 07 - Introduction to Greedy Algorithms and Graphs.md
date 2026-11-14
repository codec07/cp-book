# Topic 07 — Introduction to Greedy Algorithms and Graphs

Two ideas share this topic because they are the two things a Level 1 contestant reaches for when complete search is too slow and casework has no structure to exploit.

**Greedy** says: make the locally best choice and never look back. It is the fastest thing in competitive programming and the easiest to get wrong, because "locally best" is a claim that needs proof, not an instinct.

**Graphs** are the language in which most real problems are actually stated. Once you can see the graph hiding inside a grid, a set of intervals, or a network of friendships, a large family of problems collapses into two procedures: breadth-first search and depth-first search.

> **Prerequisites:** [Topic 05 — Introduction to Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md), [Topic 04 — Complete Searching](Topic%2004%20-%20Complete%20Searching.md)
> **You will be able to:**
> - State the exchange argument and use it to justify a greedy choice.
> - Tell, before coding, whether a greedy approach is provable or merely plausible.
> - Construct the counterexample that kills a wrong greedy.
> - Store a graph as an adjacency list and read it in either standard input format.
> - Run BFS and DFS from memory, and know which one answers which question.
> - Count connected components and flood-fill a grid.

---

## 1. What greedy means

A greedy algorithm builds a solution one decision at a time, and at each step takes the option that looks best *right now*, according to a fixed rule. It never reconsiders, never backtracks, and never keeps alternatives.

```cpp
sort(items.begin(), items.end(), bestFirst);   // 1. order by the greedy key
for (auto &x : items) {
    if (fits(x)) take(x);                      // 2. take greedily
    if (done()) break;
}
```

That is the entire shape. Nearly every greedy solution in competitive programming is some version of **sort, then one pass**.

The hard part is never the loop. It is answering:

> **Does taking the locally best option now ever prevent me from reaching a better global solution?**

If the answer is "no", the greedy is correct and you are done in $O(n \log n)$. If the answer is "sometimes", the greedy is wrong, and a single counterexample destroys it.

---

## 2. The exchange argument

The standard proof technique. You prove a greedy choice is safe by showing that any optimal solution can be transformed into one containing your choice, without becoming worse.

**Template:**

1. Let $G$ be the first choice the greedy makes.
2. Take any optimal solution $O$.
3. If $O$ already contains $G$, done.
4. Otherwise, $O$ uses some other choice $C$ where greedy used $G$. Exchange $C$ for $G$.
5. Show the exchange does not violate feasibility.
6. Show the exchange does not decrease the objective.
7. Conclude that an optimal solution containing $G$ exists, so greedy was safe.
8. Repeat inductively for the remaining choices.

**Worked instance — activity selection.** We choose the maximum number of non-overlapping intervals.

- Greedy rule: sort by end time, take each interval that starts at or after the last chosen end.
- Claim: the first interval chosen, $G$ = the one with the earliest end, belongs to some optimal solution.
- Let $O$ be optimal, and let $O_1$ be its earliest-ending interval. Since $G$ has the earliest end overall, $\text{end}(G) \le \text{end}(O_1)$.
- Replace $O_1$ with $G$. Every later interval in $O$ starts after $\text{end}(O_1) \ge \text{end}(G)$, so it still starts after $\text{end}(G)$ — feasibility is preserved.
- The number of intervals is unchanged. So $O' = O - \{O_1\} + \{G\}$ is optimal and contains $G$.
- By induction on the remaining intervals, greedy is optimal.

Note the shape: the proof works because *finishing earlier can only help*. The greedy key (end time) is exactly the property that makes the exchange safe. This is why "sort by start time" is not just a different heuristic — it fails step 4, because a long early interval blocks many later ones.

### Where the exchange argument fails

If you cannot complete step 6 — that is, if you cannot show the swap never hurts — that is evidence the greedy is wrong, not that the proof is hard. Go hunting for a counterexample.

---

## 3. Greedy is a claim, not a technique

Beginners often treat greedy as "try the obvious thing and hope". That is why greedy problems have the highest rate of wrong submissions in Div. 2 contests.

Before writing the loop, answer all four:

| Question | If the answer is no |
|---|---|
| What exactly is my greedy key? | You have no algorithm yet |
| Why is taking the best now never harmful? | You have a heuristic, not a solution |
| Can I prove it by exchange? | Look for a counterexample |
| Is the answer even a "take items in order" problem? | Greedy probably does not apply |

**The discipline:** write the proof on paper before the code. Two sentences of exchange argument is enough. If the two sentences cannot be written, do not submit.

---

## 4. A greedy that fails: coin change

The canonical warning. Coins $\{1, 3, 4\}$, make amount 6, minimise the number of coins.

```cpp
// Greedy: always take the largest coin that fits
int coins[] = {4, 3, 1};
int amount = 6, used = 0;
for (int c : coins)
    while (amount >= c) { amount -= c; used++; }

// used == 3   (4 + 1 + 1)
```

Greedy gives $4 + 1 + 1 = 3$ coins. The optimum is $3 + 3 = 2$ coins.

The greedy rule — "largest coin first" — is extremely natural and completely wrong. The failure is exactly at step 6 of the exchange argument: swapping the chosen `4` for a `3` leaves $3$ remaining, which is reachable; there is no exchange that preserves both count and feasibility.

| Coin system | Greedy correct? |
|---|---|
| $\{1, 5, 10, 25\}$ (US) | yes — provable, but only for this system |
| $\{1, 2, 5, 10, 20, 50, 100\}$ (Euro) | yes |
| $\{1, 3, 4\}$ | **no** |
| Arbitrary | **no** — needs DP |

**Lesson:** a greedy that works for the currency you happen to know is not a greedy that works. The correct general solution is dynamic programming (Chapter 5), and the correct instinct on seeing "minimum coins" is *suspect greedy, then test it*.

---

## 5. Recognising a greedy problem

Greedy tends to apply when one of these structural features is present.

| Signal | Typical greedy key |
|---|---|
| You must select a maximum/minimum number of items under one resource | Sort by the binding constraint (end time, deadline, weight) |
| Items have a ratio that determines usefulness | Sort by density, descending |
| Two sequences must be matched in order | Sort both, then pair by rank |
| A deadline constrains when work can happen | Sort by deadline ascending |
| You process a line and only the running summary matters | Keep a running max/min/count, no sort at all |
| A choice is reversible or locally dominant | Take it immediately |

And greedy tends *not* to apply when:

- Choices interact in a way that changes earlier decisions (needs DP).
- The problem counts solutions rather than finding one (needs DP or combinatorics).
- The constraint is two-dimensional with no single ordering (needs matching or flow).
- You catch yourself writing "then we might need to undo".

---

## 6. Greedy versus brute force versus DP

| Criterion | Complete search | Greedy | DP |
|---|---|---|---|
| Constraint size | tiny ($n \le 20$) | large ($n \le 2 \times 10^5$) | medium (state space) |
| Correctness burden | none — it enumerates | a proof | a recurrence |
| Typical time | $2^n$ or $n!$ | $O(n \log n)$ | $O(n \cdot \text{states})$ |
| Fails how | TLE | **silently wrong answers** | wrong state definition |
| Order of preference | reference implementation | first thing to try | when greedy dies |

The dangerous row is greedy's. A brute force that TLEs gives you a loud signal; a wrong greedy gives you a green "Wrong answer on test 3" and no idea why.

**Practical algorithm for a new problem:** write the brute force, then look for a greedy key, then prove it by exchange, then stress-test greedy against brute force on random small inputs. If they disagree, the brute force is right.

---

## 7. What a graph is

A graph is a set of **vertices** (nodes) and a set of **edges** (connections).

| Term | Meaning |
|---|---|
| **Vertex / node** | An object: a city, a person, a cell, a state |
| **Edge** | A connection between two vertices |
| **Directed edge** | Has an orientation: $u \to v$ only |
| **Undirected edge** | Symmetric: $u \leftrightarrow v$ |
| **Weighted edge** | Has a number attached: distance, cost, time |
| **Degree** | Number of edges incident to a vertex |
| **Path** | A sequence of vertices connected by edges |
| **Cycle** | A path that returns to its start |
| **Connected** | Every vertex is reachable from every other |
| **Tree** | Connected and acyclic — exactly $n - 1$ edges |
| **DAG** | Directed and acyclic |

Two numbers describe the size of a graph, and every complexity statement uses them:

- $n$ = number of vertices
- $m$ = number of edges

For a simple undirected graph, $0 \le m \le \frac{n(n-1)}{2}$. That upper bound is why $O(n^2)$ graph algorithms are only usable for small $n$ — and why adjacency lists matter.

---

## 8. Three ways to store a graph

### 8.1 Adjacency matrix

`adj[u][v] = 1` if there is an edge.

```cpp
vector<vector<int>> adj(n, vector<int>(n, 0));
adj[u][v] = adj[v][u] = 1;          // undirected
```

- **Memory:** $O(n^2)$.
- **Edge query:** $O(1)$.
- **Iterate neighbours:** $O(n)$, even if the vertex has degree 1.

Use it when $n \le 2000$ and you need constant-time "is there an edge?" queries — Floyd–Warshall, dense graphs, or counting common neighbours.

### 8.2 Adjacency list

`adj[u]` is a vector of the neighbours of `u`.

```cpp
vector<vector<int>> adj(n);
adj[u].push_back(v);
adj[v].push_back(u);                // omit for directed
```

- **Memory:** $O(n + m)$.
- **Edge query:** $O(\deg(u))$.
- **Iterate neighbours:** $O(\deg(u))$ — total work $O(n + m)$.

This is the default. Almost every problem should use it. If $n$ and $m$ can reach $10^5$ or $2 \times 10^5$, it is the only option that fits.

### 8.3 Edge list

A vector of `(u, v)` or `(u, v, w)`.

```cpp
vector<tuple<int,int,int>> edges;   // (u, v, weight)
```

- **Memory:** $O(m)$.
- **Needed by:** Kruskal, Bellman–Ford, and algorithms that reason about edges globally.

Use it when the problem is about the *set* of edges, not about any one vertex's neighbourhood.

### 8.4 The comparison

| Operation | Matrix | Adjacency list | Edge list |
|---|---|---|---|
| Memory | $O(n^2)$ | $O(n+m)$ | $O(m)$ |
| "Is there an edge $u$–$v$?" | $O(1)$ | $O(\deg u)$ | $O(m)$ |
| All neighbours of $u$ | $O(n)$ | $O(\deg u)$ | $O(m)$ |
| Add an edge | $O(1)$ | $O(1)$ | $O(1)$ |
| Full traversal | $O(n^2)$ | $O(n+m)$ | — |
| Practical bound | $n \le 2000$ | $n, m \le 2 \times 10^5$ | any |

> **Memory trap:** `vector<vector<int>> adj(n)` for $n = 10^6$ allocates a million small vectors. That is roughly 24 MB of vector headers alone, before a single edge. For very large $n$ with few edges, use a flat array-based adjacency list (CSR) or `vector<pair<int,int>>` sorted by source.

---

## 9. Reading a graph from input

Two formats appear constantly. Recognise them by the first line.

**Format A — one-based vertices, edge count given.**

```text
n m
u1 v1
u2 v2
...
um vm
```

```cpp
int n, m;
cin >> n >> m;
vector<vector<int>> adj(n);
for (int i = 0; i < m; i++) {
    int u, v;
    cin >> u >> v;
    --u; --v;                        // convert to 0-based once, here
    adj[u].push_back(v);
    adj[v].push_back(u);
}
```

**Format B — one-based vertices, $n-1$ edges implied (a tree).**

```text
n
u1 v1
...
u_{n-1} v_{n-1}
```

```cpp
int n;
cin >> n;
vector<vector<int>> adj(n);
for (int i = 0; i < n - 1; i++) {
    int u, v;
    cin >> u >> v;
    --u; --v;
    adj[u].push_back(v);
    adj[v].push_back(u);
}
```

**Always decrement once, at the point of reading.** Mixing 0-based and 1-based vertices is the most common graph bug: it does not crash, it silently builds the wrong graph, and the sample test often still passes.

---

## 10. Breadth-first search

BFS explores a graph in rings by distance from a source. It answers, for any unweighted graph:

- the shortest number of edges from the source to every vertex,
- whether each vertex is reachable,
- and, if you carry a parent array, the actual path.

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

    vector<int> dist(n, -1), parent(n, -1);
    queue<int> q;

    dist[0] = 0;
    q.push(0);

    while (!q.empty()) {
        int u = q.front();
        q.pop();

        for (int v : adj[u]) {
            if (dist[v] != -1) continue;   // already reached
            dist[v] = dist[u] + 1;
            parent[v] = u;
            q.push(v);
        }
    }

    for (int i = 0; i < n; i++)
        cout << "dist(" << i + 1 << ") = " << dist[i] << '\n';

    return 0;
}
```

**Input**
```text
6 6
1 2
1 3
2 4
3 4
4 5
5 6
```

**Output**
```text
dist(1) = 0
dist(2) = 1
dist(3) = 1
dist(4) = 2
dist(5) = 3
dist(6) = 4
```

### Why BFS gives shortest paths

BFS processes vertices in non-decreasing order of distance. Every vertex at distance $d$ is enqueued only after all vertices at distance $d - 1$, because a vertex is enqueued the moment its first neighbour pops off the queue. So the first time BFS reaches a vertex, it has used the fewest possible edges.

This argument depends on **every edge costing the same**. Put weights on the edges and BFS is no longer correct — that is Dijkstra's job, in Chapter 4.

### The visited-check placement

Notice the check is *before* the push, not after the pop:

```cpp
if (dist[v] != -1) continue;   // correct: mark on enqueue
```

If you instead mark when popping, a vertex can be pushed many times and the queue can grow to $O(m)$, and worse, distances may be assigned more than once. **Mark at enqueue.**

| | Mark on enqueue | Mark on dequeue |
|---|---|---|
| Vertex enqueued | at most once | possibly many times |
| Queue size | $O(n)$ | $O(m)$ |
| Correct distances | yes | yes, but wasteful |
| Risk | none | duplicate processing, slower |

---

## 11. Depth-first search

DFS follows one edge as far as it can before backing up. It answers:

- reachability (the same set of vertices BFS reaches),
- connected components,
- cycle detection,
- topological order,
- and the structure of trees (subtrees, entry/exit times).

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> adj;
vector<bool> visited;

void dfs(int u) {
    visited[u] = true;
    for (int v : adj[u]) {
        if (!visited[v]) dfs(v);
    }
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

    int components = 0;
    for (int i = 0; i < n; i++) {
        if (visited[i]) continue;
        components++;
        dfs(i);
    }

    cout << components << '\n';
    return 0;
}
```

**Input**
```text
7 4
1 2
2 3
4 5
6 7
```

**Output**
```text
3
```

The components are $\{1,2,3\}$, $\{4,5\}$ and $\{6,7\}$ — three maximal sets of mutually reachable vertices.

### BFS or DFS?

| Question | Use |
|---|---|
| Shortest path in an unweighted graph | **BFS** |
| Is $v$ reachable from $u$? | either |
| Count connected components | either |
| Cycle detection in an undirected graph | DFS |
| Topological sort | DFS (or Kahn's BFS) |
| Level-by-level processing | BFS |
| Subtree aggregates | DFS |
| Memory on a wide graph | BFS is $O(n)$ frontier; DFS is $O(\text{depth})$ stack |

Both take $O(n + m)$ with an adjacency list. Choose by what you need to compute, not by speed.

---

## 12. Grids are graphs

A grid is the most common disguised graph. Each cell is a vertex; two cells are adjacent if they share a side (4-neighbour) or a corner (8-neighbour).

```cpp
const int dr[4] = {-1, 1, 0, 0};
const int dc[4] = {0, 0, -1, 1};

bool inside(int r, int c, int n, int m) {
    return r >= 0 && r < n && c >= 0 && c < m;
}
```

**Counting islands** — connected regions of `.`:

```cpp
int n, m;
vector<string> grid;
vector<vector<bool>> seen;

void flood(int r, int c) {
    seen[r][c] = true;
    for (int d = 0; d < 4; d++) {
        int nr = r + dr[d], nc = c + dc[d];
        if (!inside(nr, nc, n, m)) continue;
        if (seen[nr][nc]) continue;
        if (grid[nr][nc] != '.') continue;
        flood(nr, nc);
    }
}
```

Then:

```cpp
int islands = 0;
for (int r = 0; r < n; r++)
    for (int c = 0; c < m; c++)
        if (grid[r][c] == '.' && !seen[r][c]) { islands++; flood(r, c); }
```

*Why recursion is dangerous here:* a $1000 \times 1000$ open grid gives a DFS depth of up to $10^6$, which overflows an 8 MB stack. For grids above roughly $300 \times 300$, write the flood fill with an explicit `queue` or `stack`. This is the single most common runtime error in grid problems.

| Grid size | Recursive DFS | Iterative BFS/DFS |
|---|---|---|
| $\le 200 \times 200$ | safe | safe |
| $500 \times 500$ | risky | safe |
| $1000 \times 1000$ | **stack overflow** | safe |

---

## 13. Greedy and graphs together

The two ideas meet constantly:

| Problem shape | The greedy part | The graph part |
|---|---|---|
| Minimum spanning tree | Always take the cheapest safe edge | Connectivity |
| Dijkstra | Always expand the closest unfinalised vertex | Weighted graph |
| Topological sort | Always take a vertex with in-degree 0 | DAG |
| Minimum rooms for intervals | Take the earliest-ending free room | Interval structure |

All four are covered later, but the pattern is worth naming now: **greedy chooses the order, the graph supplies the constraints.** When you meet a problem that feels like greedy but the choice depends on what is already connected, it is a greedy-on-a-graph problem.

---

## 14. Complexity summary

| Task | Time | Space |
|---|---|---|
| Build adjacency list | $O(n + m)$ | $O(n + m)$ |
| Build adjacency matrix | $O(n^2)$ | $O(n^2)$ |
| BFS from one source | $O(n + m)$ | $O(n)$ |
| DFS from one source | $O(n + m)$ | $O(n)$ |
| All connected components | $O(n + m)$ | $O(n)$ |
| Grid flood fill ($R \times C$) | $O(RC)$ | $O(RC)$ |
| Greedy with a sort | $O(n \log n)$ | $O(n)$ or $O(1)$ |
| Greedy with a heap | $O(n \log n)$ | $O(n)$ |

Everything graph-shaped at Level 1 is linear in the size of the graph. That is precisely why $m$ can be $2 \times 10^5$ and still pass — as long as you use adjacency lists.

---

## Key Takeaways

- A greedy algorithm is **sort, then one pass**. The loop is trivial; the proof is the solution.
- Use the exchange argument: take the greedy choice $G$, show any optimal solution can be modified to contain $G$ without getting worse, then induct.
- If exchange step 6 fails, that is a signal the greedy is wrong — go find the counterexample.
- Coin change with $\{1,3,4\}$ is the canonical wrong greedy: $4+1+1 = 3$ versus $3+3 = 2$.
- Greedy fails **silently**. Always stress-test it against a brute force on random small inputs.
- Graphs have two size parameters, $n$ and $m$, and every complexity statement uses both.
- **Adjacency list is the default** at $O(n+m)$ memory. A matrix is only for dense graphs with $n \le 2000$.
- Decrement vertex indices exactly once, at read time. Mixed indexing is the most common graph bug.
- BFS gives shortest paths only when every edge costs the same. Mark visited **on enqueue**.
- DFS gives reachability, components, cycles and subtree structure. It gives no distances.
- Both traversals are $O(n + m)$; choose by the question, not the speed.
- A grid is a graph with 4 (or 8) neighbours per vertex. Use an explicit stack or queue when $R \times C$ exceeds about $10^5$.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Greedy without a proof | Silently wrong on hidden tests | Exchange argument on paper |
| 2 | Sorting by start time for interval scheduling | A long early interval blocks everything | Sort by end time |
| 3 | Assuming coin greedy is valid | Works for US coins, fails for $\{1,3,4\}$ | Use DP unless proven |
| 4 | Not stress-testing greedy | One counterexample is all it takes | Compare against brute force |
| 5 | Mixing 0-based and 1-based vertices | Builds a valid but wrong graph | Convert at read time only |
| 6 | Forgetting `adj[v].push_back(u)` for undirected | Half the graph disappears | Add both directions |
| 7 | Marking visited on dequeue in BFS | Queue grows to $O(m)$; duplicate distance writes | Mark on enqueue |
| 8 | Recursive flood fill on a $1000 \times 1000$ grid | Stack overflow | Iterative BFS/DFS |
| 9 | `vector<vector<int>> adj(n, vector<int>(n))` for $n = 10^5$ | ~40 GB | Adjacency list |
| 10 | Using BFS on a weighted graph | Distances are wrong | Dijkstra (Chapter 4) |
| 11 | Forgetting `visited` entirely | Infinite loop | Allocate and check |
| 12 | Self-loops or duplicate edges unhandled | Double counting, incorrect degrees | Deduplicate, or ignore self-loops |
| 13 | Assuming the graph is connected | `dist` stays $-1$ for other components | Loop over all sources |
| 14 | Greedy that needs to undo a choice | That is DP, not greedy | Recognise and switch |
| 15 | `queue<int>` but pushing a struct that needs distance | Distances flatten to 1 | Store `(vertex, dist)` or a `dist[]` array |
| 16 | Reading $n-1$ edges when the input gives $m$ | Wrong graph or `cin` timeout | Re-read the statement's first line |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Maximum non-overlapping intervals | CSES "Movie Festival" | Easy |
| 2 | Minimum rooms for lectures | CSES "Room Allocation" | Medium |
| 3 | Fractional knapsack | Classic / GeeksforGeeks | Easy |
| 4 | Minimum coins, small amounts | CSES "Minimizing Coins" (DP, contrast with greedy) | Easy–Medium |
| 5 | Shortest path, unweighted | CSES "Message Route" | Easy |
| 6 | Connected components count | CSES "Building Roads" | Easy |
| 7 | Grid island count | CSES "Counting Rooms" | Easy |
| 8 | Bipartite check by BFS 2-colouring | CSES "Building Teams" | Medium |
| 9 | Graph diameter in edges (two BFS) | Classic / CSES "Tree Diameter" (tree version) | Medium |
| 10 | Greedy that fails — find the counterexample | AtCoder ABC "Coins" | Medium |
| 11 | Grid flood fill with an explicit stack | Codeforces Div. 3 | Medium |
| 12 | Path reconstruction from a parent array | CSES "Message Route" | Medium |

---

## Next Up

→ [Topic 08 — Problems on Greedy Algorithms and Graphs](Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md)

The theory is done. Topic 08 puts it to work: **five greedy problems and five graph problems solved in full**, each with the exchange argument or traversal invariant written out, followed by twenty unsolved problems. The greedy solutions in particular are chosen so that the *wrong* greedy is the tempting one — the point is to build the reflex of proving before coding.
