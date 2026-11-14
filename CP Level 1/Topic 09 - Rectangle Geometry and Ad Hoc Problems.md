# Topic 09 — Rectangle Geometry and Ad Hoc Problems

Some problems have no algorithm. There is no sort, no BFS, no DP — the statement describes a situation and you have to reason about it until it becomes a few lines of arithmetic. Those are **ad hoc** problems, and they are the largest category on Div. 2 and the one that most rewards a calm, systematic mind.

Axis-aligned rectangles are the most common setting for them: the geometry is simple enough that the difficulty is entirely in the casework and the boundary conventions.

> **Prerequisites:** [Topic 06 — Casework Problems](Topic%2006%20-%20Casework%20Problems.md), [Topic 05 — Introduction to Sorting, Sets and Maps](Topic%2005%20-%20Introduction%20to%20Sorting%2C%20Sets%20and%20Maps.md)
> **You will be able to:**
> - Fix a rectangle convention and never mix it up.
> - Compute intersection, union, distance, and containment for axis-aligned rectangles.
> - Decide between open and closed boundaries deliberately, and see why it changes the answer.
> - Apply the six ad hoc techniques: parity, invariant, extreme case, small cases, symmetry, construction.
> - Recognise when a geometry problem is really a sweep line in disguise.

---

## 1. What "ad hoc" means

An ad hoc problem is one where the intended solution is **specific to that problem** — there is no reusable algorithm, just an observation that collapses the statement.

| Standard problem | Ad hoc problem |
|---|---|
| "Find the shortest path" — BFS | "Find how many ways the robot can reach the end after exactly $k$ turns" |
| "Count pairs with sum $k$" — map | "Decide whether the array can be split so both halves have equal parity of sum" |
| "Maximum non-overlapping intervals" — greedy | "Decide whether a $3 \times 3$ board can be tiled after removing one cell" |

The distinguishing feature: **the constraint is usually tiny or the answer is usually a formula.** If $n \le 100$ and there is no obvious structure, you are probably meant to enumerate cases or find an invariant.

### The honest warning

Ad hoc problems are where sample tests lie. A solution can pass all samples and fail on an edge case nobody thought about — because ad hoc reasoning is exactly the kind of reasoning that has edge cases. **Write the boundary tests by hand.**

---

## 2. The six ad hoc techniques

Almost every ad hoc solution is one of these. Run through the list when you are stuck.

| Technique | Question to ask | Typical use |
|---|---|---|
| **Parity** | Does only odd/even matter? | Tilings, sums, colourings, moves |
| **Invariant** | What never changes? | Impossibility proofs, tiling |
| **Extreme case** | What if all elements are equal? If $n = 1$? | Constructions, greedy feasibility |
| **Small cases** | What are the answers for $n = 1,2,3,4$? | Spotting a formula |
| **Symmetry** | Does swapping roles give the same problem? | Reducing casework by half |
| **Construction** | Can I build one directly? | "Print any valid …" problems |

### The small-cases table

The single most effective ad hoc move: compute the answer by brute force for $n = 1..10$ and write the sequence down.

```
n : 1  2  3  4  5   6   7   8
a : 0  1  3  6  10  15  21  28      -> n(n-1)/2
```

If you can name the sequence, you have the formula. This is not laziness; it is how most closed forms are actually discovered.

---

## 3. Fixing the rectangle convention

Every rectangle bug traces back to a convention that was not written down. Choose one and state it in a comment.

```cpp
struct Rect {
    long long x1, y1;    // bottom-left
    long long x2, y2;    // top-right
};
// invariant: x1 < x2 and y1 < y2
```

| Convention | Meaning | Where it bites |
|---|---|---|
| `(x1,y1)` bottom-left, `(x2,y2)` top-right, **exclusive** upper | Contains $(x,y)$ iff $x_1 \le x < x_2$ | Area is $(x_2-x_1)(y_2-y_1)$ |
| Same corners, **inclusive** | Contains iff $x_1 \le x \le x_2$ | Area is $(x_2-x_1+1)(y_2-y_1+1)$ for unit cells |
| `(x,y,w,h)` | Top-left plus size | Corner arithmetic is off by `w` and `h` |

The choice matters most when the problem talks about **lattice points** or **unit cells**. "A rectangle covering cells $1..3$" has width 3, not $3 - 1 = 2$.

**Normalise on input.** If the statement does not guarantee $x_1 < x_2$, sort the pair:

```cpp
if (r.x1 > r.x2) swap(r.x1, r.x2);
if (r.y1 > r.y2) swap(r.y1, r.y2);
```

---

## 4. Solved Problem 1 — Intersection Area

> **Statement.** Given two axis-aligned rectangles, find the area of their intersection (0 if they do not overlap).

### The idea

The intersection is itself a rectangle. Its left edge is the **larger** of the two left edges, its right edge the **smaller** of the two right edges. If the resulting width or height is negative, the rectangles are disjoint.

This is the "clip to the inner bounds" pattern: maximum of the lower bounds, minimum of the upper bounds.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Rect { long long x1, y1, x2, y2; };

long long intersectionArea(const Rect &a, const Rect &b) {
    long long ix1 = max(a.x1, b.x1);
    long long iy1 = max(a.y1, b.y1);
    long long ix2 = min(a.x2, b.x2);
    long long iy2 = min(a.y2, b.y2);

    long long w = ix2 - ix1;
    long long h = iy2 - iy1;
    if (w <= 0 || h <= 0) return 0;      // disjoint or merely touching
    return w * h;
}

int main() {
    Rect a, b;
    cin >> a.x1 >> a.y1 >> a.x2 >> a.y2;
    cin >> b.x1 >> b.y1 >> b.x2 >> b.y2;
    cout << intersectionArea(a, b) << '\n';
    return 0;
}
```

**Input**
```text
0 0 4 4
2 2 6 6
```

**Output**
```text
4
```

The clipped rectangle is $[2,4] \times [2,4]$, area 4.

### Complexity

$O(1)$.

### The trap

Testing overlap as `!(a.x2 < b.x1 || b.x2 < a.x1 || ...)`. That is correct but easy to invert. Computing the clipped rectangle and checking `w <= 0` is shorter and has one place to get wrong instead of four.

Note `w <= 0` rather than `w < 0`: two rectangles that share only an edge overlap in zero area, and returning a negative number would corrupt any later sum.

---

## 5. Solved Problem 2 — Union Area of Two Rectangles

> **Statement.** Given two axis-aligned rectangles, find the total area covered by at least one of them.

### The idea

Inclusion–exclusion. The union covers everything, but the intersection is counted twice, so subtract it once:

$$|A \cup B| = |A| + |B| - |A \cap B|$$

**Why this is exact:** every point in exactly one rectangle contributes 1 to the right-hand side; every point in both contributes $1 + 1 - 1 = 1$; every point in neither contributes 0. Since inclusion–exclusion is a pointwise identity, integrating it over the plane gives the area identity.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Rect { long long x1, y1, x2, y2; };

long long area(const Rect &r) {
    return (r.x2 - r.x1) * (r.y2 - r.y1);
}

long long inter(const Rect &a, const Rect &b) {
    long long w = min(a.x2, b.x2) - max(a.x1, b.x1);
    long long h = min(a.y2, b.y2) - max(a.y1, b.y1);
    if (w <= 0 || h <= 0) return 0;
    return w * h;
}

int main() {
    Rect a, b;
    cin >> a.x1 >> a.y1 >> a.x2 >> a.y2;
    cin >> b.x1 >> b.y1 >> b.x2 >> b.y2;

    cout << area(a) + area(b) - inter(a, b) << '\n';
    return 0;
}
```

**Input**
```text
0 0 4 4
2 2 6 6
```

**Output**
```text
28
```

$16 + 16 - 4 = 28$.

### Complexity

$O(1)$.

### The trap

Inclusion–exclusion does **not** extend to three rectangles with the simple formula $|A|+|B|+|C| - |AB| - |BC| - |CA|$ alone — you must add back $|ABC|$. And it does **not** generalise to $n$ rectangles at all in practice, because $2^n$ terms are needed. Three or more rectangles require a sweep line (Chapter 7).

---

## 6. Solved Problem 3 — Point Strictly Inside

> **Statement.** Given a rectangle and a point, decide whether the point lies strictly inside (not on the boundary).

### The idea

Four inequalities, all strict. The only difficulty is knowing which the problem wants.

| Wording | Test | Boundary counts? |
|---|---|---|
| "strictly inside" | `x1 < x && x < x2 && y1 < y && y < y2` | no |
| "inside or on the boundary" | `x1 <= x && x <= x2 && y1 <= y && y <= y2` | yes |
| "in the interior of a cell grid" | depends on the cell convention | read carefully |

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    long long x1, y1, x2, y2, px, py;
    cin >> x1 >> y1 >> x2 >> y2 >> px >> py;

    bool inside = (x1 < px && px < x2 && y1 < py && py < y2);
    cout << (inside ? "YES" : "NO") << '\n';
    return 0;
}
```

**Input**
```text
0 0 10 10
5 5
```

**Output**
```text
YES
```

A point at `0 0` would print `NO` because the test is strict.

### Complexity

$O(1)$.

### The trap

Writing `x1 <= x <= x2` as a chained comparison. In C++ that parses as `(x1 <= x) && (x <= x2)`, which happens to be correct for booleans — but for types with overloaded operators it is not, and it is confusing to read. Write the two comparisons separately.

---

## 7. Solved Problem 4 — Distance Between Two Rectangles

> **Statement.** Given two axis-aligned rectangles, find the Euclidean distance between them, defined as 0 if they intersect or touch.

### The idea

Project onto each axis separately. The horizontal gap is how far apart the $x$-ranges are; the vertical gap is how far apart the $y$-ranges are. Both are clamped at zero, and the distance is $\sqrt{dx^2 + dy^2}$.

$$dx = \max\bigl(0,\ \max(x_1^A, x_1^B) - \min(x_2^A, x_2^B)\bigr)$$

This is the same "inner bounds" arithmetic as the intersection, read the other way round: intersection asks how much the ranges *overlap*, distance asks how much they *miss by*.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Rect { long long x1, y1, x2, y2; };

int main() {
    Rect a, b;
    cin >> a.x1 >> a.y1 >> a.x2 >> a.y2;
    cin >> b.x1 >> b.y1 >> b.x2 >> b.y2;

    long long dx = max(0LL, max(a.x1, b.x1) - min(a.x2, b.x2));
    long long dy = max(0LL, max(a.y1, b.y1) - min(a.y2, b.y2));

    double d = sqrt((double)(dx * dx + dy * dy));
    cout << fixed << setprecision(4) << d << '\n';
    return 0;
}
```

**Input**
```text
0 0 2 2
5 3 7 8
```

**Output**
```text
3.1623
```

$dx = 5 - 2 = 3$, $dy = 3 - 2 = 1$, so $d = \sqrt{10} \approx 3.1623$.

### Complexity

$O(1)$.

### The trap

Using `sqrt(dx*dx + dy*dy)` with `int` arithmetic for the squares. Coordinates up to $10^9$ square to $10^{18}$, which overflows `int` and produces a garbage distance. Cast to `long long` or `double` before squaring.

---

## 8. Solved Problem 5 — Minimum Enclosing Square

> **Statement.** Given $n$ points, find the minimum side length $L$ of an axis-aligned square that contains all of them.

### The idea

The answer is a formula, not an algorithm:

$$L = \max(\max x - \min x,\ \max y - \min y)$$

### Why

Any enclosing square of side $L$ must span the full $x$-range, so $L \ge \max x - \min x$; likewise for $y$. So $L$ is at least the larger of the two ranges.

Conversely, take $L$ equal to that larger range and place the square with its bottom-left corner at $(\min x, \min y)$. Every point satisfies $\min x \le x \le \min x + L$ because the $x$-range is at most $L$, and likewise in $y$. So this square works.

Two inequalities, one construction — that is the entire proof.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    long long minx = LLONG_MAX, maxx = LLONG_MIN;
    long long miny = LLONG_MAX, maxy = LLONG_MIN;

    for (int i = 0; i < n; i++) {
        long long x, y;
        cin >> x >> y;
        minx = min(minx, x); maxx = max(maxx, x);
        miny = min(miny, y); maxy = max(maxy, y);
    }

    cout << max(maxx - minx, maxy - miny) << '\n';
    return 0;
}
```

**Input**
```text
3
1 1
4 1
4 3
```

**Output**
```text
3
```

The $x$-range is $4 - 1 = 3$, the $y$-range is $3 - 1 = 2$, so $L = 3$.

### Complexity

$O(n)$ time, $O(1)$ space. No sorting, no binary search.

### The trap

Binary searching on $L$ with a feasibility check. It is correct, but it is $O(n \log(\text{range}))$ for a problem whose answer is one subtraction. **Before reaching for binary search, check whether the answer is a formula.**

---

## 9. Ad hoc patterns that recur

### 9.1 Parity

Only the parity of a quantity matters, so track it instead of the quantity.

| Problem | Parity insight |
|---|---|
| Can a $1 \times 2$ domino tile an $n \times m$ board? | Yes iff $nm$ is even |
| Can you make all elements equal using $\pm 1$ moves? | Only if the sum keeps a fixed residue |
| Does a bishop reach $(r_2,c_2)$ from $(r_1,c_1)$? | Only if $r_1+c_1$ and $r_2+c_2$ have the same parity |
| Can a sequence of moves change the parity of the inversion count? | Each move changes it by a fixed amount |

### 9.2 Invariants

Find a quantity that the allowed operations never change, then show the target has a different value.

*Example.* A board is coloured like a chessboard. Every domino covers one black and one white cell. So any region with unequal black and white counts cannot be tiled — **regardless of its shape**. This single sentence settles an entire family of tiling problems.

### 9.3 Extreme cases

Try the input that breaks naive assumptions:

- $n = 1$ (no pairs exist, no loop runs)
- all elements equal (no "different element" exists)
- all zeros (division, signs, `abs`)
- the maximum constraint value (overflow, array bounds)

If your solution is correct on all four, most ad hoc bugs are gone.

### 9.4 Construction

For "output any valid object" problems, build greedily and prove the construction stays valid:

```
For a string of length n with exactly k distinct characters:
  place k distinct characters first, fill the rest with the first one
  -> valid whenever k <= n and k <= 26
```

The construction is usually 3 lines; the work is enumerating when it is impossible.

---

## 10. When geometry is really a sweep

Two rectangles: inclusion–exclusion, $O(1)$.

Many rectangles: you need the area of a union of $n$ axis-aligned rectangles, which is a sweep line over $x$ with an active set of $y$-intervals — $O(n \log n)$ with a segment tree, and the subject of Chapter 7.

Recognise the difference by the constraint on $n$:

| $n$ rectangles | Technique |
|---|---|
| 2 | Inclusion–exclusion, $O(1)$ |
| 3 | Inclusion–exclusion with $2^3$ terms |
| $\le 2000$ | $O(n^2)$ pairwise contribution, or coordinate compression |
| $\le 2 \times 10^5$ | Sweep line, $O(n \log n)$ |

---

## 11. Floating point and geometry

Three rules, all learned the hard way.

1. **Prefer integers.** If the input is integral and the answer is an area, keep everything in `long long`. No rounding, no epsilon.
2. **Epsilon comparisons are a last resort.** `a == b` on `double` is usually wrong; `abs(a - b) < 1e-9` is usually better; but the best fix is often to avoid `double` entirely.
3. **Fix the output format at the start.** `cout << fixed << setprecision(4)` changes every later double print; if you set it mid-solution, earlier outputs are already formatted differently.

| Operation | Integer-safe? |
|---|---|
| Area of an axis-aligned rectangle | yes |
| Intersection area | yes |
| Distance between rectangles | no — needs `sqrt`, output as `double` |
| Comparing two such distances | square them, compare integers |
| Angles, rotations | no — trigonometry |

---

## Key Takeaways

- Ad hoc problems have no reusable algorithm; the solution is an observation specific to the statement.
- Run the six techniques in order: parity, invariant, extreme case, small cases, symmetry, construction.
- **Write down the brute-force answers for $n = 1..10$.** Most closed forms are discovered this way, not derived.
- Fix the rectangle convention in a comment (`x1,y1` bottom-left, `x2,y2` top-right) and normalise corners on input.
- Intersection clips to the inner bounds: `max` of lower edges, `min` of upper edges. A non-positive width or height means no area.
- Test disjoint by `w <= 0 || h <= 0`, not by four separate inequality checks.
- Union of two rectangles is $A + B - A \cap B$. It does not extend to $n$ rectangles without a sweep.
- Rectangle distance is the intersection arithmetic read backwards: how far the ranges *miss* rather than how much they *overlap*.
- The minimum enclosing square side is `max(x-range, y-range)` — a formula, not a binary search.
- Parity and colouring invariants settle most tiling problems with one sentence.
- Integer arithmetic for areas; `double` only for lengths that genuinely need a square root.
- Ad hoc solutions pass samples and fail edge cases. Test $n=1$, all-equal, all-zero, and the maximum value by hand.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Mixing corner conventions | Off-by-one area, wrong containment | Write the convention once, in a comment |
| 2 | Treating touching rectangles as overlapping in area | Adds 0 but the test says yes | Use strict `w > 0 && h > 0` for area |
| 3 | `int` for coordinate products | $10^9 \times 10^9$ overflows | `long long` everywhere |
| 4 | `!(a.x2 < b.x1 \|\| ...)` overlap test | Easy to invert | Compute the clipped rect instead |
| 5 | Inclusion–exclusion for $n \ge 4$ rectangles | $2^n$ terms | Sweep line |
| 6 | Forgetting the $+1$ for lattice points | "Cells 1..3" is width 3, not 2 | Decide inclusive/exclusive explicitly |
| 7 | Binary searching a formula | $O(n \log C)$ for an $O(n)$ answer | Check for a closed form first |
| 8 | Comparing doubles with `==` | Rounding | Compare integers, or use an epsilon |
| 9 | Setting `fixed << setprecision` late | Earlier output formatted differently | Set format once, before any output |
| 10 | `sqrt` of an `int` expression | Overflow before the cast | Cast the operands |
| 11 | Assuming corners arrive in order | `x1 > x2` produces negative area | Normalise on input |
| 12 | Testing only the sample | Sample never contains the degenerate case | Test $n=1$, all-equal, all-zero |
| 13 | Ignoring "output any valid answer" freedom | Over-engineering a unique solution | Construct greedily |
| 14 | Forgetting that area can be zero | Negative areas corrupt sums | Clamp at 0 |
| 15 | `double` coordinates read with `cin` for integer problems | Precision loss | Read as integers |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Intersection area of two rectangles | Classic / Codeforces | Easy |
| 2 | Union area of two rectangles | Classic | Easy |
| 3 | Point inside / on a rectangle | Codeforces Div. 2 A | Easy |
| 4 | Distance between two rectangles | Codeforces | Easy–Medium |
| 5 | Minimum enclosing square | Classic ad hoc | Easy |
| 6 | Does a bishop reach a square? | Codeforces Div. 2 A | Easy |
| 7 | Can a domino tile the board? | Classic parity | Easy |
| 8 | Count lattice points strictly inside | AtCoder ABC | Easy–Medium |
| 9 | Maximum rectangles overlapping a point | Codeforces Div. 2 C (sweep or small $n$) | Medium |
| 10 | Rotate / reflect a rectangle set | Codeforces Div. 2 B | Medium |
| 11 | Union area of $n \le 2000$ rectangles | Codeforces / CSES | Medium–Hard |
| 12 | Union area of $n \le 2 \times 10^5$ rectangles | Chapter 7, sweep line | Hard |

---

## Next Up

→ [Topic 10 — Level 1 Solved Problem Set](Topic%2010%20-%20Level%201%20Solved%20Problem%20Set.md)

Every technique in Chapter 2 is now on the table: complete search, sorting, casework, greedy, graphs, and ad hoc geometry. Topic 10 is twenty problems drawn from across all of them, each solved in full with the reasoning that *chooses* the technique — which is the part that contest time actually measures.
