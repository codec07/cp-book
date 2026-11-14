# Topic 14 — STL Part 3: Stacks, Queues and Priority Queues

Three containers, three disciplines of ordering. A `stack` hands you the most recently pushed element, a `queue` the oldest, and a `priority_queue` the "best" one under a rule you define. Between them they cover DFS, BFS, bracket matching, monotonic-stack tricks, greedy scheduling and Dijkstra — a large slice of everything you will ever submit.

> **Prerequisites:** [Topic 07 — Loops & `auto`](Topic%2007%20-%20Loops%20%26%20%27Auto%27.md), [Topic 09 — Arrays](Topic%2009%20-%20Arrays%20%281D%2C%202D%20and%203D%29.md), [Topic 11 — Functions, Classes and Lambdas](Topic%2011%20-%20Functions%2C%20Classes%20%28Basics%29%20and%20Useful%20Lambda%20Methods.md), [Topic 12 — STL Part 1](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md), [Topic 13 — STL Part 2](Topic%2013%20-%20STL%20Part%202%20%28Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps%29.md)

> **You will be able to:**
> - Explain why these are *adaptors* and what the `Container` template parameter selects.
> - Write bracket-matching, next-greater-element, stock-span and histogram solutions without fumbling `pop()`.
> - Build a min-heap three different ways and know which to use under time pressure.
> - Write a `priority_queue` comparator without inverting its sense.
> - Implement Dijkstra with lazy deletion, and know when `set<pair<int,int>>` is better.
> - Pick between `stack`, `queue`, `deque`, `priority_queue` and `set` in seconds.

---

## 1. Container Adaptors, Not Containers

A `stack` owns no storage. It holds an object of some other container type and restricts the interface you can reach — that is what *adaptor* means. The standard declares them roughly as:

```cpp
template<class T, class Container = deque<T>> class stack;
template<class T, class Container = deque<T>> class queue;
template<class T, class Container = vector<T>,
         class Compare = less<T>> class priority_queue;
```

The second parameter is the whole idea:

| Adaptor | `Container` default | `Compare` | What it requires of `Container` |
|---|---|---|---|
| `stack` | `deque<T>` | — | `back()`, `push_back()`, `pop_back()`, `empty()`, `size()` |
| `queue` | `deque<T>` | — | `front()`, `back()`, `push_back()`, `pop_front()` |
| `priority_queue` | `vector<T>` | `less<T>` | random access + `push_back`/`pop_back`, so heap algorithms work |

Consequences you can exploit immediately:

- `stack<int, vector<int>>` is legal and slightly faster (`stack` only touches the back).
- `queue<int, list<int>>` is legal. `queue<int, vector<int>>` is **not** — no `pop_front()`.
- All three expose **no iterators**: `for (int x : st)` does not compile. Need to walk it? Wrong structure — or you want `make_heap` (§12).

Two facts that cost people their first submission: **`pop()` returns `void`** on all three (`int x = st.pop();` does not compile), and there is **no `clear()`/`erase()`** — empty an adaptor with `st = stack<int>();`.

*The adaptor design protects you: with no iterator, there is no way to invalidate one. The price is that "print the stack" needs a copy-and-drain helper.*

---

## 2. `stack` — LIFO

Last in, first out. The only element you may read is `top()`.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    stack<int> st;
    st.push(10); st.emplace(20); st.push(30);        // emplace constructs in place

    cout << st.size() << ' ' << st.top() << '\n';   // 3 30

    while (!st.empty()) {
        int x = st.top();                 // 1. READ
        st.pop();                         // 2. THEN POP (returns void)
        cout << x << ' ';
    }
    cout << '\n';                         // 30 20 10
    return 0;
}
```

| Member | Effect | Complexity |
|---|---|---|
| `push` / `emplace` | add on top | $\mathcal{O}(1)$ |
| `pop()` | remove top, **returns `void`** | $\mathcal{O}(1)$ |
| `top()` | reference to top | $\mathcal{O}(1)$ |
| `size()`, `empty()` | count, emptiness | $\mathcal{O}(1)$ |

**The underflow bug.** `top()` and `pop()` on an empty `stack` are **undefined behaviour**, not exceptions. You get garbage at best; at worst you corrupt memory and fail test 47 with nothing to point at.

```cpp
stack<int> st;
int x = st.top();   // UB: st is empty
st.pop();           // UB
```

Guard with `!st.empty()`, and compile with `-D_GLIBCXX_DEBUG` while developing (see [Topic 02](Topic%2002%20-%20Understanding%20Comments%2C%20Errors%2C%20Warnings.md)) — libstdc++ turns this into a readable assertion.

---

## 3. Stack Use Cases in CP

### 3.1 Balanced parentheses

Push every opener; on a closer the top must be its partner. Two failure modes: closing with an empty stack, and openers left over at the end. Check **both**.

```cpp
#include <bits/stdc++.h>
using namespace std;

bool balanced(const string& s) {
    stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else if (c == ')' || c == ']' || c == '}') {
            if (st.empty()) return false;            // nothing to close
            char open = st.top(); st.pop();
            if ((c == ')' && open != '(') ||
                (c == ']' && open != '[') ||
                (c == '}' && open != '{')) return false;
        }
    }
    return st.empty();                               // leftovers => unclosed
}

int main() {
    for (string t : {"()[]{}", "([{}])", "(]", "(()"}) cout << t << ' ' << balanced(t) << '\n';
    return 0;
}
```

```text
()[]{} 1
([{}]) 1
(] 0
(() 0
```

$\mathcal{O}(n)$ time. The stack is never inspected for its *values* beyond matching — the signature of a stack problem is that **only the top of history matters**.

### 3.2 Next greater element via a monotonic stack

For each index find the nearest $j>i$ with $a_j>a_i$. Brute force is $\mathcal{O}(n^2)$. The monotonic stack keeps **indices whose answer is still unknown**, with values strictly decreasing from bottom to top; a new larger value is the answer for everything it beats.

```cpp
vector<int> nextGreater(const vector<int>& a) {
    int n = (int)a.size();
    vector<int> res(n, -1);
    stack<int> st;                       // indices; values decrease bottom -> top
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && a[st.top()] < a[i]) {
            res[st.top()] = a[i];        // a[i] is the first greater to the right
            st.pop();
        }
        st.push(i);
    }
    return res;
}
// {2,1,2,4,3} -> {4,2,4,-1,-1}
```

**Trace** for `a = {2, 1, 2, 4, 3}`:

| `i` | `a[i]` | Pops (index → answer) | Stack after push | `res` |
|---|---|---|---|---|
| 0 | 2 | none (empty) | `[0]` | `- - - - -` |
| 1 | 1 | none (`2 < 1` false) | `[0,1]` | `- - - - -` |
| 2 | 2 | `1` → 2 | `[0,2]` | `- 2 - - -` |
| 3 | 4 | `2` → 4, `0` → 4 | `[3]` | `4 2 4 - -` |
| 4 | 3 | none (`4 < 3` false) | `[3,4]` | `4 2 4 - -` |
| end | — | `3`, `4` unresolved | `[3,4]` | `4 2 4 -1 -1` |

Each index is pushed once and popped at most once, so the inner `while` runs at most $n$ times *in total*. Total $\mathcal{O}(n)$.

Flip `<` to `<=` for greater-or-equal; reverse the loop for "previous greater"; flip the comparison for "next smaller". Those four variants are the whole family. *Setters exploit exactly this: one test file says "strictly greater", the next says "greater or equal", and one character decides your verdict.*

### 3.3 Evaluating a postfix expression

Postfix needs no precedence table: operands push, an operator pops two and pushes the result. **Order matters** — the first pop is the right operand.

```cpp
long long evalPostfix(const string& expr) {          // space-separated tokens
    stack<long long> st;
    stringstream ss(expr);
    string tok;
    while (ss >> tok) {
        if (tok == "+" || tok == "-" || tok == "*" || tok == "/") {
            long long b = st.top(); st.pop();        // right operand
            long long a = st.top(); st.pop();        // left operand
            if      (tok == "+") st.push(a + b);
            else if (tok == "-") st.push(a - b);
            else if (tok == "*") st.push(a * b);
            else                 st.push(a / b);
        } else {
            st.push(stoll(tok));
        }
    }
    return st.top();
}
// "3 4 + 2 *" = 14      "5 1 2 + 4 * + 3 -" = 14
```

Swap the two pops and subtraction/division turn to nonsense on the first non-commutative case — a bug that survives every test built only from `+` and `*`.

### 3.4 Reversing, undo, DFS, and killing recursion

- **Reversing.** Push everything, pop everything. Same cost as `std::reverse`, but natural when you are already streaming.
- **Undo history.** Push each state, `pop` to undo. "Undo the undo" needs a second stack (redo) — which is why every editor keeps two.
- **DFS with an explicit stack.** Mandatory when depth reaches $10^6$: the default stack is a few MB and each frame costs tens of bytes, so recursive DFS on a path graph segfaults before it finishes. **Any recursion can be converted** by pushing the *arguments* of pending calls — in reverse order, so the first one pops first.

---

## 4. `queue` — FIFO

First in, first out. You read `front()`, may inspect `back()`, and push at the back.

```cpp
queue<int> q;                                   // fragment
q.push(10); q.emplace(20); q.push(30);

cout << q.front() << ' ' << q.back() << '\n';   // 10 30
while (!q.empty()) {
    int x = q.front();                          // read
    q.pop();                                    // then pop (void)
    cout << x << ' ';                           // 10 20 30
}
```

Why `deque` and not `vector`? Because `queue` calls `pop_front()`, and removing from the front of a `vector` is $\mathcal{O}(n)$ — every element shifts. `deque` supports $\mathcal{O}(1)$ push and pop at **both** ends, exactly the interface a FIFO needs. ([Topic 12](Topic%2012%20-%20STL%20Part%201%20%28Vectors%2C%20DeQue%2C%20Lists%20and%20Forward%20Lists%29.md) explains `deque`'s chunked layout.)

| Member | Effect | Complexity |
|---|---|---|
| `push` / `emplace` | append at back | $\mathcal{O}(1)$ |
| `pop()` | remove from front, **returns `void`** | $\mathcal{O}(1)$ |
| `front()`, `back()` | first / last element | $\mathcal{O}(1)$ |

Same underflow rule: `front()`, `back()` and `pop()` on an empty `queue` are UB.

---

## 5. Queue Use Cases

### 5.1 BFS — the canonical frontier

BFS *is* "process nodes in discovery order", which is a FIFO. `queue<int>` (or `queue<pair<int,int>>`) is the canonical frontier, and no other structure preserves shortest-path-in-edges guarantees.

```cpp
#include <bits/stdc++.h>
using namespace std;

// '.' open, '#' blocked; returns minimum 4-directional steps, or -1
int bfsGrid(const vector<string>& g, int sr, int sc, int tr, int tc) {
    int n = (int)g.size(), m = (int)g[0].size();
    vector<vector<int>> dist(n, vector<int>(m, -1));
    queue<pair<int,int>> q;                       // the BFS frontier
    const int dr[] = {-1, 1, 0, 0}, dc[] = {0, 0, -1, 1};

    dist[sr][sc] = 0;
    q.push({sr, sc});
    while (!q.empty()) {
        auto [r, c] = q.front(); q.pop();
        if (r == tr && c == tc) return dist[r][c];
        for (int k = 0; k < 4; ++k) {
            int nr = r + dr[k], nc = c + dc[k];
            if (nr < 0 || nr >= n || nc < 0 || nc >= m) continue;
            if (g[nr][nc] == '#' || dist[nr][nc] != -1) continue;
            dist[nr][nc] = dist[r][c] + 1;        // mark on PUSH, not on pop
            q.push({nr, nc});
        }
    }
    return -1;
}

int main() {
    vector<string> g = {"....", ".##.", "....", "#..."};
    cout << bfsGrid(g, 0, 0, 3, 3) << '\n';       // 6
    return 0;
}
```

Two rules decide correctness: **mark visited when pushing** (marking late enqueues a cell many times and can give wrong distances), and for 0/1 weights use a `deque` — `push_front` for weight 0, `push_back` for weight 1 (0-1 BFS).

### 5.2 Level-order traversal

Snapshot `q.size()` **before** the inner loop: that number is the level boundary, and without it levels bleed together.

```cpp
int sz = (int)q.size();                  // exactly this level
for (int i = 0; i < sz; ++i) {
    TreeNode* cur = q.front(); q.pop();
    level.push_back(cur->val);
    if (cur->left)  q.push(cur->left);
    if (cur->right) q.push(cur->right);
}
```

### 5.3 Sliding windows, round-robin, Josephus

- **Sliding window over a stream:** keep a `queue` of the last $k$ items; push, then pop the front while `size() > k`.
- **Round-robin:** pop the front, serve it, or push it back with reduced need.
- **Josephus teaser:** pop $k-1$, push them back, eliminate the $k$-th — $\mathcal{O}(nk)$. The recurrence $J(n) = (J(n-1)+k) \bmod n$ is $\mathcal{O}(n)$; keep the simulation as a brute-force checker for small $n$.

---

## 6. `priority_queue` — Binary Heap

A `priority_queue` keeps the maximum on top, maintained as a binary heap inside a contiguous array.

```cpp
priority_queue<int> pq;                  // MAX-heap by default (fragment)
for (int x : {5, 1, 9, 3, 7}) pq.push(x);
pq.emplace(8);

cout << pq.size() << '\n';               // 6
while (!pq.empty()) {
    cout << pq.top() << ' ';             // 9 8 7 5 3 1
    pq.pop();
}
```

| Operation | Effect | Complexity |
|---|---|---|
| `push` / `emplace` | sift up | $\mathcal{O}(\log n)$ |
| `pop()` | remove top, sift down, returns `void` | $\mathcal{O}(\log n)$ |
| `top()` | reference to maximum | $\mathcal{O}(1)$ |

$n$ individual pushes cost $\mathcal{O}(n\log n)$; `make_heap` on a vector costs $\mathcal{O}(n)$ (§12) — measurable at $n=10^7$, irrelevant at $n=10^5$. No iterators, no `clear()`, and `top()` on an empty queue is UB.

---

## 7. The Three Ways to Make a Min-Heap

The default is a **max**-heap; most problems ("cheapest first", "nearest node") want the minimum.

**(a) The `greater` comparator — idiomatic.**

```cpp
priority_queue<int, vector<int>, greater<int>> minPQ;
// {5,1,9,3,7} -> 1 3 5 7 9
```

You must spell out `vector<int>`, because once you supply the third parameter you cannot skip the second.

**(b) Negation — no template gymnastics.**

```cpp
priority_queue<int> pq;                     // still a max-heap
for (int x : {5, 1, 9, 3, 7}) pq.push(-x);
// top() negated -> 1 3 5 7 9
```

Fine for signed `int`/`long long`; breaks for unsigned types, and for pairs you must negate every field and remember the sign on read. Prefer (a).

**(c) A custom comparator — required for structs.**

```cpp
struct Node { long long dist; int id; };
struct ByDistDesc {                                    // top = SMALLEST dist
    bool operator()(const Node& a, const Node& b) const {
        return a.dist > b.dist;                        // '>' for a min-heap
    }
};
priority_queue<Node, vector<Node>, ByDistDesc> pq;
```

---

## 8. Why `greater<int>` Gives a Min-Heap

This earns its own subsection because it is the most-confused line in the STL.

A heap has one invariant: **`top()` is the greatest element under the ordering `Compare` defines.** And `Compare(a, b) == true` means "$a$ ranks *behind* $b$" — the same convention `sort` uses, where `true` means "$a$ goes first from the left". The heap serves the element that is behind nothing.

| `Compare` | `Compare(a,b)` is `true` when | Front of the ordering | Heap |
|---|---|---|---|
| `less<int>` | $a < b$ | the largest | **max**-heap |
| `greater<int>` | $a > b$ | the smallest | **min**-heap |

So `greater<int>` says "larger values are lower priority": big things sink, the smallest rises. **The comparator defines priority, not order.** If the reasoning still slips, memorise the two lines every editorial quotes:

```cpp
priority_queue<int> pq;                             // max-heap
priority_queue<int, vector<int>, greater<int>> pq;  // min-heap
```

*Contest story: the wrong-direction heap causes more silently-wrong Dijkstra submissions than any other single line. A max-heap Dijkstra still terminates and still matches the sample — it just explores in the wrong order and, once you add a visited check, returns a non-optimal distance. Always test on a graph where the optimal path is not the first one you find.*

---

## 9. Custom Comparators and the Reversed-Sense Gotcha

The rule, once and precisely:

> Your comparator must return `true` when `a` has **lower priority** than `b` — when `b` should be popped first.

This is the *opposite* of `sort`'s convention, where `cmp(a,b) == true` means "$a$ comes first". For `priority_queue`, `cmp(a,b) == true` means "$a$ is dominated by $b$". Write `a.dist < b.dist` and you get a **max**-heap — which is the default, so the bug is invisible until you needed a min-heap.

```cpp
struct Node { long long dist; int id; };

// WRONG: true when a is "better" -> produces a MAX-heap on dist.
// struct Cmp { bool operator()(const Node& a, const Node& b) const { return a.dist < b.dist; } };

struct Cmp {                                  // RIGHT: true when a is LOWER priority
    bool operator()(const Node& a, const Node& b) const {
        if (a.dist != b.dist) return a.dist > b.dist;   // bigger dist = lower priority
        return a.id > b.id;                             // bigger id  = lower priority
    }
};

priority_queue<Node, vector<Node>, Cmp> pq;
for (Node x : {Node{10,1}, Node{2,7}, Node{2,3}, Node{99,0}}) pq.push(x);
while (!pq.empty()) { Node c = pq.top(); pq.pop(); cout << c.dist << ',' << c.id << ' '; }
```

```text
2,3 2,7 10,1 99,0
```

Two mechanical facts: the comparator must be a **callable type**, not a lambda — lambda types are not default-constructible in C++17, so use a `struct` with `bool operator()(...) const`. And it must be a **strict weak ordering** on priority: `Cmp(a,a)` must be `false`, so `<`/`>` never `<=`/`>=`, or the heap operations can crash exactly as `sort` does ([Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort%2C%20Binary%20Search%2C%20Min%20Max%2C%20Array%20Manipulation%2C%20Permutations%2C%20Math%20%26%20Accumulators%29.md)).

---

## 10. Pairs and Tuples in a Priority Queue

`pair` has lexicographic `operator<`: compare `.first`, break ties with `.second`.

```cpp
priority_queue<pair<int,int>> pq;   // max-heap, lexicographic
pq.push({1, 5}); pq.push({3, 2}); pq.push({3, 9}); pq.push({2, 100});
// pops: (3,9) (3,2) (2,100) (1,5)
```

That `(3,9)` before `(3,2)` is the lexicographic rule — usually what you want, because a well-chosen pair encodes a tie-break for free. Two idioms follow:

- **Min-heap of pairs:** `priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>>`. `greater` on a pair is also lexicographic, reversing both fields. This is the Dijkstra standard — `pair<dist,node>` with `greater<>` gives smallest distance first, smallest node id on ties.
- **`greater<>` with deduced arguments** (C++14 on): `priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;` — shorter, same speed.

Watch the ordering you inherit: `(value, index)` in a max-heap breaks ties by *largest index*; `(distance, node)` in a min-heap breaks ties by *smallest node*. `tuple` works identically, and `greater<tuple<...>>` is available for 3-key orderings.

---

## 11. No `decrease-key`: Lazy Deletion and the `set` Alternative

Textbook heaps support `decrease_key`, essential to the textbook $\mathcal{O}((V+E)\log V)$ Dijkstra. The STL `priority_queue` does not — you cannot find, let alone modify, an element inside the heap.

### Workaround 1: push a duplicate, skip stale entries

Push a fresh `(newDist, node)` and let the old entry rot. When you pop, check whether the distance still matches the best known; if not, discard. Cost: up to $\mathcal{O}(E)$ entries, so a $\log E$ factor — acceptable, since $E \le V^2$ gives $\log E \le 2\log V$.

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<long long> dijkstra(int n, const vector<vector<pair<int,int>>>& adj, int src) {
    const long long INF = LLONG_MAX / 4;
    vector<long long> dist(n, INF);
    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;

    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d != dist[u]) continue;              // STALE ENTRY -> lazy deletion

        for (auto [v, w] : adj[u]) {
            if (d + w < dist[v]) {
                dist[v] = d + w;
                pq.push({dist[v], v});           // no decrease-key: push again
            }
        }
    }
    return dist;
}

int main() {
    int n = 4;
    vector<vector<pair<int,int>>> adj(n);
    auto add = [&](int a, int b, int w) { adj[a].push_back({b,w}); adj[b].push_back({a,w}); };
    add(0, 1, 1); add(1, 2, 2); add(0, 2, 5); add(2, 3, 1);
    for (long long d : dijkstra(n, adj, 0)) cout << d << ' ';   // 0 1 3 4
    cout << '\n';
    return 0;
}
```

The `if (d != dist[u]) continue;` line is the whole trick. Without it distances stay correct (relaxation is idempotent) but you do wasted work — and with a `visited` array set on the *first* pop of a stale entry, the answer goes wrong.

### Workaround 2: `set<pair<dist, node>>` as a heap with erase

A `set` allows `erase(iterator)` and ordered access, so you can implement real decrease-key: erase `{oldDist, node}`, insert `{newDist, node}`, read `*begin()`. See [Topic 13](Topic%2013%20-%20STL%20Part%202%20%28Set%20%26%20Multiset%2C%20Map%20%26%20MultiMap%2C%20Unordered%20Set%20and%20Unordered%20Maps%29.md).

| Method | Typical use |
|---|---|
| `priority_queue<int>` + lazy deletion | default choice, dense graphs fine |
| `set<pair<ll,int>>` | exactly $V$ entries, real key updates |
| Hand-written indexed heap | $10^6$ nodes and you are squeezing |

---

## 12. Manual Heaps: `make_heap`, `push_heap`, `pop_heap`

Sometimes you want a heap *and* the ability to iterate the underlying array — for debugging, or after a bulk update. Drop to the raw algorithms on a `vector` (fragment; `v` is a `vector<int>`):

```cpp
make_heap(v.begin(), v.end());              // O(n): max now at v.front()
int mx = v.front();

v.push_back(10);
push_heap(v.begin(), v.end());              // O(log n) sift-up

pop_heap(v.begin(), v.end());               // O(log n): max moved to the BACK
int biggest = v.back();
v.pop_back();                               // only now is it removed

sort_heap(v.begin(), v.end());              // heap -> ascending; v is sorted
```

`push_heap` and `pop_heap` **assert** the rest is already a heap (`[b,e-1)` and `[b,e)` respectively) and repair it; `pop_heap` never shrinks the range, which is why `pop_heap` + `pop_back` is always a pair. Heap order is *not* sorted order — that is the point, and also the trap when you print the array. `make_heap` is $\mathcal{O}(n)$; `sort_heap` is $\mathcal{O}(n\log n)$; `is_heap` checks the invariant in $\mathcal{O}(n)$. Passing `greater<int>()` to all five turns the set into a min-heap.

---

## 13. Complexity Table and Alternatives

| Operation | `stack` | `queue` | `priority_queue` | `vector` | `set` |
|---|---|---|---|---|---|
| insert | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ | $\mathcal{O}(\log n)$ | $\mathcal{O}(1)^*$ back only | $\mathcal{O}(\log n)$ |
| erase "next" | $\mathcal{O}(1)$ top | $\mathcal{O}(1)$ front | $\mathcal{O}(\log n)$ top | $\mathcal{O}(1)$ back | $\mathcal{O}(\log n)$ |
| read next | $\mathcal{O}(1)$ `top` | $\mathcal{O}(1)$ `front` | $\mathcal{O}(1)$ `top` | $\mathcal{O}(1)$ | $\mathcal{O}(1)$ `*begin` |
| erase arbitrary | ✗ | ✗ | ✗ | $\mathcal{O}(n)$ | $\mathcal{O}(\log n)$ |
| iterate | ✗ | ✗ | ✗ | $\mathcal{O}(n)$ | $\mathcal{O}(n)$ |
| memory / element | ~1 word | ~1 word | ~1 word | 1 word | 3–4 words |

$^*$ amortised, with an $\mathcal{O}(n)$ worst case on reallocation.

Reading the table: the heap is strictly less capable than `set` and strictly cheaper in constants — `set` buys erase-by-value and ordered iteration for ~3× the memory and 2–4× the per-op time. At $2\times10^5$ operations that is noise; at $2\times10^7$ it is AC versus TLE.

---

## 14. Decision Table: When to Use What

| If the statement says… | Use | Why not the alternative |
|---|---|---|
| "most recent", "undo", "matching/nested", "previous greater" | `stack` | a queue gives the oldest — meaningless for nesting |
| "BFS", "unit weights", "level by level" | `queue` | a stack is DFS, wrong distances |
| both ends needed (0-1 BFS, sliding max) | `deque` | `queue` cannot pop the back |
| "always take the smallest/largest", greedy, Dijkstra | `priority_queue` | `set` is slower for the same job |
| erase an arbitrary element while popping max | `multiset` | `priority_queue` has no erase |
| fixed-size sliding window extremum | monotonic `deque` | a heap costs $\mathcal{O}(n\log n)$ |
| $k$-th largest in a stream | min-heap of size $k$ | sorting is $\mathcal{O}(n\log n)$ and unbounded memory |

`queue` vs `deque`: `queue` *restricts* `deque`. Use it to document intent and stop yourself touching the back; switch the moment you need `push_front`, `pop_back` or indexing. `priority_queue` vs `set`: need only "extract max"? The heap wins on constants. Need "remove *this* element" or in-order iteration? The `set` wins on capability — lazy deletion simulates erase for Dijkstra, not erase-by-value in the middle.

---

## 15. Recipe Book

### R1. Next greater element — `O(n)`

See §3.2. Variants: `>` → `>=` for greater-or-equal, right-to-left for "previous greater", `<` for "next smaller". Applications: daily temperatures, next warmer day.

### R2. Stock span — `O(n)`

Span of day $i$ = consecutive days up to $i$ with price $\le p_i$ = "previous strictly greater index", i.e. a monotonic stack.

```cpp
vector<int> stockSpan(const vector<int>& p) {
    int n = (int)p.size();
    vector<int> span(n, 1);
    stack<int> st;                                    // indices, prices decreasing
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && p[st.top()] <= p[i]) st.pop();
        span[i] = st.empty() ? i + 1 : i - st.top();
        st.push(i);
    }
    return span;
}
// {100,80,60,70,60,75,85} -> {1,1,1,2,1,4,6}
```

### R3. Largest rectangle in a histogram — `O(n)`

For each bar, the widest rectangle using it as the minimum extends to the first strictly smaller bar on each side. A monotonic stack finds both boundaries in one pass; the sentinel `0` at the end flushes the stack.

```cpp
long long largestRectangle(const vector<int>& h) {
    int n = (int)h.size();
    long long best = 0;
    stack<int> st;                                    // indices, heights increasing
    for (int i = 0; i <= n; ++i) {
        long long cur = (i == n ? 0 : h[i]);          // sentinel flushes
        while (!st.empty() && h[st.top()] > cur) {
            long long height = h[st.top()];
            st.pop();
            int left = st.empty() ? 0 : st.top() + 1;
            best = max(best, height * (i - left));
        }
        if (i < n) st.push(i);
    }
    return best;
}
// {2,1,5,6,2,3} -> 10
```

Memorise this. The 2D version ("maximal rectangle in a binary matrix") is just this function called once per row after converting each row to a histogram.

### R4. Sliding window maximum with a monotonic `deque` — `O(n)`

The deque holds **indices** with strictly decreasing values. The front is the window maximum; the back is trimmed whenever a newer, larger value makes those indices useless forever.

```cpp
vector<int> slidingMax(const vector<int>& a, int k) {
    deque<int> dq;                                    // indices, values decreasing
    vector<int> res;
    for (int i = 0; i < (int)a.size(); ++i) {
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();  // expire
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();  // dominate
        dq.push_back(i);
        if (i >= k - 1) res.push_back(a[dq.front()]);
    }
    return res;
}
// {1,3,-1,-3,5,3,6,7}, k=3 -> {3,3,5,5,6,7}
```

Each index enters and leaves the deque once, so $\mathcal{O}(n)$. A heap with lazy deletion also works at $\mathcal{O}(n\log n)$ — correct, but the deque is what survives a $10^6$ limit with room to spare.

### R5. Top-$k$ frequent elements — `O(n log k)`

Count with a hash map, then keep a **min-heap of size $k$** keyed by frequency; anything below the current $k$-th best is evicted at once.

```cpp
vector<int> topKFrequent(const vector<int>& a, int k) {
    unordered_map<int,int> freq;
    for (int x : a) ++freq[x];

    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;  // (freq, value)
    for (auto& [val, f] : freq) {
        pq.push({f, val});
        if ((int)pq.size() > k) pq.pop();             // drop least frequent
    }
    vector<int> out;
    while (!pq.empty()) { out.push_back(pq.top().second); pq.pop(); }
    reverse(out.begin(), out.end());                  // most frequent first
    return out;
}
```

### R6. Merge $k$ sorted lists — `O(N log k)`

```cpp
vector<int> mergeK(const vector<vector<int>>& lists) {
    using T = tuple<int,int,int>;                     // value, list, position
    priority_queue<T, vector<T>, greater<T>> pq;
    for (int i = 0; i < (int)lists.size(); ++i)
        if (!lists[i].empty()) pq.push({lists[i][0], i, 0});

    vector<int> out;
    while (!pq.empty()) {
        auto [val, i, j] = pq.top(); pq.pop();
        out.push_back(val);
        if (j + 1 < (int)lists[i].size()) pq.push({lists[i][j + 1], i, j + 1});
    }
    return out;
}
```

`greater<tuple<...>>` is lexicographic — "smallest value first, arbitrary tie-break". The same skeleton solves k-way stream merging.

### R7. Task scheduling with cooldown — heap + waiting queue

Always run the most frequent available task; a max-heap of counts plus a queue of cooling-down tasks is the standard shape. The pattern reappears in seat reservation, meeting rooms and replenishment problems.

```cpp
int leastInterval(vector<int> counts, int cooldown) {
    priority_queue<int> pq(counts.begin(), counts.end());
    queue<pair<int,int>> cooling;                     // (remaining count, ready time)
    int time = 0;
    while (!pq.empty() || !cooling.empty()) {
        ++time;
        if (!pq.empty()) { int c = pq.top() - 1; pq.pop(); if (c > 0) cooling.push({c, time + cooldown}); }
        if (!cooling.empty() && cooling.front().second == time) { pq.push(cooling.front().first); cooling.pop(); }
    }
    return time;
}
// {3,3,2}, cooldown 2 -> 8
```

### R8. Dijkstra with lazy deletion — `O((V+E) log E)`

Written out in §11. The three lines worth memorising:

```cpp
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d != dist[u]) continue;                       // skip stale
    for (auto [v, w] : adj[u])
        if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
}
```

Graphs get a full chapter later; this belongs here because Dijkstra is the single biggest consumer of `priority_queue`, and lazy deletion is a heap technique, not a graph technique.

---

## Key Takeaways

- `stack`, `queue`, `priority_queue` are **adaptors**: they own a `Container` and hide its interface. Parameters are `(T, Container)` and `(T, Container, Compare)`.
- `pop()` returns `void`, and `top()`/`front()`/`pop()` on an empty adaptor is **undefined behaviour**, not an exception. Read the value, then pop, and guard with `empty()`.
- `queue<int>` is the canonical BFS frontier; mark visited **on push**, not on pop.
- A monotonic stack solves next-greater/smaller, stock span and histogram rectangles in $\mathcal{O}(n)$ by storing *indices* and popping what the new element beats.
- `priority_queue<int>` is a **max**-heap; `greater<int>` as the third parameter makes it a min-heap, because the comparator defines *priority*: `Compare(a,b) == true` means $a$ ranks behind $b$.
- Your PQ comparator must return `true` when `a` has **lower** priority — inverted relative to `sort`, and the top source of wrong Dijkstra answers.
- No `decrease-key` exists: push a duplicate and skip stale entries, or use `set<pair<dist,id>>`.
- `pop_heap` moves the top to the back without shrinking the range: always pair it with `pop_back`.
- Choose `priority_queue` over `set` on constants only when $n \ge 10^7$; otherwise choose on capability.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `int x = st.pop();` | `pop()` returns `void` | `int x = st.top(); st.pop();` |
| 2 | Popping an empty adaptor | UB: garbage or memory corruption, often no crash | `while (!st.empty())` |
| 3 | `queue<int, vector<int>>` | `vector` has no `pop_front()` | keep `deque`, or use `deque` directly |
| 4 | Assuming `priority_queue` is a min-heap | It is a max-heap; the smallest comes out last | add `greater<int>` |
| 5 | PQ comparator written `a.dist < b.dist` for a min-heap | Produces a max-heap; Dijkstra explores worst-first | return `a.dist > b.dist` |
| 6 | Lambda as PQ comparator (C++17) | Lambda types are not default-constructible | `struct { bool operator()(...) const; }` |
| 7 | Comparator uses `<=` | Breaks strict weak ordering; may crash | strict `<` / `>` only |
| 8 | Marking BFS visited on pop | Node enqueued repeatedly; TLE or wrong distance | mark on push |
| 9 | Level-order without `q.size()` snapshot | Levels merge; wrong answer on any multi-level tree | capture `sz` first |
| 10 | Monotonic stack storing values, not indices | Cannot compute distances or spans | store indices |
| 11 | `<=` where `<` was needed in a monotonic stack | Duplicates collapse wrongly; off-by-one spans | decide strictness from the statement |
| 12 | `pop_heap` alone "removes" the max | It only swaps to the back | `pop_heap(...); v.pop_back();` |
| 13 | Dijkstra without the stale check plus a `visited` array | Non-optimal distances, no crash | `if (d != dist[u]) continue;` |
| 14 | Trusting a heap to preserve insertion order on ties | Order among equals is unspecified | encode the tie-break in the comparator |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Validate three bracket types | CSES / Codeforces Div. 4 A | Easy |
| 2 | Next greater element, all four variants | AtCoder ABC | Easy |
| 3 | Stock span | CSES-style classic | Easy |
| 4 | Evaluate a postfix expression | CSES parsing style | Easy |
| 5 | Grid shortest path with obstacles (BFS) | CSES "Labyrinth" | Easy–Medium |
| 6 | Largest rectangle in a histogram | CSES "Advertisement" | Medium |
| 7 | Maximal rectangle in a binary matrix | Codeforces Div. 2 D | Medium |
| 8 | Sliding window maximum | CSES "Sliding Window Maximum" | Medium |
| 9 | Top-$k$ frequent elements | Codeforces Div. 3 D | Medium |
| 10 | Merge $k$ sorted lists | AtCoder ABC | Medium |
| 11 | Task scheduling with cooldown | Codeforces Div. 2 C | Medium |
| 12 | Dijkstra on a sparse graph | CSES "Shortest Routes I" | Medium |
| 13 | Josephus, $n \le 10^5$ | CSES "Josephus Problem I" | Medium |
| 14 | 0-1 BFS with `deque` | Codeforces Div. 2 D | Medium–Hard |

## Next Up

→ [Topic 15 — STL Part 4: Essential Prebuilt Algorithms](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort%2C%20Binary%20Search%2C%20Min%20Max%2C%20Array%20Manipulation%2C%20Permutations%2C%20Math%20%26%20Accumulators%29.md)

You now have every container you routinely need. The last STL topic turns them into algorithms: sorting with custom comparators, the `lower_bound` family (including binary search on the answer), the array helpers, permutations, and the `<numeric>` accumulators. It is the highest-leverage file in this chapter — read it before your next contest, not after.