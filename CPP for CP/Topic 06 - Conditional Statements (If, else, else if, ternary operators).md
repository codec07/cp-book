# Topic 06 — Conditional Statements: `if`, `else`, `else if`, and the Ternary Operator

Almost every line of competitive code is a decision: is this index inside the grid, is this prefix sum already seen, is this candidate better than the current best. C++ gives you exactly four ways to branch — `if`, `switch`, the ternary operator, and short-circuit operators used as guards — and each of them has a trap that costs submissions. This chapter covers the syntax you already half-know, then spends its time on the parts that actually decide contests: evaluation order, truthiness of odd values, branchless arithmetic, and the boundary check you will write ten thousand times.

> **Prerequisites:** [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems,%20Variables%20and%20Datatypes.md), [Topic 05 — String Manipulation Methods & Statements](Topic%2005%20-%20String%20Manipulation%20Methods%20&%20Statements.md)
> **You will be able to:**
> - Write branch conditions that cannot be read wrongly by you at 2 a.m.
> - Explain exactly when `if (i < n && a[i] > 0)` is safe and the reverse order segfaults.
> - Use `switch` without the fall-through bug, and know when a lookup table beats 30 branches.
> - Use the ternary operator without hitting its precedence and type-deduction surprises.
> - Write a reusable `valid()` guard for grid problems.
> - Estimate what a mispredicted branch costs in a hot loop.

---

## 1. The `if` statement and what "true" means

The grammar is minimal:

```cpp
if (condition)
    statement;          // runs when condition is non-zero

if (condition) {
    statement1;         // a block is a single statement
    statement2;
}
```

C++ has no dedicated `bool` requirement for a condition. Any scalar converts to `bool` with one rule: **zero is false, everything else is true**. That includes negative numbers, which is where people coming from languages with stricter typing get surprised.

```cpp
if (-1)  cout << "runs\n";        // runs: -1 is non-zero
if (0.0) cout << "never\n";       // does not run: 0.0 is zero
if (0.0001) cout << "runs\n";     // runs: non-zero double
if ('0') cout << "runs\n";        // runs: '0' is 48, not 0
if (nullptr) cout << "never\n";   // does not run: null pointer is false
```

That fourth line is worth memorising. `'0'` and `0` are different values. A string of digits is never "false", even when every character is `'0'`.

| Expression | Value | Truthy? | Note |
|---|---|---|---|
| `0` | 0 | no | |
| `-1` | −1 | **yes** | only zero is false |
| `0.0` | 0.0 | no | |
| `-0.0` | −0.0 | no | equals zero |
| `0.0001` | tiny | **yes** | |
| `'0'` | 48 | **yes** | the character, not the digit |
| `""` | address of a byte | **yes** | a non-null pointer, see §2 |
| `nullptr` | 0 | no | |
| `NaN` | NaN | **yes** | NaN is non-zero, and `NaN == 0` is false |

The `NaN` row deserves a note because it breaks reasoning by cases. `NaN` is not equal to anything, including itself, but it *is* truthy. If a floating-point computation can produce `NaN` (a `sqrt` of a negative, a `0.0/0.0`, an `acos` outside $[-1,1]$), the branching logic downstream may take a path you never considered.

---

## 2. The assignment-in-condition typo

The single most expensive typo in beginner code is `=` where `==` was meant:

```cpp
int x = 0;
if (x = 5) {                 // assigns 5, then tests 5 -> always true
    cout << "x is now " << x << '\n';
}
```

```text
x is now 5
```

It compiles, it runs, and it is wrong. Every compiler worth using warns about it:

```text
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

Three defences, in order of usefulness:

1. **Compile with `-Wall -Wextra`.** The warning is exact and fires on the typo, not on the legitimate use.
2. **When you genuinely mean it** (rare — the canonical case is `while ((c = getchar()) != EOF)`), write the extra parentheses. `if ((x = 5))` silences the warning and documents intent.
3. **Put the constant on the left** (`if (5 == x)`) if you like the old C habit — the typo `5 = x` then fails to compile. Modern style prefers the warning instead, because `if (5 == x)` reads backwards.

The related smell is `if (x == true)` and `if (flag == false)`. They are correct, just noisy: write `if (x)` and `if (!flag)`. The exception is when `x` is not a `bool` (`int`, pointer), where the explicit comparison can document intent — `if (ptr != nullptr)` is clearer than `if (ptr)`.

---

## 3. Blocks, scope, and the brace-less trap

C++ lets you omit braces for a single statement. Do not.

```cpp
if (score >= 60)
    cout << "pass\n";
    cout << "congratulations\n";   // NOT part of the if — always runs
```

The second line is indented as if it belonged to the branch, but the `if` ended at the semicolon. This bug survives code review because the shape of the code lies. **Always brace.** It costs one line and removes an entire bug class, including the one you create later when you add a second statement to the branch and forget the braces.

Variables declared inside the block die at the closing brace:

```cpp
if (n > 0) {
    int half = n / 2;
}
cout << half;      // compile error: 'half' was not declared in this scope
```

This is a feature: it stops you from reading a value that only exists on one path. The compiler is telling you that `half` might not have been computed. Declare it outside with a default if you need it afterwards:

```cpp
int half = 0;
if (n > 0) half = n / 2;
```

Also note that C++17 allows an initialiser inside the condition, which keeps a variable alive for the whole `if`/`else` chain:

```cpp
if (int d = dist[u]; d < best) {      // d is valid in the else branch too
    best = d;
} else {
    cout << "worse by " << d - best << '\n';
}
```

Use it sparingly — it is a readability win for one-liners and a puzzle for anything longer.

---

## 4. `else` and `else if` chains

`else if` is not a keyword; it is an `else` whose single statement happens to be another `if`. That is why the chain has no special syntax and why exactly one branch runs:

```cpp
if (x < 0)        cout << "negative\n";
else if (x == 0)  cout << "zero\n";
else if (x < 10)  cout << "small\n";
else              cout << "large\n";
```

Branches are tested **top to bottom** and the first true one wins. Order is therefore part of the logic, not cosmetic. Reverse the two middle tests and `x = 5` reports `zero`? No — it reports `small`, still correct here because the tests are disjoint. But make them overlap and order becomes everything:

```cpp
// wrong: 15 is reported as "small"
if (x < 10)       cout << "small\n";
else if (x < 0)   cout << "negative\n";
```

Put the most specific or most restrictive test first. When the tests are mutually exclusive (as in the first example), any order works — prefer the order a human would read.

### The dangling `else`

An `else` binds to the **nearest unmatched `if`**, regardless of indentation:

```cpp
if (a > 0)
    if (b > 0)
        cout << "both\n";
else                       // binds to the INNER if, not the outer one
    cout << "outer failed\n";
```

The compiler is right and the indentation is lying. Braces fix it, and so does `-Wdangling-else` (included in `-Wall`). This is the strongest argument for braces that exists, and it is why style guides ban brace-less nested `if`s outright.

---

## 5. Comparison operators and the equality traps

| Operator | Meaning | Trap |
|---|---|---|
| `==` | equal | `=` typo; float equality |
| `!=` | not equal | |
| `<` `<=` `>` `>=` | ordering | mixing signed and unsigned |
| `<=>` | three-way (C++20) | returns a comparison category, not a `bool` |

### Floating-point equality

`==` on `double` compares exact bit patterns after rounding, so it answers a question you rarely meant to ask:

```cpp
double a = 0.1 + 0.2;
cout << (a == 0.3) << '\n';        // 0
cout << setprecision(20) << a << '\n';   // 0.30000000000000004441
```

0.1, 0.2 and 0.3 have no exact binary representation, so the sum carries a tiny error. Compare with a tolerance instead:

```cpp
const double EPS = 1e-9;
bool eq(double a, double b) { return fabs(a - b) <= EPS; }
bool lt(double a, double b) { return a < b - EPS; }   // strict, with slack
bool le(double a, double b) { return a <= b + EPS; }
```

The tolerance must be scaled to the magnitude of the values. `1e-9` is invisible next to $10^{12}$ and enormous next to $10^{-15}$. A relative test is more robust:

```cpp
bool close(double a, double b) {
    return fabs(a - b) <= 1e-9 * max({1.0, fabs(a), fabs(b)});
}
```

*Contest story: a geometry problem asked whether three points are collinear. The intended solution used exact integer cross products in `long long`. Half the submissions used `double` slopes and `==`, and failed on the one test with points whose slope was exactly representable in one order and not the other. Exact integer arithmetic whenever the input is integral — always.*

### Signed/unsigned comparisons

Comparing an `int` with a `size_t` converts the `int` to unsigned. If the `int` is negative, it becomes a huge positive number:

```cpp
vector<int> v(10);
int i = -1;
if (i < v.size()) cout << "true\n";   // prints "true": -1 became 18446744073709551615
```

Wait — `-1` converted to `size_t` is $2^{64}-1$, which is **not** less than 10. The condition is false, so nothing prints. The dangerous version is the one where the intent was "is this index valid":

```cpp
if (i >= 0 && i < (int)v.size()) { ... }   // the cast is the point
```

`-Wsign-compare` (in `-Wall`) reports every mixed comparison. Fix by casting the size once, or by using `int n = (int)v.size();` at the top of the function and comparing `int` to `int`.

### `NaN` never compares

For any `NaN` value `x`: `x == x` is false, `x < y` is false, `x > y` is false, `x != y` is true. Sorting a container with `NaN` in it, or using `NaN` as a binary-search bound, produces nonsense with no error message. Guard at the source: `if (std::isnan(x)) { ... }`.

---

## 6. Short-circuit evaluation: where correctness lives

`&&` and `||` evaluate left to right and **stop as soon as the answer is known**. `&&` stops at the first false, `||` at the first true. This is not an optimisation detail; it is a correctness mechanism you use constantly.

```cpp
// SAFE: the first test protects the second
if (i < n && a[i] == target) { ... }

// BROKEN: a[i] is read before i is validated
if (a[i] == target && i < n) { ... }     // out-of-bounds read, then maybe SIGSEGV
```

The same pattern guards pointers and containers:

```cpp
if (p != nullptr && p->value > 0) { ... }        // safe
if (mp.count(key) && mp[key] == 0) { ... }       // safe: count() does not insert
if (mp[key] == 0 && mp.count(key)) { ... }       // WRONG: operator[] already inserted
```

That last pair is a real `std::map` trap: `mp[key]` default-constructs the entry if it is missing, so "check first" must be written with `count` or `find`.

### Side effects and order

Because the right side may not run, never put a needed side effect there:

```cpp
int i = 0;
if (false && i++) { }        // i stays 0 — the increment never happens
if (true  || i++) { }        // i stays 0
```

And do not rely on the order of operations *within* one operand: `f() + g()` has unspecified evaluation order in C++17, so `a[i] = i++` is undefined behaviour. `&&` and `||` are the only operators with a guaranteed left-to-right order (along with the comma operator).

### De Morgan's laws

The negation of a compound condition flips the operator and negates both sides:

$$\neg(A \land B) = \neg A \lor \neg B, \qquad \neg(A \lor B) = \neg A \land \neg B$$

```cpp
if (!(x > 0 && y > 0))       // is the same as
if (x <= 0 || y <= 0)
```

When negating a guard, apply the law mechanically and re-check the boundary operators: `>` becomes `<=`, not `<`. The off-by-one that appears when people negate by intuition instead of by rule is the source of a large share of wrong answers on "count the invalid cells" problems.

---

## 7. Chained comparisons do not exist in C++

In Python, `0 < x < 10` means "x is between 0 and 10". In C++ it compiles and means something else entirely:

```cpp
int x = 50;
if (0 < x < 10) cout << "in range\n";    // prints! and it is wrong
```

The reason is associativity. `<` is left-associative, so the expression parses as `(0 < x) < 10`. `0 < 50` yields `true`, which converts to `1`, and `1 < 10` is `true`. The test is therefore **true for every positive `x`** and false only for `x <= 0`. No warning fires by default, because every individual comparison is well-formed.

| Language | `0 < x < 10` means |
|---|---|
| Python | `0 < x and x < 10` |
| C++ | `(0 < x) < 10` — i.e. `x > 0` for all practical inputs |
| Rust | compile error |
| Java | compile error |

Write it out:

```cpp
if (0 < x && x < 10) { ... }
```

This is the highest-value single line in this chapter. If you take one habit away, take this one.

---

## 8. `switch`: fast dispatch, one sharp edge

`switch` compares one integral expression against a list of compile-time constants:

```cpp
switch (ch) {
    case 'a':
    case 'e':
    case 'i':
    case 'o':
    case 'u':
        cout << "vowel\n";
        break;                 // without this, execution falls through to 'default'
    case ' ':
        cout << "space\n";
        break;
    default:
        cout << "consonant\n";
}
```

Facts that matter in contests:

- **Only integral types.** `switch` works on `char`, `int`, `long long`, `enum`. It does not work on `std::string`, `double`, or a `pair`. For strings you use an `if` chain or a `map`.
- **`case` labels must be compile-time constants.** `case n:` where `n` is a variable is an error.
- **Fall-through is the default.** Forgetting `break` runs the next case's body. This is occasionally useful (grouping labels, as with the vowels above) and usually a bug. g++ warns with `-Wimplicit-fallthrough`.
- **No duplicate labels.** `case 1:` twice is a compile error.
- **`default` is optional** but should be present in contest code — it is your assertion that the input is in range.

The compiler may build a jump table, making dispatch $\mathcal{O}(1)$ instead of a chain of comparisons. That is the only performance reason to prefer `switch` over `if`. For 30 branches keyed on a small integer, a lookup array is faster and clearer:

```cpp
static const int dx[4] = {1, -1, 0, 0};
static const int dy[4] = {0, 0, 1, -1};

for (int d = 0; d < 4; d++) {
    int nx = x + dx[d], ny = y + dy[d];
    ...
}
```

versus the same logic spelled out with four `if`s. The array version is shorter, has no branch misprediction, and extends to eight directions by adding four numbers. **Whenever a branch selects a constant, ask whether an array indexed by the case value would do the job.**

---

## 9. The ternary operator

```cpp
condition ? value_if_true : value_if_false
```

It is an **expression**, not a statement, so it produces a value you can assign, pass, or print:

```cpp
int a = 7, b = 3;
int mx = (a > b) ? a : b;                 // 7
cout << (a % 2 ? "odd" : "even") << '\n'; // odd
string label = (n == 1) ? "item" : "items";
```

### The precedence trap

`?:` has very low precedence, lower than `<<`. Without parentheses this does not compile the way you expect:

```cpp
cout << n == 0 ? "zero" : "nonzero";        // parses as (cout << n) == 0 ? ... 
cout << (n == 0 ? "zero" : "nonzero");      // correct
```

The first line compiles — it prints `n` and then evaluates the comparison on the stream — and produces output you did not intend. **Always parenthesise a ternary inside a stream expression** (or anywhere else with operators around it).

### Type deduction

Both branches must produce a common type, and the result type is that common type. This is where a truncating bug can hide:

```cpp
int n = 5;
auto x = (n > 0) ? n : 0.5;    // common type is double -> x is 5.0
long long big = (n > 0) ? n : 0LL;   // common type is long long -> fine
int bad = (n > 0) ? 3000000000LL : 0; // common type long long, then narrowed to int
```

The third line is a silent narrowing. If both branches are different integer types, the usual arithmetic conversions apply and the *larger* type wins — but only if one branch is actually of that type.

### Nesting

Nested ternaries are legal, right-associative, and quickly unreadable:

```cpp
string s = (x < 0) ? "neg" : (x == 0) ? "zero" : "pos";     // tolerable
string t = a ? b ? "bb" : "bc" : c ? "cb" : "cc";           // write this as if/else
```

One level is fine. Two levels is the limit before a reviewer (or you, tomorrow) misreads it.

### Ternary as a conditional lvalue

In C++, if both branches are lvalues of the same type, the result is an lvalue and can be assigned:

```cpp
int a = 1, b = 2;
(a < b ? a : b) = 100;      // assigns to a, since a < b — legal but weird
```

Do not do this in contest code. It is a party trick, not a technique.

### When the ternary earns its place

- A one-line `min`/`max` on values whose type is awkward for `std::min`: `int v = (a < b) ? a : b;`
- Printing a plural or a label: `cout << n << (n == 1 ? " apple" : " apples");`
- Choosing between two constants: `int dir = forward ? 1 : -1;`
- Initialising a `const` from a branch, where an `if` would need a mutable variable.

Anything longer belongs in an `if`.

---

## 10. Guard clauses, early return, and flat code

Deeply nested `if`s are how a 30-line function becomes unreadable. The fix is to **reject invalid input at the top and return**, then write the happy path unindented:

```cpp
// Nested version: four levels deep
bool check(int n, vector<int>& a) {
    if (n > 0) {
        if ((int)a.size() == n) {
            for (int x : a) {
                if (x < 0) {
                    return false;      // finally
                }
            }
            return true;
        }
    }
    return false;
}

// Guard-clause version: flat
bool check(int n, const vector<int>& a) {
    if (n <= 0) return false;
    if ((int)a.size() != n) return false;
    for (int x : a)
        if (x < 0) return false;
    return true;
}
```

Same behaviour, half the indentation, one exit per condition. This style is now standard in CP because it also makes the failure conditions explicit — you can read the guards and know exactly what the function refuses.

The pattern generalises to the `solve()` function of a problem:

```cpp
void solve() {
    int n; cin >> n;
    vector<int> a(n);
    for (int &x : a) cin >> x;

    if (n == 1) { cout << 0 << '\n'; return; }   // special case, out of the way
    if (*min_element(a.begin(), a.end()) < 0) { cout << -1 << '\n'; return; }

    // ... the actual solution, never indented by a special case
}
```

Putting every degenerate case at the top as a guard is the single biggest readability improvement available in contest code, and it reduces the chance of an edge case hiding inside the main loop.

### Predicate functions

When a condition is complex or reused, name it:

```cpp
auto isLeap = [](int y) {
    return (y % 4 == 0 && y % 100 != 0) || (y % 400 == 0);
};

auto isPrime = [](long long n) {
    if (n < 2) return false;
    for (long long d = 2; d * d <= n; d++)
        if (n % d == 0) return false;
    return true;
};
```

A named predicate can be tested in isolation, reused in `count_if` and `find_if` (see [Topic 15](Topic%2015%20-%20STL%20Part%204%20%28Essential%20Prebuilt%20Algorithms%20-%20Sort,%20Binary%20Search,%20Min%20Max,%20Array%20Manipulation,%20Permutations,%20Math%20&%20Accumulators%29.md)), and read at the call site as English.

---

## 11. Validity checks: the grid guard

The most-written condition in all of competitive programming is "is this cell inside the grid". Get it wrong once per contest and you lose more points than any algorithm will win you.

```cpp
int n, m;                       // rows, columns

auto valid = [&](int r, int c) {
    return r >= 0 && r < n && c >= 0 && c < m;
};

for (int r = 0; r < n; r++)
    for (int c = 0; c < m; c++) {
        for (int d = 0; d < 4; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (!valid(nr, nc)) continue;
            // use grid[nr][nc]
        }
    }
```

Three things to notice:

1. **All four comparisons use `&&`, not `||`.** With `||`, a cell at `r = -1` passes because `c` is in range. This is the classic grid bug.
2. **`>= 0` must come first** in a short-circuit chain if the next term indexes with `r` — which is why the version above, which never indexes, is safest of all.
3. **The lambda captures by reference** and is inlined, so it costs nothing.

For a single check without a helper, write the comparison in the order that short-circuits usefully:

```cpp
if (nr >= 0 && nr < n && nc >= 0 && nc < m) grid[nr][nc] = 1;
```

For a torus (wrap-around grid), the modulo must be normalised:

```cpp
nr = ((r + dr[d]) % n + n) % n;      // C++ % can be negative
nc = ((c + dc[d]) % m + m) % m;
```

`-1 % 5` is `-1` in C++, not `4`. See [Topic 08](Topic%2008%20-%20Useful%20Math%20functions%20&%20Operations%20on%20Data.md) for the full discussion of negative modulo.

---

## 12. Branchless techniques and the cost of a branch

A modern CPU predicts branches and runs ahead. A correctly predicted branch is nearly free. A **mispredicted** one costs roughly 10–20 cycles, because the pipeline must be flushed. In a loop running $10^8$ times over unpredictable data, that is the difference between 0.5 s and 3 s.

*Contest story: a Div. 2 problem asked for the count of elements greater than a threshold, with $n = 2 \times 10^7$. The `if (a[i] > t) cnt++;` version passed in 1.1 s on random data and TLE'd on the adversarial test where the data was sorted — every branch mispredicted. Rewriting it branchlessly took it to 0.35 s. The algorithm was identical.*

The standard branchless idioms:

```cpp
// Count with an arithmetic trick instead of a branch
cnt += (a[i] > t);

// Sign of x, as -1, 0, +1, with no branch
int sign = (x > 0) - (x < 0);

// Conditional select: same idea as ternary, compiler emits cmov
int m = (a < b) ? a : b;            // usually compiles to a conditional move
int M = a * (a > b) + b * (b >= a);

// Bit trick: mask is all-ones when the condition holds
int mask = -(a > b);
int mn = (a & mask) | (b & ~mask);
```

In practice you should reach for `std::min`, `std::max` and `std::clamp` rather than hand-rolled masks — the compiler knows how to lower them to `cmov` or `min`/`max` instructions, and they are readable. The bit-mask versions are worth recognising in other people's code, and worth writing only when you have profiled the loop.

Two other performance-adjacent habits:

- **Put the common case first** in an `if`/`else` chain. The predictor learns the common path, and the fall-through is cheaper than the jump.
- **Prefer arithmetic to branching in hot loops** when the operation is cheap: `cnt += cond;` beats `if (cond) cnt++;` for a `bool`-valued `cond`.

None of this matters in code that runs $10^4$ times. Do not complicate a loop that runs $10^6$ times for a 1 ms gain.

---

## 13. Worked mini-problems

### 13.1 Leap year

```cpp
bool isLeap(int y) {
    return (y % 4 == 0 && y % 100 != 0) || (y % 400 == 0);
}
```

| Year | Divisible by 4 | by 100 | by 400 | Leap |
|---|---|---|---|---|
| 2024 | yes | no | no | **yes** |
| 1900 | yes | yes | no | no |
| 2000 | yes | yes | yes | **yes** |
| 2023 | no | — | — | no |

The order matters for reading, not for correctness: `&&` binds tighter than `||`, so no parentheses are needed around the first clause — but they make the 4/100/400 structure obvious.

### 13.2 Grade classifier with `switch`

```cpp
char grade(int marks) {                  // 0..100
    if (marks < 0 || marks > 100) return '?';   // guard first
    switch (marks / 10) {
        case 10: case 9: return 'A';
        case 8:          return 'B';
        case 7:          return 'C';
        case 6:          return 'D';
        default:         return 'F';
    }
}
```

Two lessons: the guard clause rejects invalid input before the switch, and the grouped `case 10: case 9:` uses fall-through **deliberately** — the only place it is acceptable.

### 13.3 Triangle validity

```cpp
bool isTriangle(long long a, long long b, long long c) {
    if (a <= 0 || b <= 0 || c <= 0) return false;
    return a + b > c && b + c > a && c + a > b;
}
```

All three inequalities are required. Checking only `a + b > c` is the classic error. Use `long long` because `a + b` overflows `int` for inputs near $2 \times 10^9$.

Degenerate (collinear) triangles satisfy `a + b == c` and must be rejected — hence `>` not `>=`.

### 13.4 Sign function without branches

```cpp
int sign(long long x) { return (x > 0) - (x < 0); }
```

`(x > 0)` is `1` or `0`, `(x < 0)` is `1` or `0`, so the difference is `+1`, `0`, or `-1`. No branch, no comparison chain, works for `INT_MIN` and `LLONG_MIN` (unlike `x < 0 ? -1 : x > 0 ? 1 : 0`, which is also fine but longer).

### 13.5 Maximum of three

```cpp
int mx = max({a, b, c});            // initializer-list form, C++11
int mx2 = max(a, max(b, c));        // nested, works everywhere
int mx3 = (a > b) ? ((a > c) ? a : c) : ((b > c) ? b : c);   // don't
```

The third is correct and unreadable. This is exactly the kind of code the ternary should not be used for.

### 13.6 Quadrant of a point

```cpp
int quadrant(long long x, long long y) {
    if (x > 0 && y > 0) return 1;
    if (x < 0 && y > 0) return 2;
    if (x < 0 && y < 0) return 3;
    if (x > 0 && y < 0) return 4;
    return 0;                       // on an axis
}
```

Note the four `&&` and the final fall-through to `0` for axis points. Forgetting the zero case is the standard wrong answer on "quadrant" problems — the statement always includes points on the axes in at least one test.

### 13.7 Parity of a negative number

```cpp
bool isEven(long long x) { return x % 2 == 0; }        // correct for negatives
bool isOddBad(long long x) { return x % 2 == 1; }     // WRONG for negatives
```

`-3 % 2` is `-1`, not `1`, so `isOddBad(-3)` returns `false`. Use `x % 2 != 0`, or `abs(x) % 2 == 1`, or `(x & 1) == 1` — the bit test works for negatives in two's complement, and `x & 1` is `1` for any odd `x` including negative ones.

### 13.8 FizzBuzz, the canonical branch problem

```cpp
for (int i = 1; i <= n; i++) {
    if (i % 15 == 0)      cout << "FizzBuzz\n";
    else if (i % 3 == 0)  cout << "Fizz\n";
    else if (i % 5 == 0)  cout << "Buzz\n";
    else                  cout << i << '\n';
}
```

The order is load-bearing: the `% 15` test must come first, or `i = 15` prints `Fizz` and never reaches the combined case. This is the cleanest demonstration of why `else if` chains encode priority.

### 13.9 Chessboard colour

```cpp
bool sameColour(int r1, int c1, int r2, int c2) {
    return ((r1 + c1) & 1) == ((r2 + c2) & 1);
}
```

A cell is dark when the parity of `row + column` is fixed. This parity trick reappears in every checkerboard problem.

### 13.10 Validating a date

```cpp
bool validDate(int d, int m, int y) {
    static const int days[] = {0,31,28,31,30,31,30,31,31,30,31,30,31};
    if (m < 1 || m > 12) return false;
    int lim = days[m] + (m == 2 && isLeap(y));
    return d >= 1 && d <= lim;
}
```

Note the lookup table replacing a 12-way `switch`, and the `+ (m == 2 && isLeap(y))` branchless adjustment. Both are patterns to reuse.

---

## Key Takeaways

- Only **zero** is false. `-1`, `'0'`, `0.0001`, and `NaN` are all truthy.
- Always brace `if` bodies. Brace-less `if`s create the dangling-`else` bug and the "second statement always runs" bug.
- `if (x = 5)` compiles; `-Wall` catches it; `if ((x = 5))` documents intent when it is deliberate.
- `&&` and `||` short-circuit left to right — that is what makes `i < n && a[i]` safe. Order is part of correctness.
- **C++ has no chained comparison.** `0 < x < 10` means `(0 < x) < 10` and is true for every positive `x`. Write `0 < x && x < 10`.
- `switch` needs `break`, only accepts integral types, and falls through by default. Grouped `case` labels are the one legitimate use.
- When a branch selects a constant, an array lookup indexed by the value is shorter, faster, and extends more easily.
- `?:` is an expression with very low precedence — parenthesise it inside `cout` chains. Its result type is the common type of both branches, which can silently convert.
- Guard clauses (reject early, return) flatten functions and surface edge cases; do not nest deeper than two levels.
- Grid validity needs all four comparisons joined by `&&`; `||` is the classic bug. Write it once as a `valid()` lambda.
- A mispredicted branch costs ~10–20 cycles. Branchless arithmetic matters in $10^7$-iteration loops, never in $10^4$-iteration ones.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `if (x = 5)` | Assigns instead of compares; always true | `==`; enable `-Wall` |
| 2 | `if (0 < x < 10)` | Parses as `(0 < x) < 10`; true for all `x > 0` | `0 < x && x < 10` |
| 3 | Brace-less `if` with two statements | Second statement always runs | Always use braces |
| 4 | Misindented `else` after nested `if` | Binds to the inner `if` | Braces, `-Wdangling-else` |
| 5 | `if (a[i] == 0 && i < n)` | Reads out of bounds before the guard | Put the bound test first |
| 6 | `if (i < v.size())` with `int i = -1` | Signed→unsigned conversion keeps the test false or true unexpectedly | `i >= 0 && i < (int)v.size()` |
| 7 | `a == b` on doubles | Exact bit comparison; `0.1+0.2 != 0.3` | `fabs(a-b) <= EPS`, scaled |
| 8 | Fixed absolute epsilon for huge values | `1e-9` is meaningless at $10^{12}$ | Relative epsilon, or exact integer maths |
| 9 | Grid guard using `\|\|` | One bad coordinate passes the test | `r >= 0 && r < n && c >= 0 && c < m` |
| 10 | `switch` without `break` | Silent fall-through to the next case | `break`, or `return`, or grouped labels |
| 11 | `switch` on `std::string` | Not an integral type | `if` chain or `map` |
| 12 | `cout << x ? "a" : "b";` | Precedence: compares the stream | Parenthesise the ternary |
| 13 | `x % 2 == 1` for odd check | Negative values give `-1` | `x % 2 != 0` or `(x & 1)` |
| 14 | Triangle check with only one inequality | Misses the other two | All three, with `>` for non-degenerate |
| 15 | FizzBuzz with `% 3` before `% 15` | 15 never reaches the combined branch | Most specific test first |
| 16 | `if (ptr->x)` where `ptr` may be null | Null dereference | `if (ptr && ptr->x)` |
| 17 | `mp[key] == 0` used as a presence test | Inserts a default entry as a side effect | `mp.count(key)` / `mp.find` |
| 18 | Ternary with branches of different signedness | Result narrowed to the smaller type | Make both branches the same type |
| 19 | Missing `default:` in a `switch` over user input | Invalid input silently does nothing | Always add `default` |
| 20 | Deeply nested `if`s | Edge cases hide in the indent | Guard clauses with early `return` |

---

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Leap year, month length, day-of-week from a date | CSES Introductory / AtCoder ABC A | Easy |
| 2 | Classify a triangle as equilateral / isosceles / scalene / invalid | AtCoder ABC A / Codeforces Div. 4 A | Easy |
| 3 | Quadrant and axis classification of a point | Codeforces Div. 4 A | Easy |
| 4 | FizzBuzz and its variants with custom words | AtCoder ABC B | Easy |
| 5 | Grade bands, using both `if` chains and `switch` | Any judge, warm-up | Easy |
| 6 | Chessboard / grid colour and parity problems | Codeforces Div. 3 A | Easy |
| 7 | Validate a date including leap years and month lengths | CSES "Datatypes" style tasks | Easy–Medium |
| 8 | Range containment with `0 <= x < n` bugs hunted deliberately | Self-imposed drill | Easy but instructive |
| 9 | Count cells in a grid whose four neighbours are all valid | Codeforces Div. 3 B | Easy–Medium |
| 10 | Ternary-heavy output formatting (singular/plural labels) | AtCoder ABC A | Easy |
| 11 | Branch-prediction drill: counting with and without `if` in a $2 \times 10^7$ loop | Self-imposed timing exercise | Medium |
| 12 | Simulate a simple state machine with `switch` (traffic light, vending machine) | Codeforces Div. 2 A/B | Medium |
| 13 | Binary search predicate written as a named `bool` function | CSES "Binary Search" set | Medium |

---

## Next Up

→ [Topic 07 — Loops & 'Auto'](Topic%2007%20-%20Loops%20&%20'Auto'.md)

Conditions decide one branch. Loops decide how many times you pay for it — and the multiplication of iteration counts is where complexity estimates come from. Topic 07 covers the three loop forms, the off-by-one that causes most wrong answers, range-for and `auto` (including the two places where `auto` silently changes your type), and the loop patterns — two-pointer, sliding window, digit extraction, grid traversal — that you will reuse for the rest of the book.