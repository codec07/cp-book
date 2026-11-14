# Topic 05 - Graph Theory Beginner Part 1 (Graph Traversal and Flood Fill)

A graph problem rarely announces itself as one.

"The virus spreads to adjacent cells each minute" is a graph. "Can these cities be split into two groups with no road inside a group" is a graph. "How many clicks to get from this word to that word, changing one letter at a time" is a graph - the words are the vertices and you never see an edge written down.

This topic is about the two traversals that answer almost every beginner question, **BFS** and **DFS**, and about the one skill that matters more than either: *seeing* the graph. Once the graph exists, the algorithms are twenty lines each and you will write them from memory.

> **Prerequisites:** [Chapter 2, Topic 07 - Introduction to Greedy Algorithms and Graphs](../CP%20Level%201/Topic%2007%20-%20Introduction%20to%20Greedy%20Algorithms%20and%20Graphs.md) and [Chapter 2, Topic 08 - Problems on Greedy Algorithms and Graphs](../CP%20Level%201/Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md)
> **You will be able to:**
> - Store a graph in the representation the constraints demand.
> - Write BFS and DFS from memory, correctly, on the first try.
> - Find shortest paths in unweighted graphs and say why BFS is exactly right.
> - Detect cycles and count connected components.
> - Treat a grid as a graph and run flood fill without stack overflows.
> - Test bipartiteness by two-colouring.
> - Recognise a multi-source BFS problem before writing the wrong loop.

---

## 1. A graph is a set of pairs

Formally, a graph is a set of vertices $V$ and a set of edges $E \subseteq V \times V$. Every contest graph problem is a question about that structure, dressed in a story.

The vocabulary you need:

| Term | Meaning | Typical contest phrasing |
|---|---|---|
| Undirected edge | $\{u, v\}$ - travel both ways | "roads", "friendships" |
| Directed edge | $(u, v)$ - travel one way | "one-way streets", "prerequisites" |
| Weighted edge | an edge that carries a cost | "distance", "time", "toll" |
| Degree | number of edges at a vertex | "number of friends" |
| Path | a sequence of adjacent vertices | "route" |
| Cycle | a path that returns to its start | "loop" |
| Connected component | a maximal set of mutually reachable vertices | "islands", "provinces", "groups" |

Two facts decide most beginner solutions before any code is written:

- **Unweighted $\Rightarrow$ BFS.** Every edge costs the same, so the shortest path is the one with the fewest edges, and BFS discovers vertices in order of edge count.
- **"How many separate groups" $\Rightarrow$ count the components.** One traversal from an unvisited vertex marks exactly one component.

The third fact is a warning: a graph with $10^5$ vertices and $10^5$ edges is fine, but the same graph stored as a $10^5 \times 10^5$ matrix is $10^{10}$ cells. The next section is entirely about not doing that.

---

## 2. Three ways to store a graph, and how the constraints choose

### Adjacency matrix

`adj[u][v] = 1` if the edge exists. $O(1)$ edge lookup, $O(V^2)$ memory.

Use it only when $V \le 5000$ **and** you need constant-time edge queries - for example, Floyd-Warshall, which needs the matrix anyway.

### Adjacency list

```cpp
vector<vector<int>> adj(n + 1);
// undirected edge u-v
adj[u].push_back(v);
adj[v].push_back(u);
```

$O(V + E)$ memory, $O(\deg u)$ to iterate the neighbours of $u$. This is the default. At $V = 2 \times 10^5$ and $E = 2 \times 10^5$ it uses a few megabytes.

### Edge list

```cpp
vector<pair<int,int>> edges;   // (u, v)
```

Good when the algorithm processes edges once and never needs a vertex's neighbourhood - Kruskal, Bellman-Ford. Not a traversal structure.

### Choosing

| Constraint | Representation |
|---|---|
| $V \le 5000$, dense, $E \approx V^2$ | matrix |
| $V \le 2 \times 10^5$, sparse | adjacency list |
| Edges processed once, no traversal | edge list |
| Grid, implicit | the grid itself |

The input convention matters more than the structure. Read the first line as `n m`, then $m$ lines of `u v`, and remember that **a graph with $n$ vertices and $m$ edges does not have $n = m$.** Mixing those two up is the most common wrong-answer in this topic, and it usually appears as a crash after a randomly correct sample.

---

## 3. Solved Problem 1 - Counting Connected Components

> **Statement.** Given an undirected graph with $n$ vertices and $m$ edges, count its connected components. Vertices are numbered $1 \ldots n$. $n, m \le 2 \times 10^5$.

### Why this approach

A component is "everything reachable from one vertex". Start a traversal at any unvisited vertex; the traversal consumes exactly that component. Then move to the next unvisited vertex and repeat. The number of times the outer loop starts a traversal is the number of components.

### Solution

Mark vertices when they are **pushed**, not when they are popped. Marking on push prevents the same vertex from entering the queue twice, which keeps the queue at $O(V)$ and the total time at $O(V + E)$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;

    vector<vector<int>> adj(n + 1);
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);          // undirected: both directions
    }

    vector<int> seen(n + 1, 0);
    int components = 0;

    for (int s = 1; s <= n; s++) {
        if (seen[s]) continue;
        components++;

        queue<int> q;
        q.push(s);
        seen[s] = 1;

        while (!q.empty()) {
            int u = q.front();
            q.pop();

            for (int v : adj[u]) {
                if (!seen[v]) {
                    seen[v] = 1;      // mark on push
                    q.push(v);
                }
            }
        }
    }

    cout << components << '\n';
    return 0;
}
```

**Input**
```text
6 3
1 2
2 3
4 5
```

**Output**
```text
3
```

The components are $\{1, 2, 3\}$, $\{4, 5\}$ and $\{6\}$. Vertex 6 has no edges at all, which is why the outer loop must scan every vertex rather than starting from vertex 1.

**Complexity.** $O(n + m)$ time, $O(n + m)$ space.

### The trap

Starting the traversal only from vertex 1. Isolated vertices and separate components are then never counted, and on inputs where the whole graph happens to be connected the code passes. The outer `for` is not optional.

The second trap is marking on pop instead of on push. That version is still *correct* for counting components, but it enqueues each vertex once per incident edge, so a vertex of high degree can appear in the queue thousands of times.

---

## 4. BFS: the shortest-path machine

Breadth-first search explores in rings: first the source, then everything at distance 1, then everything at distance 2. The queue is what enforces the ordering - a vertex discovered at distance $d$ is enqueued behind every vertex at distance $< d$, so it is processed after them.

```cpp
vector<int> dist(n + 1, -1);
queue<int> q;
dist[s] = 0;
q.push(s);

while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : adj[u]) {
        if (dist[v] == -1) {          // unvisited
            dist[v] = dist[u] + 1;
            q.push(v);
        }
    }
}
```

Three properties are worth stating explicitly, because they are what you will be asked to justify:

1. **`dist[v]` is the minimum number of edges from `s` to `v`.** When `v` is first discovered, it is discovered from a vertex at level $d - 1$, where $d$ is the smallest level that can reach it. Any shorter path would have been discovered earlier.
2. **`-1` means unreachable**, and that is the only correct sentinel - not `0`, which is a legitimate distance.
3. **BFS is $O(V + E)$**, and it stops being correct the moment edges have weights. A weighted graph needs Dijkstra, which is Level 3.

The distance array doubles as the visited array. There is no reason to keep both.

---

## 5. Solved Problem 2 - Shortest Paths in an Unweighted Graph

> **Statement.** Given an undirected unweighted graph and a source $s$, print the shortest distance from $s$ to every vertex $1 \ldots n$, or `-1` if unreachable. $n, m \le 2 \times 10^5$.

### Why this approach

This is the definition of what BFS computes. There is no search over answers, no binary search, no greedy argument - the algorithm and the problem are the same thing.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n, m, s;
    cin >> n >> m >> s;

    vector<vector<int>> adj(n + 1);
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    vector<int> dist(n + 1, -1);
    queue<int> q;

    dist[s] = 0;
    q.push(s);

    while (!q.empty()) {
        int u = q.front();
        q.pop();

        for (int v : adj[u]) {
            if (dist[v] == -1) {
                dist[v] = dist[u] + 1;
                q.push(v);
            }
        }
    }

    for (int i = 1; i <= n; i++) cout << dist[i] << " \n"[i == n];
    return 0;
}
```

**Input**
```text
5 3 1
1 2
2 3
3 4
```

**Output**
```text
0 1 2 3 -1
```

The path $1 \to 2 \to 3 \to 4$ has length 3, and vertex 5 is isolated.

**Complexity.** $O(n + m)$ time, $O(n + m)$ space.

### The trap

Using the `visited` array of Problem 1 as the distance array, so every reachable vertex reports distance `0` or `1`. The distance *is* the visited marker: initialise it to `-1`, and the first write records both facts.

Also notice the output format `" \n"[i == n]`: a space after every element except the last. It is a one-line idiom for exactly-one-space-separated output.

---

## 6. DFS: the same graph, different order

Depth-first search goes as deep as it can, then backtracks. It does not produce shortest paths. What it does produce is **structure**: the order in which vertices finish, the nested shape of recursion, and a cheap way to mark an entire region.

```cpp
vector<int> seen(n + 1, 0);

function<void(int)> dfs = [&](int u) {
    seen[u] = 1;
    for (int v : adj[u])
        if (!seen[v]) dfs(v);
};
```

Use DFS when you want:

- to mark a whole connected region (flood fill),
- to detect a cycle by tracking the parent,
- to compute entry/exit times or subtree sizes,
- a topological order (Level 3).

Use BFS when you want distances.

The recursion depth is the length of the longest simple path, which can be $n$. At $n = 10^5$ that is fine with a normal stack on most judges and marginal on a few; at $n = 10^6$ it is a guaranteed crash. If the graph is that large, write the iterative version with an explicit `vector<int>` stack, or simply use BFS - for marking regions, BFS and DFS are interchangeable.

---

## 7. Solved Problem 3 - Cycle Detection in an Undirected Graph

> **Statement.** Given an undirected graph, decide whether it contains a cycle. $n, m \le 2 \times 10^5$.

### Why this approach

In a DFS, the edges you follow form a tree. Any other edge goes to a vertex you have already visited - and in an undirected graph, the only visited vertex you are allowed to see is your **parent**, because that is the edge you just came from. Seeing any other visited neighbour means two different paths join, which closes a cycle.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, m;
vector<vector<int>> adj;
vector<int> seen;
bool hasCycle = false;

void dfs(int u, int parent) {
    seen[u] = 1;

    for (int v : adj[u]) {
        if (v == parent) continue;        // the edge we came through

        if (seen[v]) {                    // visited and not the parent
            hasCycle = true;
            return;
        }
        dfs(v, u);
    }
}

int main() {
    cin >> n >> m;

    adj.assign(n + 1, {});
    seen.assign(n + 1, 0);

    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    for (int s = 1; s <= n; s++)
        if (!seen[s] && !hasCycle) dfs(s, 0);

    cout << (hasCycle ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
3 3
1 2
2 3
1 3
```

**Output**
```text
YES
```

Vertex 3 sees two visited neighbours, 2 and 1. The edge $3 - 1$ is not the edge DFS arrived by, so the triangle closes.

**Complexity.** $O(n + m)$ time.

### The trap

The parent test assumes at most one edge between a pair of vertices. With a **multi-edge** - the input contains `1 2` twice - the second copy looks like a cycle to this code but is not one, and the answer is a false `YES`.

There are two clean fixes. Merge duplicate edges while reading (sort each adjacency list and `unique` it), or use the edge-count criterion: a component with $V_c$ vertices and $E_c$ edges has a cycle if and only if $E_c \ge V_c$. The second is more robust and is worth remembering, because it also works when you are counting components anyway.

The `!hasCycle` guard in the outer loop is not required for correctness but avoids pointless work once the answer is known.

---

## 8. Grids are graphs in disguise

A grid is a graph where the vertices are cells and the edges join **side-adjacent** cells. That definition gives you everything:

| Grid concept | Graph concept |
|---|---|
| Cell $(r, c)$ | Vertex |
| Side-adjacent cells | Edge, cost 1 |
| Wall / blocked cell | Vertex removed from the graph |
| "Number of regions" | Connected components |
| "Minimum steps" | Shortest path, so BFS |
| "Spread in one minute" | BFS levels |

The two decisions that produce most grid bugs:

**Directions.** Four-way movement is `dr = {1,-1,0,0}`, `dc = {0,0,1,-1}`. Eight-way adds the four diagonals. Do not hand-write the four neighbour calls unless the problem is tiny; the array makes the code uniform and the mistakes visible.

**Bounds.** Check `0 <= nr < R && 0 <= nc < C` **before** reading `grid[nr][nc]`. Reading first and checking later is out-of-bounds access, which is undefined behaviour - it may crash, and on some judges it silently reads a neighbouring row.

Marking visited by overwriting the cell (turning `'1'` into `'0'`) is standard in flood fill and saves an entire boolean grid. It also changes the input, so do it deliberately and never read the original grid afterwards.

---

## 9. Solved Problem 4 - Islands in a Binary Grid

> **Statement.** Given a grid of `0`s (water) and `1`s (land), count the islands. Cells connect through shared **sides**, not corners. $R, C \le 1000$.

### Why this approach

An island is a connected component of land cells. Scan the grid; every time an untouched `1` is found, it is a new island, so flood fill from it and sink the whole region. Every cell is visited a constant number of times.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int R, C;
vector<string> grid;

void sink(int r, int c) {
    if (r < 0 || r >= R || c < 0 || c >= C) return;   // bounds first
    if (grid[r][c] != '1') return;                    // water or already sunk
    grid[r][c] = '0';                                 // mark visited

    sink(r + 1, c);
    sink(r - 1, c);
    sink(r, c + 1);
    sink(r, c - 1);
}

int main() {
    cin >> R >> C;

    grid.resize(R);
    for (auto &row : grid) cin >> row;

    int islands = 0;

    for (int r = 0; r < R; r++) {
        for (int c = 0; c < C; c++) {
            if (grid[r][c] == '1') {
                islands++;
                sink(r, c);
            }
        }
    }

    cout << islands << '\n';
    return 0;
}
```

**Input**
```text
4 5
11000
11000
00100
00011
```

**Output**
```text
3
```

The regions are the $2 \times 2$ block at the top left, the single `1` in the middle, and the pair in the bottom right corner.

**Complexity.** $O(R \cdot C)$ time and $O(R \cdot C)$ stack space in the worst case.

### The trap

Recursion depth. A grid that is a single snake of $10^6$ land cells recurses a million frames deep and crashes - and the input that triggers it is easy for a setter to include. At $R \cdot C \ge 250{,}000$, use an explicit stack or BFS.

The second trap is the order of the two checks inside `sink`. Bounds must be tested before the grid is indexed; an out-of-range `r` dereferences outside the vector.

Reading rows as `string` is also deliberate: `cin >> row` skips whitespace, so grids written with or without spaces between digits both work, and a row of `'0'` characters is preserved instead of being parsed as a single number.

---

## 10. Two colours and one odd cycle

A graph is **bipartite** if its vertices can be split into two sets with every edge crossing between them. Equivalently, it can be 2-coloured so that no edge joins equal colours.

The test is one traversal. Colour the first vertex 0, give every neighbour colour $1 - c$, and if a neighbour already has the same colour as its parent, the graph is not bipartite.

Why this is equivalent to "no odd cycle": walking around any cycle flips the colour at each step, so returning to the start with the same colour requires an **even** number of steps. An odd cycle therefore forces a conflict - and conversely, every non-bipartite graph contains one.

```cpp
// inside a BFS or DFS from u with colour c
color[u] = c;
for (int v : adj[u]) {
    if (color[v] == -1)      dfs(v, 1 - c);
    else if (color[v] == c)  bipartite = false;
}
```

BFS and DFS both work here. DFS is shorter, so use DFS.

---

## 11. Solved Problem 5 - Is the Graph Bipartite?

> **Statement.** Given an undirected graph, print `YES` if it is bipartite and `NO` otherwise. The graph may be disconnected. $n, m \le 2 \times 10^5$.

### Why this approach

Two-colour every component. A disconnected graph is bipartite exactly when each component is, so the outer loop runs the colouring once per component and a single conflict anywhere settles the answer.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, m;
vector<vector<int>> adj;
vector<int> color;
bool ok = true;

void dfs(int u, int c) {
    color[u] = c;

    for (int v : adj[u]) {
        if (color[v] == -1) {
            dfs(v, 1 - c);
        } else if (color[v] == c) {
            ok = false;          // an edge inside one colour class
            return;
        }
    }
}

int main() {
    cin >> n >> m;

    adj.assign(n + 1, {});
    color.assign(n + 1, -1);

    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    for (int s = 1; s <= n; s++)
        if (color[s] == -1 && ok) dfs(s, 0);

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

The cycle has length 4: colours alternate $0,1,0,1$ and the closing edge $4 - 1$ joins different colours. A triangle with the same vertices and one fewer edge would force $1 - 3$ to join equal colours, and the answer would be `NO`.

**Complexity.** $O(n + m)$ time.

### The trap

Returning `true` from the DFS as soon as one component is successfully coloured. The outer loop must continue over the remaining components; a conflict in the third component still makes the whole graph non-bipartite.

A subtler version of the same mistake: resetting `color` between components. The vertex numbering is global, so the array is global too.

---

## 12. Multi-source BFS: the super-source trick

"Distance from each cell to the nearest of several sources" is still a single BFS. Push **all** sources into the queue at the start, all at distance 0, and run the ordinary loop.

```cpp
vector<int> dist(n + 1, -1);
queue<int> q;

for (int s : sources) {
    dist[s] = 0;
    q.push(s);
}

while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : adj[u]) {
        if (dist[v] == -1) {
            dist[v] = dist[u] + 1;
            q.push(v);
        }
    }
}
```

The correctness argument: adding a virtual vertex connected to every source with a zero-cost edge makes this an ordinary BFS from that vertex. The virtual vertex is never materialised, only its effect on the queue.

This is exactly the shape of "rotten oranges" and "time until every cell is infected". Running one BFS per source is $O(k(n+m))$ and times out; the multi-source version is $O(n+m)$.

---

## 13. Complexity, memory, and the sizes that actually appear

Every traversal in this topic is $O(V + E)$ time. The constants are what you have to watch at the top of the constraint range.

| Input size | What to use | What breaks |
|---|---|---|
| $n, m \le 2 \times 10^5$ | adjacency list + BFS | nothing |
| $n \le 10^6$, grid | BFS or iterative DFS | recursion depth |
| $n \le 5000$, dense | matrix or list, both fine | reading $m$ as $n$ |
| $m$ up to $10^6$ | adjacency list of `int` | `vector<vector<int>>` overhead |

A `vector<vector<int>>` with $10^6$ entries costs about 24 bytes per vector before any element is stored - roughly 24 MB of pure overhead. That is usually acceptable; if it is not, use a flat adjacency array (CSR format), which is a Level 3 concern.

Two input details cost more marks than any algorithmic mistake:

- **`n` and `m` are different.** The graph has $n$ vertices and $m$ edges, and both can be $2 \times 10^5$.
- **Undirected means two pushes.** Forgetting the reverse edge turns the graph into a directed one, and results become direction-dependent in a way that passes some samples.

---

## 14. Unsolved problems

| # | Problem | Constraints |
|---|---|---|
| 1 | **Rotten Oranges.** A grid holds fresh (`1`), rotten (`2`) and empty (`0`) cells. Each minute every fresh orange adjacent to a rotten one rots. Print the minutes until none are fresh, or `-1` if impossible. | $R, C \le 1000$ |
| 2 | **Grid Shortest Path.** A grid contains open cells (`.`) and walls (`#`). Find the minimum number of steps from the top-left cell to the bottom-right, moving in four directions, or `-1`. | $R, C \le 1000$ |
| 3 | **Flood Fill.** Given a grid of colours and a starting cell, replace the colour of the whole region connected to it (four-directionally, same original colour) with a new colour. Print the resulting grid. | $R, C \le 500$ |
| 4 | **Directed Cycle.** Given a directed graph, decide whether it contains a directed cycle. | $n, m \le 2 \times 10^5$ |
| 5 | **Word Ladder.** Given a start word, an end word, and a dictionary, find the length of the shortest transformation sequence changing one letter at a time, where every intermediate word is in the dictionary. | up to $5000$ words, length $\le 10$ |

### Hints, in increasing strength

1. Problem 1 is multi-source BFS with the answer being the maximum distance written, plus a final scan for unreached fresh cells. The empty cells are simply not part of the graph.
2. Problem 2 is BFS where the only difference from Problem 4 is that the visited grid stores distances and walls are never enqueued.
3. Problem 3 stores the **original** colour before painting, otherwise the region boundary moves as you paint and the fill leaks or stops early.
4. Problem 4 cannot use the parent trick from §7 - in a directed graph an edge back to an ancestor along a different branch is still not a cycle. Use three states: unvisited, in the current recursion stack, finished. A back edge to an "in stack" vertex is the cycle.
5. Problem 5 has no edges in the input. Build them by comparing every pair of words of equal length, or - better - group words by their pattern with one character replaced, which turns an $O(k^2 L)$ edge build into $O(k L^2)$.

---

## Key Takeaways

- A graph is a set of pairs; the story, the grid and the word list are all disguises.
- Adjacency list by default; matrix only for dense graphs that need $O(1)$ edge queries.
- An undirected edge needs two `push_back`s. Forgetting one is the most common silent wrong answer.
- BFS explores in rings and therefore computes shortest paths in unweighted graphs.
- `dist = -1` doubles as the visited marker; there is no separate `visited` array.
- Mark on push, not on pop, to keep the queue linear.
- DFS gives structure (regions, cycles, subtrees), not distances.
- Undirected cycle detection: a visited neighbour that is not the parent closes a cycle.
- A component with $E_c \ge V_c$ edges contains a cycle - the robust test when multi-edges are possible.
- Grids are graphs: cells are vertices, sides are edges, walls delete vertices.
- Test bounds before indexing, and pick four or eight directions explicitly.
- Flood fill may destroy the input grid; that is normal and should be intentional.
- Bipartite $\iff$ two-colourable $\iff$ no odd cycle, and one traversal decides it.
- Multi-source BFS pushes all sources at distance 0 - never one BFS per source.
- Recursion depth equals the longest simple path; at $10^6$ vertices, go iterative.

---

## Common Mistakes & Pitfalls

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Reading `m` as `n` (or vice versa) | Crash or garbage after a correct sample | Read `n m` separately |
| 2 | One `push_back` for an undirected edge | Wrong answers on asymmetric samples | Push both directions |
| 3 | Traversal started only from vertex 1 | Undercounts components | Outer loop over all vertices |
| 4 | Marking on pop | Still correct, but $O(E)$ queue entries | Mark on push |
| 5 | `0` as the unreachable sentinel | Distances of 0 for unreachable vertices | Use `-1` |
| 6 | BFS on a weighted graph | Non-minimal paths | Dijkstra (Level 3) |
| 7 | Parent-only cycle check with multi-edges | False `YES` | Deduplicate edges or compare $E_c$ with $V_c$ |
| 8 | Grid bounds checked after indexing | UB, crashes, or silent wrong answers | Bounds first |
| 9 | Diagonal moves in a four-direction problem | Regions merge that should not | Choose directions deliberately |
| 10 | Deep recursion on a $10^6$-cell grid | Stack overflow | Iterative DFS or BFS |
| 11 | Colour array reset per component | Misses conflicts across components | One global array |
| 12 | One BFS per source in a multi-source problem | Time limit exceeded | Initialise the queue with all sources |
| 13 | Forgetting unreached cells in grid BFS | Reports a distance that does not exist | Scan for `-1` at the end |
| 14 | Reading a digit grid with `cin >> int` per cell | Works, but slow and error-prone | Read rows as `string` |

---

## Practice Problems

The five unsolved problems above are the practice set. A sensible order:

| Order | Problem | Why first |
|---|---|---|
| 1 | 2 - Grid Shortest Path | Plain BFS; no extra state |
| 2 | 3 - Flood Fill | Plain DFS; introduces "original colour" |
| 3 | 1 - Rotten Oranges | Multi-source BFS on a grid |
| 4 | 5 - Word Ladder | The graph must be discovered, not given |
| 5 | 4 - Directed Cycle | Three-state DFS; the hardest idea here |

For more, the CSES *Graph Algorithms* section opens with *Counting Rooms* and *Labyrinth*, which are exactly Problems 3 and 2 at a larger scale. Solve at least one of them with BFS and one with DFS to feel the difference.

---

## Next Up

→ [Topic 06 - Graph Theory Beginner Part 2](Topic%2006%20-%20Graph%20Theory%20Beginner%20Part%202%20%28Intro%20to%20Tree%20Algos%20and%20Intro%20to%20Functional%20Graphs%29.md)

Every graph so far has been arbitrary. The next topic restricts the shape - to trees, where there is exactly one path between any two vertices and recursion becomes a tool rather than a traversal, and to functional graphs, where every vertex has exactly one outgoing edge and cycles hide in plain sight.
