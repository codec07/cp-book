# Topic 03 — Pairs, Tuples and Simulation

Two things make up the cheapest problems in every contest: grouping values together, and implementing exactly what the statement says. The first is `pair` and `tuple` — small, fast, and sortable, which makes them the standard tool for "keep the value and its index" and "sort by two keys". The second is **simulation**, where there is no clever trick and the entire challenge is care: the right state, the right transition, the right termination condition, and no off-by-one at the boundary.

This topic ends with five complete simulation programs, because simulation is a skill you acquire by reading finished, careful code — not by reading a description of it.

> **Prerequisites:** [Topic 02 — Time Complexity](Topic%2002%20-%20Time%20Complexity%20(Basics%20to%20Advanced).md), [Chapter 1, Topic 09 — Arrays](../CPP%20for%20CP/Topic%2009%20-%20Arrays%20(1D,%202D%20and%203D).md), [Chapter 1, Topic 15 — Algorithms](../CPP%20for%20CP/Topic%2015%20-%20STL%20Part%204%20(Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators).md)
> **You will be able to:**
> - Use `pair` and `tuple` with their comparison rules, for sorting by multiple keys.
> - Build composite keys for `set` and `map` without writing a comparator.
> - Turn a problem statement into a (state, transition, clock, termination) quadruple.
> - Choose between time-step and event-driven simulation.
> - Write cycle detection into a simulation so it always terminates.
> - Recognise the four mistakes that break almost every simulation.

---

## 1. Why `pair` exists

A function returns one value. But half the time you need two: a distance and the node it belongs to, a value and its original index, a coordinate and a direction. `pair` is the smallest possible container for two values of different types.

```cpp
pair<int, string> p = {42, "answer"};
cout << p.first << ' ' << p.second << '\n';    // 42 answer
```

The type is templated, so the two halves can be anything — `int`, `double`, `string`, even another `pair`. There is no overhead beyond the two members: `sizeof(pair<int,int>) == 8` on a typical 64-bit build.

| Form | Meaning |
|---|---|
| `pair<int,int> p;` | default-initialised (garbage for `int` at block scope) |
| `pair<int,int> p = {1, 2};` | brace initialisation |
| `make_pair(1, 2)` | deduce the types |
| `p.first`, `p.second` | access |
| `v.push_back({x, y})` | append a pair to a vector |

`make_pair` deduces types, which is occasionally wrong:

```cpp
auto a = make_pair(1, 2.5);        // pair<int, double>
pair<int,int> b = make_pair(1, 2.5); // 2.5 truncated to 2 — silent
```

Prefer brace initialisation `{1, 2}` inside a context where the target type is known; it converts safely and fails loudly.

---

## 2. `pair` comparison — the reason it is everywhere

`pair` has a built-in lexicographic `operator<`: compare `.first`; if equal, compare `.second`.

```cpp
pair<int,int> a = {2, 5}, b = {2, 3};
cout << (b < a) << '\n';       // 1 — same first, 3 < 5
```

That single fact does an enormous amount of work. Two consequences:

**Sorting gives you a two-key sort for free.**

```cpp
vector<pair<int,int>> vp = {{2, 5}, {1, 9}, {2, 3}, {1, 2}};
sort(vp.begin(), vp.end());
// (1,2) (1,9) (2,3) (2,5)
```

No comparator needed: first ascending, then second ascending.

**The `{value, index}` idiom.** Push the value alongside its original position, sort, and you recover both the order and the provenance:

```cpp
vector<pair<int,int>> v;
for (int i = 0; i < n; i++) v.push_back({a[i], i});
sort(v.begin(), v.end());
// v[k].first = k-th smallest value, v[k].second = where it came from
```

This is the standard answer to "print the original indices in sorted order" and it is cheaper to write than an index array with a custom comparator.

**Descending order.** Two ways:

```cpp
sort(vp.rbegin(), vp.rend());                                  // entire pair reversed
sort(vp.begin(), vp.end(), greater<pair<int,int>>());          // same thing
sort(vp.begin(), vp.end(), [](auto &x, auto &y){ return x.first > y.first; });  // first only
```

Note the difference: the first two reverse **both** keys, the third reverses only `.first`. Problems that say "sort by value descending, ties by index ascending" need the third form.

---

## 3. `tuple` — three or more values

Once you need a third value, `pair` nests awkwardly. `tuple` is the generalisation.

```cpp
tuple<int, string, double> t = {1, "abc", 2.5};

cout << get<0>(t) << '\n';            // 1
cout << get<1>(t) << '\n';            // abc

get<2>(t) = 3.5;                       // assign a member
```

`get<>` takes the index as a **template argument**, so it must be a compile-time constant. You cannot write `get<i>(t)` with a loop variable.

### Structured bindings (C++17)

The readable form — and the one you should use:

```cpp
auto [id, name, score] = t;
cout << id << ' ' << name << ' ' << score << '\n';
```

You can also bind by reference to modify in place:

```cpp
auto &[id, name, score] = t;
score = 99;
```

### `tie` — unpacking and comparing

```cpp
tuple<int,int,int> t = {1, 2, 3};
int a, b, c;
tie(a, b, c) = t;                     // unpack
```

The more useful trick is building a comparison key on the fly:

```cpp
// compare (a,b,c) against (d,e,f) lexicographically, no tuple object needed
if (tie(a, b, c) < tie(d, e, f)) { /* ... */ }
```

This replaces a three-branch comparator with one line. Use `tie` in a comparator when the keys are already separate variables.

`ignore` lets you skip a position:

```cpp
tie(a, ignore, c) = t;                // take positions 0 and 2
```

### Tuple comparison

`tuple` also has lexicographic `operator<`, extending to any arity:

```cpp
tuple<int,int,int> x = {1, 2, 9}, y = {1, 2, 10};
cout << (x < y) << '\n';              // 1
```

| | `pair` | `tuple` |
|---|---|---|
| Members | Exactly 2 | Any arity |
| Access | `.first`, `.second` | `get<i>(t)` |
| Structured binding | `auto [a,b] = p;` | `auto [a,b,c] = t;` |
| Comparison | lexicographic | lexicographic |
| Overhead | none | none |
| Typical use | value + index | 3+ key sort, geometry |

---

## 4. Composite keys in `set` and `map`

Because `pair` and `tuple` have `operator<`, they work directly as keys:

```cpp
set<pair<int,int>> visited;                 // grid coordinates
visited.insert({r, c});
if (visited.count({r, c})) { /* already seen */ }

map<pair<int,int>, int> dist;               // distance per cell
dist[{0, 0}] = 0;

set<tuple<int,int,int>> states;             // 3D state
```

This is the standard way to key a BFS on a grid or a DP on a multi-dimensional state, without writing a hash or a comparator. The ordering is lexicographic, which is exactly what you want.

| Need | Structure |
|---|---|
| Two-coordinate key, ordered | `set<pair<int,int>>` |
| Three-plus coordinates | `set<tuple<...>>` |
| Same, but faster lookups | `set<pair<int,int>>` is fine; `unordered_set` needs a custom hash |
| Key plus payload | `map<pair<int,int>, int>` |

> **Warning:** `pair<int,int>` as a key in `unordered_map` does not compile without a hash function. You would need a custom `hash<pair<int,int>>`. For grid problems, either use a `set` or encode the pair as a single integer `r * m + c` — the second is faster and simpler. See §10.4.

---

## 5. What simulation actually means

A simulation problem gives you a system, an initial state, a rule for how the state changes, and asks what happens after $T$ steps — or whether some condition is ever reached.

There is no algorithm to discover. The entire task is transcription, and the entire difficulty is that transcription is easy to get subtly wrong.

**Every simulation has exactly four parts. Identify all four before you code:**

| Part | Question to answer | Where it goes wrong |
|---|---|---|
| **State** | What is the complete configuration at one instant? | Missing a variable (direction, velocity, remaining time) |
| **Transition** | Given the state, what is the next state? | Order of updates; using an already-updated value |
| **Clock** | Is time discrete (ticks) or event-based? | Off-by-one in the number of steps |
| **Termination** | When do you stop? | No bound, so an infinite loop |

If you cannot state all four in one sentence each, you do not yet understand the problem — and coding will not fix that.

### The state discipline

The state is the **complete** information needed to compute the future. Nothing more, nothing less.

```cpp
// Robot on a grid: position AND direction are both needed.
int r, c, dir;
// Without dir, the next 'F' command is undefined.

// Round-robin scheduler: remaining time per process AND the queue order.
vector<long long> rem;
queue<int> q;
```

Test: *given the state, could two different histories lead to the same state but different futures?* If yes, your state is missing something.

---

## 6. Time-step versus event-driven

Two ways to advance a simulation.

### Time-step (tick-based)

Advance one unit of time at a time and recompute.

```cpp
for (int t = 0; t < T; t++) {
    apply_rules();
}
```

Cost: $\mathcal{O}(T \times \text{cost per step})$.

Use when $T$ is small, or when the system changes every tick anyway. This is what a cellular automaton needs.

**The cost blows up when $T$ is large.** If $T = 10^{18}$, you cannot step. You need either a formula, matrix exponentiation, or cycle detection.

### Event-driven

Jump to the next moment something changes, skipping idle time.

```cpp
while (!events.empty()) {
    auto [time, type, id] = events.top();   // min-heap by time
    events.pop();
    advance_clock_to(time);
    process(type, id);
    schedule_followups();
}
```

Cost: $\mathcal{O}(E \log E)$ for $E$ events. This is how priority-queue simulations work (queueing systems, Dijkstra, task scheduling with releases).

| | Time-step | Event-driven |
|---|---|---|
| Cost | $T \times$ step cost | $E \log E$ |
| Good when | $T$ small, every tick matters | $T$ huge, events are sparse |
| Data structure | None | Priority queue |
| Typical problems | Game of Life, falling sand | Scheduling, queueing, shortest paths |

**The decision rule:** if $T$ is up to $10^7$ and the step is cheap, time-step. If $T$ is up to $10^{18}$ or events are sparse, event-driven. If $T$ is huge *and* the state is finite, use cycle detection.

---

## 7. Termination: the part everyone forgets

A simulation must stop. Three mechanisms, in order of preference:

1. **A known bound.** `for (t = 0; t < T; t++)` — always terminates, and the complexity is obvious.
2. **A natural end condition.** The queue empties, the pointer leaves the grid, the process finishes. Terminates if the state strictly progresses.
3. **Cycle detection.** Track visited states; if you revisit one, the answer repeats forever.

Mechanism 3 is essential whenever the state space is finite but the time bound is not given. A finite state space plus a deterministic transition means the trajectory is eventually periodic — so `visited` finds the cycle.

```cpp
set<State> seen;
State s = initial();
while (!seen.count(s)) {
    seen.insert(s);
    s = next(s);
    if (is_answer(s)) { /* found */ }
}
// s repeats a previous state; anything not found will never be found
```

This is the same structure as Floyd's tortoise-and-hare, but with a set instead of two pointers — $\mathcal{O}(\text{states})$ memory instead of $\mathcal{O}(1)$.

> *Contest story: a problem asked for the number of steps until a process repeats, with the process applied up to $10^9$ times. Half the submissions looped $10^9$ times; the intended solution used a `set` of seen states, found the cycle in under $10^5$ steps, and computed the answer with modular arithmetic on the cycle length. Same simulation, different termination logic.*

---

## 8. Solved program 1 — Robot on a grid

**Problem.** An $n \times m$ grid contains `.` (free) and `#` (wall). A robot starts at $(sr, sc)$ facing one of four directions. It receives a command string of `L` (turn left 90°), `R` (turn right 90°), and `F` (move one cell forward if the target is in bounds and not a wall). Output the final position and direction.

**State:** row, column, direction index.
**Transition:** rotate or move, depending on the command.
**Clock:** one command per step, bounded by the string length.
**Termination:** the string ends.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, m;
    cin >> n >> m;

    vector<string> g(n);
    for (auto &row : g) cin >> row;

    int sr, sc, d;
    cin >> sr >> sc >> d;        // 0 = north, 1 = east, 2 = south, 3 = west

    string cmds;
    cin >> cmds;

    // direction vectors, ordered N, E, S, W
    const int dr[4] = {-1, 0, 1, 0};
    const int dc[4] = {0, 1, 0, -1};

    int r = sr, c = sc;

    for (char cmd : cmds) {
        if (cmd == 'L') {
            d = (d + 3) % 4;                 // -1 mod 4, written positively
        } else if (cmd == 'R') {
            d = (d + 1) % 4;
        } else {                              // 'F'
            int nr = r + dr[d], nc = c + dc[d];
            if (nr >= 0 && nr < n && nc >= 0 && nc < m && g[nr][nc] != '#') {
                r = nr;
                c = nc;
            }
            // otherwise: blocked, the command is consumed and nothing happens
        }
    }

    cout << r << ' ' << c << ' ' << d << '\n';
    return 0;
}
```

**Input**
```text
3 4
....
.#..
....
2 0 1
FRFLFF
```

**Output**
```text
2 3 1
```

**The four things that go wrong here:**

- `d = (d - 1) % 4` gives `-1` in C++, not `3`. Use `(d + 3) % 4`.
- Forgetting to check `g[nr][nc] != '#'` — the robot walks through walls.
- Reflecting instead of blocking: `F` into a wall does **not** turn the robot around. Read the statement; these two behaviours appear in different problems.
- Moving then checking bounds instead of checking first — an out-of-bounds write, not just a wrong answer.

---

## 9. Solved program 2 — Conway's Game of Life, $k$ steps

**Problem.** An $n \times m$ binary grid. Each step, every cell simultaneously updates: a live cell with 2 or 3 live neighbours survives; a dead cell with exactly 3 live neighbours becomes alive; everything else dies. Output the grid after $k$ steps.

**State:** the whole grid.
**Transition:** count 8-neighbours for every cell, apply the rules **simultaneously**.
**Clock:** $k$ discrete steps.
**Termination:** fixed bound $k$.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, m, k;
    cin >> n >> m >> k;

    vector<string> cur(n);
    for (auto &row : cur) cin >> row;

    const int dr[8] = {-1, -1, -1, 0, 0, 1, 1, 1};
    const int dc[8] = {-1, 0, 1, -1, 1, -1, 0, 1};

    for (int step = 0; step < k; step++) {
        vector<string> nxt = cur;              // copy: unchanged cells stay as they are

        for (int r = 0; r < n; r++) {
            for (int c = 0; c < m; c++) {
                int live = 0;
                for (int d = 0; d < 8; d++) {
                    int nr = r + dr[d], nc = c + dc[d];
                    if (nr < 0 || nr >= n || nc < 0 || nc >= m) continue;
                    live += (cur[nr][nc] == '1');
                }

                if (cur[r][c] == '1')
                    nxt[r][c] = (live == 2 || live == 3) ? '1' : '0';
                else
                    nxt[r][c] = (live == 3) ? '1' : '0';
            }
        }
        cur.swap(nxt);                          // O(1): swap the buffers, do not copy back
    }

    for (const auto &row : cur) cout << row << '\n';
    return 0;
}
```

**Input**
```text
3 3 1
010
010
010
```

**Output**
```text
000
111
000
```

**Two lessons.**

The **double-buffer** is mandatory. If you write into `cur` while reading from it, later cells see already-updated neighbours — a different (and wrong) automaton. `nxt = cur` copies the grid, then you overwrite every cell, so the copy is only there for clarity; a plain `vector<string> nxt(n, string(m, '0'));` is equivalent.

`cur.swap(nxt)` is $\mathcal{O}(1)$ — it swaps three pointers. `cur = nxt` copies $nm$ characters every step.

**Complexity:** $\mathcal{O}(k \cdot n \cdot m)$ with a constant of 8. At $k = 100$, $n = m = 100$, that is 8 million operations — comfortable.

---

## 10. Solved program 3 — The Josephus problem

**Problem.** $n$ people stand in a circle numbered $1..n$. Starting from person 1, count $k$ people, eliminate the $k$-th, then continue counting from the next person. Output the elimination order.

**State:** the circle and the current counting position.
**Transition:** skip $k-1$ people, eliminate the next.
**Clock:** one elimination per step.
**Termination:** the circle empties.

A queue models the circle exactly: the front is the current person, and rotating moves the front to the back.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, k;
    cin >> n >> k;

    queue<int> q;
    for (int i = 1; i <= n; i++) q.push(i);

    bool first = true;
    while (!q.empty()) {
        // skip k-1 people: move them to the back
        for (int i = 0; i < k - 1; i++) {
            q.push(q.front());
            q.pop();
        }
        if (!first) cout << ' ';
        first = false;
        cout << q.front();
        q.pop();                       // eliminate
    }
    cout << '\n';
    return 0;
}
```

**Input**
```text
7 3
```

**Output**
```text
3 6 2 7 5 1 4
```

**Complexity:** $\mathcal{O}(nk)$. At $n = 10^5$, $k = 10^5$ that is $10^{10}$ — TLE.

**The optimisation.** If you only need the *last survivor* (not the order), there is a closed form:

$$J(1) = 0, \qquad J(n) = (J(n-1) + k) \bmod n$$

with 1-indexed indices mapped to 0-indexed internally:

```cpp
int survivor = 0;                       // 0-indexed survivor for one person
for (int i = 2; i <= n; i++)
    survivor = (survivor + k) % i;
cout << survivor + 1 << '\n';           // convert back to 1-indexed
```

$\mathcal{O}(n)$ time, $\mathcal{O}(1)$ space. This is a classic example of a simulation problem where the simulation is not the intended solution — always check whether the question asks for the whole process or just the endpoint.

---

## 11. Solved program 4 — Round-robin scheduling

**Problem.** $n$ processes each need a known amount of CPU time. A scheduler gives each process a quantum of $q$ time units in arrival order, cycling back to the end of the queue if unfinished. Compute the completion time of each process.

**State:** remaining time per process, plus the queue order.
**Transition:** run the front for `min(q, remaining)`, then either finish or requeue.
**Clock:** event-driven — time jumps by the amount actually run.
**Termination:** the queue empties.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n;
    long long q;
    cin >> n >> q;

    vector<long long> rem(n), done(n, -1);
    for (int i = 0; i < n; i++) cin >> rem[i];

    queue<int> ready;
    for (int i = 0; i < n; i++) ready.push(i);

    long long clock = 0;

    while (!ready.empty()) {
        int p = ready.front();
        ready.pop();

        long long run = min(q, rem[p]);
        clock += run;
        rem[p] -= run;

        if (rem[p] > 0) ready.push(p);      // still needs time: back of the queue
        else            done[p] = clock;    // finished at this instant
    }

    for (int i = 0; i < n; i++) cout << done[i] << " \n"[i == n - 1];
    return 0;
}
```

**Input**
```text
4 3
5 3 8 2
```

**Output**
```text
13 6 18 11
```

**Trace, because this is where the off-by-one lives:**

| Step | Process | Run | Clock after | Remaining after | Event |
|---|---|---|---|---|---|
| 1 | 0 | 3 | 3 | 2 | requeued |
| 2 | 1 | 3 | 6 | 0 | **done at 6** |
| 3 | 2 | 3 | 9 | 5 | requeued |
| 4 | 3 | 2 | 11 | 0 | **done at 11** |
| 5 | 0 | 2 | 13 | 0 | **done at 13** |
| 6 | 2 | 3 | 16 | 2 | requeued |
| 7 | 2 | 2 | 18 | 0 | **done at 18** |

So the completion times are process 0 → 13, 1 → 6, 2 → 18, 3 → 11.

The definitional question is whether completion time is the clock *after* the final quantum (as here) or the clock *before* it — some statements ask for the time the process started its last slice. Both conventions appear in real problems; the trace table is how you tell which one you are implementing.

**Complexity:** $\mathcal{O}(n + \text{total time} / q)$ iterations. The number of queue operations is at most $n + \sum \lceil \text{rem}_i / q \rceil$.

---

## 12. Solved program 5 — Cycle detection in a state space

**Problem.** Starting from $n$, replace it with the sum of the squares of its digits, and repeat. If the process reaches 1, output the number of steps. If it repeats a value it has seen before (and never reaches 1), output $-1$.

This is the "happy number" process. It is a simulation whose time bound is **not given** — so termination must come from cycle detection.

**State:** the current integer.
**Transition:** digit-square sum.
**Clock:** one application per step.
**Termination:** reach 1, or revisit a state.

```cpp
#include <bits/stdc++.h>
using namespace std;

long long digitSquareSum(long long x) {
    long long s = 0;
    while (x > 0) {
        long long d = x % 10;
        s += d * d;
        x /= 10;
    }
    return s;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    long long n;
    cin >> n;

    unordered_set<long long> seen;
    long long steps = 0;

    while (n != 1 && !seen.count(n)) {
        seen.insert(n);
        n = digitSquareSum(n);
        steps++;
    }

    cout << (n == 1 ? to_string(steps) : "-1") << '\n';
    return 0;
}
```

**Input**
```text
19
```

**Output**
```text
4
```

Trace: $19 \to 1^2 + 9^2 = 82 \to 8^2 + 2^2 = 68 \to 6^2 + 8^2 = 100 \to 1^2 + 0 + 0 = 1$. That is four transitions, and the four states $82, 68, 100$ are all distinct, so the loop runs four times and exits on `n == 1`.

**Why it terminates:** the process maps integers to smaller integers once $x \ge 100$, so the reachable set is finite. A finite deterministic trajectory must eventually repeat. The `seen` set finds the repeat in $\mathcal{O}(\text{states})$ steps.

**Complexity:** $\mathcal{O}(S \log S)$ where $S$ is the number of distinct states reached before a repeat, because each `unordered_set` operation is expected $\mathcal{O}(1)$ and each digit-square sum is $\mathcal{O}(\log n)$.

**The general pattern**, applicable to any deterministic finite-state process:

```cpp
set<State> seen;
State s = initial;
while (!is_goal(s) && !seen.count(s)) {
    seen.insert(s);
    s = transition(s);
}
// if is_goal(s): found in some number of steps
// else: entered a cycle that never reaches the goal
```

| Variant | Extra state | Use when |
|---|---|---|
| `set` of states | memory $\mathcal{O}(S)$ | You need the step count or the cycle start |
| Floyd's tortoise-hare | $\mathcal{O}(1)$ | You only need "does it cycle" |
| `map<State,int>` | index of first visit | You need the cycle length and entry point |

---

## 13. The four mistakes that break simulations

### 13.1 Updating in place when you need a snapshot

Automata and anything with "simultaneously" require two buffers. Writing into `cur[r][c]` while later cells read their neighbours gives a different process.

**Fix:** always allocate `nxt` and swap at the end. The copy costs $\mathcal{O}(nm)$ per step and is never the bottleneck.

### 13.2 The boundary is inclusive or exclusive, and you guessed

Grids are `0..n-1`. Walls reflect or block — opposite behaviours in different problems. "Count $k$ people" eliminates the $k$-th, so you skip $k-1$.

**Fix:** hand-trace a $2 \times 2$ or $n = 5$ case on paper before coding. If the trace matches the sample, the boundary convention is right.

### 13.3 The clock advances at the wrong moment

Does time increment before or after the action? Does a process that completes at $t = 6$ have completion time 6 or 9? Is the step count the number of transitions or the number of states visited?

**Fix:** write the trace table. There is no other method. Every one of these is a question about a definition, and the statement always answers it if you read the output specification closely.

### 13.4 No termination condition

`while (true)` with no `break`, or a loop bounded by something that never becomes false.

**Fix:** every simulation needs one of the three termination mechanisms from §7. If the time bound is absent from the statement, you need cycle detection — and you should add it even when you think the loop terminates, as a safety net.

---

## 14. Complexity of simulation

| Shape | Complexity | Feasible when |
|---|---|---|
| $T$ steps, $\mathcal{O}(1)$ each | $\mathcal{O}(T)$ | $T \le 10^7$ |
| $T$ steps on an $n \times m$ grid | $\mathcal{O}(Tnm)$ | $Tnm \le 10^7$ |
| $T$ steps, 8-neighbour scan | $\mathcal{O}(8Tnm)$ | $Tnm \le 10^6$ |
| $E$ events with a heap | $\mathcal{O}(E \log E)$ | $E \le 10^6$ |
| Round-robin | $\mathcal{O}(n + \sum \lceil r_i/q \rceil)$ | Total time / $q \le 10^7$ |
| Cycle detection | $\mathcal{O}(S \log S)$ | $S \le 10^6$ |

The common TLE in simulation problems is not a wrong algorithm — it is $T \cdot n \cdot m$ being $10^9$ when you assumed $10^7$. Compute the product before you code.

---

## Key Takeaways

- `pair` is two values with a built-in lexicographic `<`, which gives you a two-key sort for free.
- The `{value, index}` idiom answers "print sorted values with original positions" without a custom comparator.
- `sort` on `pair` reverses **both** keys with `rbegin()`; to reverse only the first, write a comparator.
- `tuple` generalises to any arity; access with `get<i>` (compile-time index) or structured bindings.
- `tie(a,b,c) < tie(d,e,f)` is a one-line three-key comparison — the cleanest comparator shortcut there is.
- `pair` and `tuple` work directly as `set` and `map` keys because they have `operator<`. In `unordered_map` they need a custom hash — encode as `r*m + c` instead.
- Every simulation has four parts: **state, transition, clock, termination**. Name all four before coding.
- State must be complete: if two different histories can reach the same state with different futures, something is missing.
- Time-step when $T$ is small; event-driven with a heap when events are sparse; cycle detection when $T$ is huge and the state space is finite.
- **Simultaneous updates need a second buffer.** Writing in place changes the process.
- If the statement gives no time bound, the simulation needs cycle detection. A finite deterministic system always becomes periodic.
- Off-by-one conventions (inclusive bounds, $k$-th vs $k-1$, clock before vs after) are decided by a hand trace, never by intuition.
- Compute $T \times n \times m$ before coding. That product, not the algorithm, is what TLEs.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `make_pair(1, 2.5)` assigned to `pair<int,int>` | Silent truncation of the double | Use brace init with the target type |
| 2 | `sort(vp.rbegin(), vp.rend())` expecting only `.first` reversed | Reverses both keys | Custom comparator on `.first` |
| 3 | `get<i>(t)` in a loop | `i` must be a template constant | Structured bindings, or unroll |
| 4 | `tie` with a `const` in the target | Cannot assign to a const | Drop `const`, or use `ignore` |
| 5 | `unordered_map<pair<int,int>,...>` | No default hash for `pair` | `set`, or encode as `r*m+c` |
| 6 | Updating a grid in place during an automaton step | Cells see already-updated neighbours | Double buffer, then swap |
| 7 | `cur = nxt` every step | $\mathcal{O}(nm)$ copy per step | `cur.swap(nxt)`, $\mathcal{O}(1)$ |
| 8 | `d = (d - 1) % 4` for a left turn | `-1 % 4 == -1` in C++ | `(d + 3) % 4` |
| 9 | Moving before checking bounds | Out-of-bounds write, RE not just WA | Validate the target, then move |
| 10 | Assuming a wall reflects the robot | Many problems block instead | Re-read the movement rule |
| 11 | `while (true)` with no reachable `break` | Infinite loop, TLE or OLE | Bound it, or add cycle detection |
| 12 | Simulating when the question asks only for the endpoint | Josephus in $\mathcal{O}(nk)$ instead of $\mathcal{O}(n)$ | Look for a formula |
| 13 | Clock incremented before the action when it should be after | Every completion time is off by one quantum | Hand-trace and compare to the sample |
| 14 | Counting `k` people and eliminating the $(k-1)$-th | Classic Josephus off-by-one | Skip $k-1$, eliminate the next |
| 15 | Forgetting `(long long)` on accumulators | Overflow in long simulations | `long long` for every clock and sum |
| 16 | Recomputing the whole grid when only part changed | $Tn^2$ where $Tn$ was enough | Track a work list of changed cells |
| 17 | `set` of visited states with no ordering | Compile error for custom structs | Provide `operator<`, or use `tuple` |
| 18 | Treating a simulation's answer as unique when multiple are valid | Wrong answer on a special judge | Read whether any valid output is accepted |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Robot on a grid with `L`/`R`/`F` commands | AtCoder ABC B | Easy |
| 2 | Game of Life, one or $k$ steps | Codeforces Div. 3 B / LeetCode-style | Easy–Medium |
| 3 | Josephus elimination order, $n \le 1000$ | CSES "Josephus Problem I" | Easy |
| 4 | Josephus last survivor, $n \le 10^5$ | CSES "Josephus Problem II" family | Medium |
| 5 | Round-robin CPU scheduling with a quantum | Codeforces Div. 2 B | Medium |
| 6 | Happy number / digit-square cycle detection | AtCoder ABC / interview classic | Easy–Medium |
| 7 | Bouncing ball in a box, position after $T$ ticks | Codeforces Div. 3 C | Medium |
| 8 | Ants on a line, collisions treated as pass-through | Codeforces Div. 2 C | Medium |
| 9 | Falling sand / gravity settle in a grid | Codeforces Div. 3 D | Medium |
| 10 | Snakes and ladders board simulation | Codeforces Div. 3 B | Easy–Medium |
| 11 | Simulation with $T \le 10^{18}$: detect the cycle and reduce $T$ | Codeforces Div. 1 B | Hard |
| 12 | Event-driven queue simulation with a heap | CSES / Codeforces Div. 2 D | Medium–Hard |

---

## Next Up

→ [Topic 04 — Complete Searching](Topic%2004%20-%20Complete%20Searching.md)

Simulation implements a process you were given. **Complete search** implements a process you have to invent: enumerate every candidate solution, test each one, keep the best. It is the most reliable problem-solving technique in existence — slow, always correct, and the foundation on which every optimisation is built. Topic 04 covers brute force over subsets, permutations and combinations, then adds recursion so you can search spaces that no nested loop can reach.