# Topic 06 - Graph Theory Beginner Part 2 (Intro to Tree Algorithms and Functional Graphs)

Two graph shapes account for a surprising fraction of contest problems.

The first is the **tree**: connected, acyclic, exactly $n - 1$ edges. Its power is that there is exactly one path between any two vertices, so "the path" needs no search - and recursion over a rooted tree can summarise an entire subtree in one number.

The second is the **functional graph**: $n$ vertices, each with exactly one outgoing edge. It looks like an arbitrary directed graph, and it is not - every component is a single cycle with trees hanging off it. That structure is what makes problems like "how many steps until you repeat" solvable in linear time.

Both shapes are recognised from the **edge description**, not from the story, and both are solved by the traversals from Topic 05 plus one extra idea.

> **Prerequisites:** [Topic 05 - Graph Theory Beginner Part 1](Topic%2005%20-%20Graph%20Theory%20Beginner%20Part%201%20%28Graph%20Traversal%20and%20Flood%20Fill%29.md) and [Chapter 2, Topic 08 - Problems on Greedy Algorithms and Graphs](../CP%20Level%201/Topic%2008%20-%20Problems%20on%20Greedy%20Algorithms%20and%20Graphs.md)
> **You will be able to:**
> - Recognise a tree from $n - 1$ edges and root it without recursion.
> - Compute subtree aggregates in one reverse pass.
> - Find a tree's diameter in two traversals and prove why that works.
> - Answer path queries with a naive LCA built from depth and parent arrays.
> - Recognise a functional graph and extract its cycle.
> - Compute distances to the cycle by peeling and reversing.

---

## 1. What a tree is, and the five definitions that are all equivalent

For a graph with $n$ vertices, the following statements are equivalent. Any one of them can be used as the definition, and each is useful in a different situation.

1. Connected and acyclic.
2. Exactly one simple path between every pair of vertices.
3. Connected, with exactly $n - 1$ edges.
4. Acyclic, with exactly $n - 1$ edges.
5. Connected, and removing any edge disconnects it.

The fourth is the one that shows up in input checks: **if you are given $n$ and $m$, and $m \ne n - 1$, it is not a tree** - no traversal needed. If $m = n - 1$, a single traversal that reaches all $n$ vertices settles it.

Two warnings about that shortcut. A graph with $m = n - 1$ and a cycle is necessarily disconnected, so connectivity is the missing half. And a graph with $m = n - 1$ and no cycles is necessarily a tree, because a forest with $c$ components has $n - c$ edges.

### Rooted versus unrooted

A tree has no natural top. **Rooting** it means choosing a vertex and orienting every edge away from it, which gives each vertex a parent and a set of children. Almost every tree algorithm is easier after rooting, and the root is usually given (or may be chosen freely - if the answer depends on the root, the problem will say so).

Rooting costs one traversal and produces `parent[v]` for every $v$. **Depth** is another array filled during the same traversal.

---

## 2. Rooting a tree iteratively

Recursion is the natural way to root a tree, and it is also the way that crashes at $n = 2 \times 10^5$ on a path-shaped tree. The iterative version uses a stack for the traversal and an `order` vector recording the sequence in which vertices were first reached.

```cpp
vector<int> parent(n + 1, 0), order;
vector<int> st = {1};
parent[1] = -1;

while (!st.empty()) {
    int u = st.back();
    st.pop_back();
    order.push_back(u);

    for (int v : adj[u]) {
        if (v == parent[u]) continue;    // do not walk back up
        parent[v] = u;
        st.push_back(v);
    }
}
```

There are no visited arrays here. In a tree, the only already-visited neighbour of `u` is its parent, so skipping the parent is a complete test. That is a genuine property of trees - it is **not** true for general graphs, where a non-parent neighbour closes a cycle.

`order` is a pre-order: every vertex appears after its parent. Reverse it and every vertex appears before its children, which is exactly the order a post-order computation needs.

---

## 3. Solved Problem 1 - Subtree Sizes

> **Statement.** Given a tree with $n$ vertices rooted at 1, print the size of the subtree of every vertex. $n \le 2 \times 10^5$.

### Why this approach

The size of a subtree is one plus the sizes of its children's subtrees. Computing it bottom-up means processing children before parents - which is precisely the reverse of the pre-order from §2. No recursion and no post-order bookkeeping are required.

This is the template for **every** tree aggregate: sums, maximum depths, counts of marked vertices, DP values. Change the combination rule, keep the two passes.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<vector<int>> adj(n + 1);
    for (int i = 0; i < n - 1; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    vector<int> parent(n + 1, 0), order;
    order.reserve(n);
    vector<int> st = {1};
    parent[1] = -1;

    while (!st.empty()) {                 // pass 1: pre-order
        int u = st.back();
        st.pop_back();
        order.push_back(u);

        for (int v : adj[u]) {
            if (v == parent[u]) continue;
            parent[v] = u;
            st.push_back(v);
        }
    }

    vector<int> sz(n + 1, 1);
    for (int i = n - 1; i >= 0; i--) {    // pass 2: reverse pre-order
        int u = order[i];
        if (parent[u] > 0) sz[parent[u]] += sz[u];
    }

    for (int i = 1; i <= n; i++) cout << sz[i] << " \n"[i == n];
    return 0;
}
```

**Input**
```text
7
1 2
1 3
2 4
2 5
3 6
6 7
```

**Output**
```text
7 3 3 1 1 2 1
```

Vertex 1 owns everything. Vertex 2's subtree is $\{2, 4, 5\}$; vertex 3's is $\{3, 6, 7\}$; vertex 6's is $\{6, 7\}$; and the leaves 4, 5, 7 each have size 1.

**Complexity.** $O(n)$ time, $O(n)$ space.

### The trap

Recursive DFS for the traversal. It is correct and it is shorter, but a path of $2 \times 10^5$ vertices is a legal input and recurses $2 \times 10^5$ frames deep. The iterative version is barely longer and never fails.

The second trap is `parent[1] = -1` combined with `sz[parent[u]] += sz[u]` guarded by `parent[u] > 0`. If the root's parent is `0`, that guard is what stops a write to `sz[0]`; if you skip the guard, the root's size silently leaks into an unused slot and the answer for vertex 1 is wrong.

---

## 4. The diameter of a tree: why two passes are enough

The **diameter** is the length of the longest path. Computing all pairs is $O(n^2)$ and unnecessary, because of one theorem:

> From any vertex $s$, a farthest vertex $a$ is an endpoint of some diameter.

So: BFS from 1 to find $a$, then BFS from $a$; the largest distance found in the second traversal is the diameter.

**Why it is true, briefly.** Suppose the true diameter is $u \to v$, and let $a$ be farthest from $s$. The paths from $s$ to $a$ and from $s$ to the diameter must meet somewhere at a vertex $x$ (a tree has unique paths, so they diverge at one point). Distances then add up: $d(a, x) \ge d(u, x)$ because $a$ is farthest from $s$ and $u$ is reachable from $s$ through $x$. Substituting into $d(a, v) \ge d(a, x) + d(x, v)$ gives $d(a, v) \ge d(u, v)$, so $a$ reaches as far as the diameter endpoint does - and the diameter is maximal, so the two are equal.

The theorem needs **non-negative** edge weights. With negative weights the notion of "farthest" collapses and the two-pass trick fails; that case needs a tree DP.

---

## 5. Solved Problem 2 - Tree Diameter

> **Statement.** Given an unweighted tree, output the number of edges on its longest path. $n \le 2 \times 10^5$.

### Why this approach

Two BFS runs by §4. The graph is a tree, so BFS from any vertex reaches all of it, and distances are edge counts.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> adj;

pair<int,int> bfs(int s) {                 // returns (farthest vertex, its distance)
    vector<int> dist(n + 1, -1);
    queue<int> q;

    dist[s] = 0;
    q.push(s);

    int far = s;
    while (!q.empty()) {
        int u = q.front();
        q.pop();

        if (dist[u] > dist[far]) far = u;

        for (int v : adj[u]) {
            if (dist[v] == -1) {
                dist[v] = dist[u] + 1;
                q.push(v);
            }
        }
    }

    return {far, dist[far]};
}

int main() {
    cin >> n;

    adj.assign(n + 1, {});
    for (int i = 0; i < n - 1; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    int a = bfs(1).first;                  // endpoint of a diameter
    cout << bfs(a).second << '\n';         // its distance to the other end
    return 0;
}
```

**Input**
```text
7
1 2
1 3
2 4
2 5
3 6
6 7
```

**Output**
```text
5
```

The longest path is $4 \to 2 \to 1 \to 3 \to 6 \to 7$, which has five edges. BFS from 1 reaches vertex 7 at distance 3; BFS from 7 reaches 4 and 5 at distance 5.

**Complexity.** $O(n)$ time and space - two traversals, each linear.

### The trap

Using DFS and keeping a global maximum of the *depth* at each vertex. That computes the height of the tree from the root, which is the diameter only when the root happens to be an endpoint. The two-pass BFS is not an optimisation of that idea; it is a different and correct one.

Also note that the answer is in **edges**, not vertices. A path through $k$ vertices has $k - 1$ edges, and statements are explicit about which they want.

---

## 6. Ancestors, depth, and the LCA

After rooting, every vertex has an ancestor chain ending at the root. The **lowest common ancestor** of $u$ and $v$ is the deepest vertex that is an ancestor of both, and it is the hinge of every path query.

The naive computation is two steps:

1. Climb the deeper vertex until both are at the same depth.
2. Climb both together until they meet.

```cpp
int lca(int u, int v) {
    while (depth[u] > depth[v]) u = parent[u];
    while (depth[v] > depth[u]) v = parent[v];
    while (u != v) { u = parent[u]; v = parent[v]; }
    return u;
}
```

With it, the distance between two vertices is

$$d(u, v) = \text{depth}[u] + \text{depth}[v] - 2 \cdot \text{depth}[\text{lca}(u, v)].$$

The naive version is $O(\text{height})$ per query and $O(n)$ in the worst case - fine for a single query, hopeless for $2 \times 10^5$ of them on a path-shaped tree.

**Binary lifting** removes the climb. Precompute `up[k][v]`, the $2^k$-th ancestor of `v`:

- `up[0][v] = parent[v]`.
- `up[k][v] = up[k-1][ up[k-1][v] ]`.

There are $\lceil \log_2 n \rceil$ levels, so the table costs $O(n \log n)$ to build and each query becomes $O(\log n)$: lift the deeper vertex by the binary representation of the depth difference, then lift both vertices together from the highest level downward. Vertex `0` is a sentinel whose ancestors are all `0`, which is what makes the top of the table safe.

The distance formula does not change. Only `lca` does.

---

## 7. Solved Problem 3 - Distance Between Two Nodes

> **Statement.** Given a tree with $n$ vertices and $q$ queries $(u, v)$, print the number of edges on the path from $u$ to $v$. $n, q \le 2 \times 10^5$.

### Why this approach

Root the tree to get `parent` and `depth`, build the jump table, and each query is one `lca` call plus the distance formula. The table is what makes the solution $O((n + q)\log n)$ instead of $O(nq)$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> adj;
vector<int> parent, depth;

int LOGN;
vector<vector<int>> up;                  // up[k][v] = 2^k-th ancestor of v

void build(int root) {
    parent.assign(n + 1, 0);
    depth.assign(n + 1, 0);

    vector<int> st = {root};
    parent[root] = 0;

    while (!st.empty()) {
        int u = st.back();
        st.pop_back();

        for (int v : adj[u]) {
            if (v == parent[u]) continue;
            parent[v] = u;
            depth[v] = depth[u] + 1;
            st.push_back(v);
        }
    }

    LOGN = 1;
    while ((1 << LOGN) <= n) LOGN++;

    up.assign(LOGN, vector<int>(n + 1, 0));
    up[0] = parent;                      // 2^0 ancestor; up[k][0] stays 0

    for (int k = 1; k < LOGN; k++)
        for (int v = 1; v <= n; v++)
            up[k][v] = up[k - 1][up[k - 1][v]];
}

int lca(int u, int v) {
    if (depth[u] < depth[v]) swap(u, v);

    int diff = depth[u] - depth[v];
    for (int k = 0; k < LOGN; k++)       // lift the deeper vertex, low levels up
        if ((diff >> k) & 1) u = up[k][u];

    if (u == v) return u;

    for (int k = LOGN - 1; k >= 0; k--)  // lift both together, high levels down
        if (up[k][u] != up[k][v]) { u = up[k][u]; v = up[k][v]; }

    return up[0][u];
}

int main() {
    int q;
    cin >> n >> q;

    adj.assign(n + 1, {});
    for (int i = 0; i < n - 1; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    build(1);

    while (q--) {
        int u, v;
        cin >> u >> v;
        int w = lca(u, v);
        cout << depth[u] + depth[v] - 2 * depth[w] << '\n';
    }

    return 0;
}
```

**Input**
```text
7 3
1 2
1 3
2 4
2 5
3 6
6 7
4 5
4 7
2 3
```

**Output**
```text
2
5
2
```

From 4 to 5 the path is $4 \to 2 \to 5$. From 4 to 7 it is $4 \to 2 \to 1 \to 3 \to 6 \to 7$, five edges. From 2 to 3 it is $2 \to 1 \to 3$.

**Complexity.** $O(n \log n)$ preprocessing, $O(\log n)$ per query, $O((n + q)\log n)$ overall.

### The trap

The `LOGN` bound. `while ((1 << LOGN) <= n) LOGN++;` produces exactly enough levels for every jump up to `n`. One level too few and lifting the deepest vertex silently stops short of the root.

The second trap is the sentinel. `up[k][0]` must remain `0` for every `k`, which is why the table starts zeroed and only vertices `1..n` are filled. Without that, lifting above the root reads uninitialised entries and the comparisons stop meaning anything.

The third is the direction of the two loops. The equalising lift runs from level 0 upward, matching the bits of `diff`; the joint lift runs from the top level downward, so the two vertices approach the LCA without overshooting it. A joint lift that runs upward can jump past the LCA and return an ancestor that is too high - a wrong answer on deep trees only.

---

## 8. Functional graphs: one outgoing edge each

A **functional graph** is a directed graph in which every vertex has out-degree exactly 1. Concretely: an array `f[1..n]` where `f[i]` is the vertex you move to from $i$. Problems phrase it as teleporters, as "each person points at someone", as a permutation composed with something, or as a deterministic state machine.

The structure is completely determined by the out-degree:

> Every component of a functional graph is a single directed cycle, with trees whose edges point toward the cycle.

Follow the edges from any vertex. There are only $n$ vertices, so after at most $n$ steps you must revisit one - and from that point the walk is periodic. The first repeated vertex starts the cycle, and the part before it is the "tail" of a rho ($\rho$) shape.

That gives an immediate $O(n)$ algorithm for "steps until a repeat" by explicit simulation with timestamps - but for anything more, the **peel** is better.

---

## 9. Solved Problem 4 - Extract the Cycle

> **Statement.** Given a functional graph as an array `f[1..n]`, let $s$ be the smallest-numbered vertex that lies on a cycle. Print the cycle through $s$: first its length, then its vertices in traversal order starting from $s$. At least one cycle always exists, because a walk on $n$ finite vertices must eventually repeat one.

### Why this approach

Peel leaves: every vertex with in-degree 0 cannot be on a cycle, so removing it (and its outgoing edge) cannot destroy a cycle. Repeating this leaves **exactly** the cycle vertices. This is Kahn's algorithm, reused.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> f(n + 1), indeg(n + 1, 0);
    for (int i = 1; i <= n; i++) {
        cin >> f[i];
        indeg[f[i]]++;
    }

    queue<int> q;
    for (int i = 1; i <= n; i++)
        if (indeg[i] == 0) q.push(i);

    while (!q.empty()) {                  // peel everything that is not a cycle
        int u = q.front();
        q.pop();

        int v = f[u];
        if (--indeg[v] == 0) q.push(v);
    }

    int start = -1;                       // smallest vertex still standing
    for (int i = 1; i <= n; i++) {
        if (indeg[i] > 0) { start = i; break; }
    }

    vector<int> cycle;
    int cur = start;
    do {
        cycle.push_back(cur);
        cur = f[cur];
    } while (cur != start);

    cout << cycle.size() << '\n';
    for (int i = 0; i < (int)cycle.size(); i++)
        cout << cycle[i] << " \n"[i + 1 == (int)cycle.size()];

    return 0;
}
```

**Input**
```text
6
2 3 1 3 4 4
```

**Output**
```text
3
1 2 3
```

The edges are $1 \to 2$, $2 \to 3$, $3 \to 1$, $4 \to 3$, $5 \to 4$, $6 \to 4$. Vertices 5 and 6 have in-degree 0, so they peel first; that removes 4's incoming edges and 4 peels next; 1, 2 and 3 survive.

**Complexity.** $O(n)$ time and space.

### The trap

Peeling with a stack instead of a queue is fine - the order does not matter for correctness, only the peel itself. What does matter is decrementing the in-degree of `f[u]` **only**, and not of `u`. Removing `u` deletes exactly one edge, the one leaving `u`.

The second trap is treating "remaining after peel" as "reachable from the cycle". It is the opposite: the survivors are precisely the cycle, and everything peeled is off it.

---

## 10. Peeling, then reversing

The peel produces an order in which vertices were removed, and that order is a topological order of the trees hanging off the cycle. Reverse it and every vertex comes after the vertex it points to - which is the correct order for propagating values **towards** the cycle.

That is the general functional-graph technique:

1. Peel with Kahn, recording the removal order.
2. Whatever survives is the cycle; give those vertices their cycle values.
3. Walk the removal order **backwards**, computing each vertex's value from `f[u]`.

The same shape as the subtree sizes in §3: forward pass to get an order, reverse pass to combine. Trees and functional graphs are the same trick with the edges pointing differently.

---

## 11. Solved Problem 5 - Distance to the Cycle

> **Statement.** For every vertex of a functional graph, print the number of steps needed to reach a cycle vertex. Cycle vertices themselves answer 0. $n \le 2 \times 10^5$.

### Why this approach

Peel, then reverse. A cycle vertex is at distance 0; a peeled vertex is one step further than the vertex it points to. Processing the removal order backwards guarantees that `f[u]`'s distance is already known.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<int> f(n + 1), indeg(n + 1, 0);
    for (int i = 1; i <= n; i++) {
        cin >> f[i];
        indeg[f[i]]++;
    }

    vector<int> order;
    queue<int> q;

    for (int i = 1; i <= n; i++)
        if (indeg[i] == 0) q.push(i);

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        order.push_back(u);

        int v = f[u];
        if (--indeg[v] == 0) q.push(v);
    }

    vector<int> dist(n + 1, 0);           // survivors keep 0

    for (int i = (int)order.size() - 1; i >= 0; i--) {
        int u = order[i];
        dist[u] = dist[f[u]] + 1;
    }

    for (int i = 1; i <= n; i++) cout << dist[i] << " \n"[i == n];
    return 0;
}
```

**Input**
```text
6
2 3 1 3 4 4
```

**Output**
```text
0 0 0 1 2 2
```

Vertices 1, 2, 3 are the cycle. Vertex 4 reaches it in one step, and 5 and 6 in two.

**Complexity.** $O(n)$ time and space.

### The trap

Iterating the removal order **forwards**. Then `dist[u]` is computed before `dist[f[u]]` exists, so every tail distance comes out as 1 (from the initialised zeros) and the answer looks plausible on shallow inputs. The direction of the second pass is the whole algorithm.

The second trap: initialising `dist` to `-1` and forgetting to set the cycle vertices to `0`. The formula then adds one to `-1` and produces zeros for the first ring - which is indistinguishable from "on the cycle" in the output.

---

## 12. Trees and functional graphs side by side

| Question | Tree | Functional graph |
|---|---|---|
| Recognise it | $m = n - 1$, connected | every vertex has out-degree 1 |
| Structure | one path between any two | one cycle per component, trees feeding it |
| Natural order | pre-order and its reverse | peel order and its reverse |
| Cycle finding | parent test (§7 of Topic 05) | Kahn peel |
| Distance queries | LCA + depth | follow `f`, or precompute per component |
| Where recursion fails | path-shaped trees | long chains |

The last row is the practical warning. Both structures look shallow in samples and both can be a single chain of length $n$ in system tests. The iterative forms above are the ones to memorise.

---

## 13. Complexity summary

| Task | Time | Extra space |
|---|---|---|
| Root a tree (parent, depth, order) | $O(n)$ | $O(n)$ |
| Subtree aggregate | $O(n)$ | $O(n)$ |
| Diameter (unweighted, non-negative weights) | $O(n)$ | $O(n)$ |
| LCA, naive | $O(n)$ per query | $O(n)$ |
| LCA, binary lifting | $O(n \log n)$ build, $O(\log n)$ per query | $O(n \log n)$ |
| Cycle extraction in a functional graph | $O(n)$ | $O(n)$ |
| Distance to cycle | $O(n)$ | $O(n)$ |

**Input pitfalls specific to these two shapes:**

- A tree input has $n - 1$ edges. A loop `for (i = 0; i < m; i++)` with `m` read from input is correct; hard-coding `n` reads one edge too many and blocks.
- A functional graph input is usually a single line of $n$ values, `f[1..n]`, with no edge list at all. Do not look for `m`.
- Vertices are 1-indexed in both cases, which is why every array is sized `n + 1`.

---

## 14. Unsolved problems

| # | Problem | Constraints |
|---|---|---|
| 1 | **Path Sum Queries.** Each vertex has a value. Given $q$ queries $(u, v)$, print the sum of values on the path between them. | $n, q \le 2 \times 10^5$ |
| 2 | **Weighted Diameter.** The tree's edges have positive weights. Find the maximum total weight of a path. | $n \le 2 \times 10^5$, $w \le 10^9$ |
| 3 | **Count Functional Components.** Given `f[1..n]`, print the number of connected components, treating edges as undirected. | $n \le 2 \times 10^5$ |
| 4 | **Is It a Tree?** Given $n$ vertices and $m$ edges, decide whether the graph is a tree. | $n, m \le 2 \times 10^5$ |
| 5 | **Meeting Point.** In a functional graph, for two vertices $u$ and $v$, find the first vertex that both eventually reach, or report that they never meet. | $n, q \le 2 \times 10^5$ |

### Hints, in increasing strength

1. Problem 1 needs the root-to-vertex sums: precompute `pref[v] = pref[parent[v]] + value[v]`, then the path sum is `pref[u] + pref[v] - 2*pref[lca] + value[lca]`. Check the inclusion-exclusion on a two-vertex path before trusting it.
2. Problem 2 is the same two-pass diameter as §5, with BFS replaced by a DFS that accumulates weights. The theorem holds for non-negative weights, so no tree DP is needed.
3. Problem 3: each component has exactly one cycle, so the number of components equals the number of cycles. After the peel, count the cycles among the survivors by walking each one and marking it.
4. Problem 4 is two independent checks: $m = n - 1$, and one traversal reaches all $n$ vertices. Either alone is insufficient.
5. Problem 5 is the functional-graph version of LCA. Both walks must eventually reach the same cycle; if their cycles differ, the answer is "never". If not, precompute for each cycle the positions of its members, and the meeting point is determined by the entry points and the cycle length.

---

## Key Takeaways

- Tree $\iff$ connected and acyclic $\iff$ $n - 1$ edges and connected $\iff$ unique paths.
- Rooting is one traversal producing `parent`, `depth` and a pre-order.
- Reverse pre-order is post-order: the order in which children are finished before parents.
- Every subtree aggregate is two passes: build the order, then walk it backwards.
- The iterative traversal is the default at $10^5$ vertices; recursion is the exception.
- A farthest vertex from any start is an endpoint of a diameter, so two traversals find it.
- The two-pass diameter needs non-negative weights; negative weights require tree DP.
- Distance between two vertices is `depth[u] + depth[v] - 2*depth[lca]`.
- Naive LCA is $O(\text{height})$; binary lifting is $O(\log n)$ and only changes that function.
- A functional graph has out-degree 1 everywhere, and every component is one cycle with trees feeding it.
- Kahn's peel leaves exactly the cycle vertices.
- Reverse peel order propagates values from the cycle outwards.
- Peeling and reverse-order combination is the same idea as reverse pre-order on a tree.
- Both shapes can be a single chain of length $n$; the iterative forms are the safe ones.

---

## Common Mistakes & Pitfalls

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Reading $n$ edges for a tree | Blocks on input or reads out of range | `n - 1` edges |
| 2 | Recursive DFS on a path-shaped tree | Stack overflow | Iterative stack |
| 3 | Writing `sz[parent[u]]` without a root guard | Wrong size for the root | Guard `parent[u] > 0` |
| 4 | Applying reverse pre-order after a BFS that already reversed | Sizes propagate downward | Keep the pre-order, then reverse once |
| 5 | Diameter as the maximum depth from the root | Correct only if the root is an endpoint | Two traversals |
| 6 | Diameter with negative weights | Wrong endpoint found | Tree DP |
| 7 | Reporting vertices instead of edges | Off by one | Read the statement |
| 8 | LCA climbing both vertices before equalising depth | Loop never ends or wrong ancestor | Equalise first |
| 9 | `parent[root]` left uninitialised | Random ancestor, infinite loop | `parent[root] = 0` with 1-indexing |
| 10 | Path sum forgetting the LCA vertex | Answer short by one value | Subtract twice, add once |
| 11 | Assuming the input gives `m` for a functional graph | Reads garbage | It is an array of $n$ values |
| 12 | Peeling the in-degree of the wrong vertex | Cycles survive incorrectly | Decrement `indeg[f[u]]` |
| 13 | Forward iteration of the peel order | All tail distances equal 1 | Iterate backwards |
| 14 | `dist` initialised to `-1` with cycle vertices left at `-1` | First ring reports 0 | Set cycle vertices to 0 explicitly |

---

## Practice Problems

The five unsolved problems above are the practice set, in a sensible order:

| Order | Problem | Why first |
|---|---|---|
| 1 | 4 - Is It a Tree? | Pure recognition; no new algorithm |
| 2 | 2 - Weighted Diameter | Two traversals with a sum instead of a count |
| 3 | 1 - Path Sum Queries | The distance formula plus one prefix array |
| 4 | 3 - Count Functional Components | The peel, counting rather than extracting |
| 5 | 5 - Meeting Point | Functional-graph LCA; the hardest idea here |

For more, CSES *Tree Algorithms* contains *Subordinates* (subtree sizes), *Tree Diameter*, and *Company Queries I*, which is binary lifting and the natural continuation of §6.

---

## Next Up

→ [Topic 07 - Bitwise Operators](Topic%2007%20-%20Bitwise%20Operators.md)

Graphs are behind us. The next topic is a change of representation: the same integer read bit by bit, where `&`, `|`, `^` and shifts turn set operations into single instructions - and where a loop over subsets of a mask becomes a technique of its own.
