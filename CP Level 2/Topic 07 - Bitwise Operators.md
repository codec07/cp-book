# Topic 07 - Bitwise Operators

Every integer is already a string of bits. Bitwise operators let you work on that string directly: one instruction touches 32 or 64 positions at once, so an operation that would otherwise be a loop over an array becomes a single expression.

That is the mechanical reason bits matter. The structural reason is more interesting: **a subset of $n$ items is a number between $0$ and $2^n - 1$**, and set operations become bitwise operations. Once you see that, an exponential search over $2^{20}$ subsets becomes a loop, and states that would need an array of booleans become a single `int`.

This topic covers the operators, the handful of identities that are worth memorising, the three patterns that appear constantly (XOR cancellation, counting per bit, subset enumeration), and the overflow traps that make bit code fail on large inputs.

> **Prerequisites:** [Chapter 1, Topic 04 - Number Systems, Variables and Datatypes](../CPP%20for%20CP/Topic%2004%20-%20Number%20Systems%2C%20Variables%20and%20Datatypes.md) and [Chapter 1, Topic 07 - Loops and 'Auto'](../CPP%20for%20CP/Topic%2007%20-%20Loops%20%26%20%27Auto%27.md)
> **You will be able to:**
> - Read and write the six bitwise operators without hesitating over precedence.
> - Apply the identities `x ^ x = 0`, `x & (x-1)`, and `x & -x` on sight.
> - Count set bits with `__builtin_popcount` and know why it is fast.
> - Replace a per-pair loop with a per-bit loop.
> - Enumerate every subset of $n$ items with one integer loop.
> - Use prefix XOR to answer subarray questions.
> - Avoid the three overflow bugs that break bit code.

---

## 1. Why bits, and where they appear

Three kinds of problem use bitwise operators, and they look unrelated until you see the pattern.

**Real bit manipulation.** Masks, flags, permissions, compression: "set bit 5", "is bit 3 on", "how many bits are set". These are direct uses.

**Algebra that happens to be XOR.** XOR is addition without carries, so it is both associative and self-inverse. Any problem about pairs cancelling, parity, or "find the element that appears once" is an XOR problem in disguise.

**Sets as integers.** If a set has at most 20 elements, encoding it as a bitmask costs one `int`, and union/intersection/membership become `|`, `&`, `>>`. This is the foundation of bitmask DP, which is Level 3, but the enumeration loop itself is Level 2 and appears often.

The distinction matters because the first kind needs care with signedness, the second needs an algebraic eye, and the third needs a bound on $n$ - usually $n \le 20$ or $n \le 25$. Nothing else signals that a bitmask is intended, so the constraint is the signal.

---

## 2. The operators

Assume `int` is 32 bits throughout, and `unsigned` where it matters.

| Operator | Name | Example | Result |
|---|---|---|---|
| `&` | AND | `6 & 3` | `110 & 011 = 010 = 2` |
| `\|` | OR | `6 \| 3` | `110 \| 011 = 111 = 7` |
| `^` | XOR | `6 ^ 3` | `110 ^ 011 = 101 = 5` |
| `~` | NOT | `~6` | flips every bit, including the sign bit |
| `<<` | left shift | `6 << 2` | `24` - multiply by $2^2$ |
| `>>` | right shift | `6 >> 1` | `3` - divide by $2^1$, rounding down |

Two properties decide most of the code you will write:

- **XOR is its own inverse.** `a ^ b ^ b == a`. Every value that appears an even number of times cancels.
- **OR only sets bits, AND only clears them.** `x | y` is never smaller than `x`; `x & y` is never larger than either.

Precedence is where the bugs live. In C++, `&`, `^` and `|` bind **looser** than `==`, `<` and `+`. So

```cpp
if (x & 1 == 0)        // parsed as: x & (1 == 0)  -> always false
if ((x & 1) == 0)      // correct
```

`x >> b & 1` is fine, because `>>` binds tighter than `&`. `x & 1 << b` is also fine, and also confusing - write the parentheses.

---

## 3. The identities worth memorising

Six facts cover almost everything at this level.

| Identity | Meaning |
|---|---|
| `x ^ x = 0` | pairs cancel |
| `x ^ 0 = x` | zero is the identity for XOR |
| `x & (x - 1)` | clears the **lowest set bit** |
| `x & -x` | isolates the **lowest set bit** |
| `x & (x - 1) == 0` | `x` is a power of two (`x > 0`) |
| `x \| (1 << k)` / `x & ~(1 << k)` | set / clear bit `k` |

Why `x & (x - 1)` works: subtracting 1 flips the lowest set bit to 0 and every bit below it to 1. ANDing removes that bit and leaves everything above intact. Repeating the operation counts set bits in time proportional to the number of ones rather than the number of bit positions - which is the historical reason `popcount` was written this way.

Why `x & -x` works: `-x` is the two's-complement negation, which leaves the lowest set bit in place and flips everything else. The AND keeps only that bit. Its value is the largest power of two dividing `x`.

```cpp
int lowbit = x & -x;                     // e.g. x = 12 (1100) -> 4
int withoutLowest = x & (x - 1);         // 12 -> 8
bool isPow2 = x > 0 && (x & (x - 1)) == 0;

int ones = __builtin_popcount(x);        // unsigned int
int ones64 = __builtin_popcountll(x);    // unsigned long long
int trailingZeros = __builtin_ctz(x);    // count of trailing 0 bits (x != 0)
```

The `__builtin_*` family compiles to a single instruction on modern x86 (`popcnt`), so it is not "cheating" - it is faster than any hand-written loop, and it is what you should write in a contest.

---

## 4. Solved Problem 1 - Total Set Bits From 1 to n

> **Statement.** Given $n$, print the total number of 1 bits in the binary representations of all integers from 1 to $n$. $n \le 10^9$.

### Why this approach

Looping over every number is $O(n \log n)$ - about $3 \times 10^{10}$ operations at the limit. Instead, count **per bit position**. For bit $b$, the pattern of that bit in $0, 1, 2, \ldots$ is periodic with period $2^{b+1}$: $2^b$ zeros, then $2^b$ ones, repeating. Counting complete periods and the partial one gives the answer in $O(\log n)$.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    long long n;
    cin >> n;

    long long total = 0;

    for (long long bit = 1; bit <= n; bit <<= 1) {
        long long period = bit << 1;              // 2^(b+1)
        long long full = (n + 1) / period;        // complete periods in 0..n
        long long rem = (n + 1) % period;         // leftover positions
        long long ones = full * bit + max(0LL, rem - bit);
        total += ones;
    }

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
10
```

**Output**
```text
17
```

From 1 to 10 the popcounts are $1, 1, 2, 1, 2, 2, 3, 1, 2, 2$, which sum to 17.

**Complexity.** $O(\log n)$ time, $O(1)$ space.

### The trap

Using `n` instead of `n + 1` in the period arithmetic. The range is $0 \ldots n$, which contains $n + 1$ numbers, and forgetting the extra one shifts every count by a partial period. The error is small - often one or two - so it survives the sample and fails the large test.

The second trap is `bit <<= 1` overflowing. With $n$ near $10^9$ the loop is fine in `long long`; with $n$ near $10^{18}$ the shift passes `2^63` and the loop never ends.

---

## 5. Two more low-bit tricks, and what they are for

`x & (x - 1)` and `x & -x` are not just curiosities; they are the two halves of every "walk the set bits" loop.

```cpp
// visit each set bit, in increasing position
while (x) {
    int b = __builtin_ctz(x);     // position of the lowest set bit
    // ... use bit b ...
    x &= x - 1;                   // clear it
}
```

This runs in time proportional to the number of set bits, which beats scanning all 31 positions whenever the mask is sparse. The complementary loop - visiting zero bits - is the same code on `~x`.

Two direct applications:

- **Power-of-two check.** `x > 0 && (x & (x - 1)) == 0`. The `x > 0` guard matters: zero satisfies the bit test and is not a power of two.
- **`x & -x` as a value.** In a Fenwick tree, the range that index `i` covers is exactly `i & -i`, which is why the data structure works - it is the lowbit, used as a length.

---

## 6. Solved Problem 2 - The Single Number

> **Statement.** Every element of an array appears exactly twice except one, which appears once. Find it. $n$ is odd, $n \le 2 \times 10^5$, values fit in `int`. $O(n)$ time, $O(1)$ extra space.

### Why this approach

XOR is associative and self-inverse, so the order of operations does not matter and every pair cancels:

$$a \oplus a \oplus b \oplus c \oplus b \oplus c = a \oplus (b \oplus b) \oplus (c \oplus c) = a.$$

XOR the whole array and the survivor is left. No hash map, no sorting.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    int ans = 0;
    for (int i = 0; i < n; i++) {
        int x;
        cin >> x;
        ans ^= x;
    }

    cout << ans << '\n';
    return 0;
}
```

**Input**
```text
5
4 1 2 1 2
```

**Output**
```text
4
```

The `1`s cancel and so do the `2`s, leaving `4`.

**Complexity.** $O(n)$ time, $O(1)$ space.

### The trap

Reaching for a frequency array when the values are up to $10^9$. It forces a hash map and turns a one-line solution into a memory-heavy one. The statement "every element appears twice except one" is itself the signal for XOR - and if it says "three times except one", the answer is no longer a single XOR (see unsolved problem 1).

Reading the values inside the loop instead of storing them is worth noticing: the array is never needed, only its XOR.

---

## 7. Counting per bit instead of per pair

A large family of problems asks for a sum over **pairs**. With $n = 2 \times 10^5$ there are $2 \times 10^{10}$ pairs, so enumerating them is out. But if the quantity decomposes over bit positions, each position can be counted independently:

$$\sum_{i<j} \text{popcount}(a_i \oplus a_j) = \sum_{b} (\text{number of pairs differing at bit } b).$$

For a fixed bit, let `ones` be the count of elements with that bit set and `zeros = n - ones`. A pair differs exactly when one member is from each group, so the number of such pairs is `ones * zeros`. Sum over the 31 relevant bits and the job is done in $O(31n)$.

The same decomposition solves "sum of XOR over all pairs", "sum of AND over all pairs" (two set bits: `ones*(ones-1)/2`), and "sum of OR" (`total_pairs - both_zero_pairs`).

---

## 8. Solved Problem 3 - Total Bit Difference

> **Statement.** Given $n$ integers, compute $\sum_{i<j} \text{popcount}(a_i \oplus a_j)$. $n \le 2 \times 10^5$, values $\le 10^9$.

### Why this approach

Per bit, as in §7. Each pair contributes once for every bit where the two numbers differ, so summing `ones * zeros` over all bits gives exactly the total.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    long long total = 0;

    for (int b = 0; b < 31; b++) {
        long long ones = 0;
        for (long long x : a)
            if ((x >> b) & 1) ones++;

        long long zeros = n - ones;
        total += ones * zeros;
    }

    cout << total << '\n';
    return 0;
}
```

**Input**
```text
3
1 3 5
```

**Output**
```text
4
```

Binary: $1 = 001$, $3 = 011$, $5 = 101$. Bit 0 is set in all three (no differing pairs), bit 1 differs in one pair, bit 2 differs in one pair, and $3$ vs $5$ differs in two bits. The total over all three pairs is $0 + 1 + 1 + 2 = 4$.

**Complexity.** $O(31n)$, effectively $O(n)$.

### The trap

`ones * zeros` computed in `int` when $n = 2 \times 10^5$: the product is $10^{10}$ and overflows a 32-bit integer. The result `total` must be `long long`, and `ones` / `zeros` are declared `long long` here for exactly that multiplication.

The second trap is the bit bound. Values up to $10^9$ need 30 bits (bit 30 is $2^{30} \approx 1.07 \times 10^9$), so 31 positions cover it. Reading the constraint and computing `31` from the maximum value rather than hard-coding it is the habit worth forming.

---

## 9. Enumerating subsets with a mask

A mask of $n$ bits represents a subset: bit `i` is 1 if item `i` is included. Then:

| Operation | Expression |
|---|---|
| Empty set | `0` |
| Full set | `(1 << n) - 1` |
| Add item `i` | `mask \| (1 << i)` |
| Remove item `i` | `mask & ~(1 << i)` |
| Test item `i` | `(mask >> i) & 1` |
| Size | `__builtin_popcount(mask)` |
| Union / intersection | `a \| b` / `a & b` |

Enumerating all $2^n$ subsets is one loop:

```cpp
for (int mask = 0; mask < (1 << n); mask++) {
    // mask is one subset
}
```

At $n = 20$ that is about a million iterations - fast. At $n = 25$ it is 33 million - borderline. At $n = 30$ it is a billion - a different algorithm is required. **The constraint $n \le 20$ is the author telling you this loop is intended.**

To iterate the **submasks** of a given mask (all sets contained in it):

```cpp
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // sub is a submask of mask
    if (sub == 0) break;
}
```

This visits $2^{\text{popcount(mask)}}$ values and never leaves the mask. Summed over all masks it is $3^n$, which is why subset-of-subset DP is only viable for $n \le 15$ or so.

---

## 10. Solved Problem 4 - Subsets With a Given XOR

> **Statement.** Given $n$ integers and a target $K$, count the subsets whose XOR equals $K$. The empty subset counts (its XOR is 0). $n \le 20$, values $\le 10^9$.

### Why this approach

$2^n \le 1{,}048{,}576$ subsets, so enumeration is exactly the intended complexity. Each mask names a subset; maintain the XOR incrementally per mask and compare.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    vector<long long> a(n);
    for (auto &x : a) cin >> x;

    long long count = 0;

    for (int mask = 0; mask < (1 << n); mask++) {
        long long x = 0;
        for (int i = 0; i < n; i++)
            if ((mask >> i) & 1) x ^= a[i];

        if (x == k) count++;
    }

    cout << count << '\n';
    return 0;
}
```

**Input**
```text
3 3
1 2 3
```

**Output**
```text
2
```

The subsets $\{3\}$ and $\{1, 2\}$ both have XOR 3. The empty set has XOR 0 and does not count, and $\{1,2,3\}$ has XOR 0 as well.

**Complexity.** $O(n \cdot 2^n)$ time, $O(n)$ space.

### The trap

`(1 << n)` with `n = 31`. Signed left shift past the top bit is undefined behaviour, and at `n = 31` the value is already negative. If the constraint really is $n \le 20$ this cannot happen - which is another reason to read the constraint before writing the loop.

The second trap is the empty set. "Subset" includes it unless the statement says "non-empty", and its XOR is 0, so it inflates the answer whenever $K = 0$.

---

## 11. Solved Problem 5 - Count Subarrays With XOR Equal to K

> **Statement.** Count the contiguous subarrays whose XOR is exactly $K$. $n \le 2 \times 10^5$, values and $K \le 10^9$.

### Why this approach

XOR over a range is the XOR of two prefixes: $\text{xor}(l \ldots r) = P_r \oplus P_{l-1}$, where $P_i = a_1 \oplus \cdots \oplus a_i$. So the condition $P_r \oplus P_{l-1} = K$ becomes $P_{l-1} = P_r \oplus K$.

That is the prefix-sum counting pattern from Topic 01, with XOR in place of addition: sweep `r`, keep a hash map of how many earlier prefixes equal each value, and add the count of `P_r ^ K`. The self-inverse property is what makes the rearrangement legal.

### Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    long long k;
    cin >> n >> k;

    unordered_map<long long, long long> count;
    count[0] = 1;                     // the empty prefix

    long long prefix = 0, answer = 0;

    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        prefix ^= x;

        answer += count[prefix ^ k];
        count[prefix]++;
    }

    cout << answer << '\n';
    return 0;
}
```

**Input**
```text
4 2
4 2 2 6
```

**Output**
```text
3
```

The prefixes are $0, 4, 6, 4, 2$. The qualifying subarrays are $[4, 2, 2, 6]$ (XOR 2), the third element alone, and the second element alone.

**Complexity.** $O(n)$ expected time, $O(n)$ space.

### The trap

Forgetting `count[0] = 1`. Without the empty prefix, any subarray starting at index 0 is invisible, and the answer is short by exactly the number of qualifying prefixes. This is the same bug as in the prefix-sum version in Topic 01, and it is the most common wrong answer on this problem.

The second trap is using `int` for the answer: with $2 \times 10^5$ elements there can be about $2 \times 10^{10}$ subarrays.

---

## 12. Signed numbers, shifts, and overflow

Three rules cover almost every bit-related crash.

**Rule 1: shifting into the sign bit is undefined.** `1 << 31` is not "the value with bit 31 set" - for a signed 32-bit `int` it is undefined behaviour. Use `1LL << k` for $k \ge 31$, and prefer `unsigned int` masks when the top bit is genuinely needed.

**Rule 2: right-shifting a negative number is implementation-defined** before C++20 (arithmetic on every real compiler, but not guaranteed). If you need a logical shift, cast to unsigned first:

```cpp
unsigned u = x;
u >>= 1;
```

Bit-counting code is safest with `unsigned` throughout, which is why `__builtin_popcount` takes an unsigned argument.

**Rule 3: complement flips the sign bit.** `~0` is `-1`, not `2147483647`. `x & ~(1 << k)` is still correct because the high bits of `~(1 << k)` are 1s in a two's-complement representation - but "clear the low bits with `~mask`" idioms that assume a width need an explicit width:

```cpp
unsigned lowBits = mask & ((1u << width) - 1);
```

A related habit: `n & 1` tests parity, and `n >> 1` is `n / 2` only for non-negative `n`. For a negative odd number, `-3 >> 1` is `-2`, while `-3 / 2` is `-1`.

---

## 13. Complexity summary

| Task | Technique | Time |
|---|---|---|
| Popcount of one value | `__builtin_popcount` | $O(1)$ |
| Popcount of $n$ values | builtin per value | $O(n)$ |
| Total set bits $1 \ldots n$ | count per bit | $O(\log n)$ |
| Find the unique element | XOR all | $O(n)$ |
| Sum of XOR over all pairs | count per bit | $O(31n)$ |
| Enumerate all subsets | mask loop | $O(2^n)$ |
| Enumerate submasks of a mask | `(sub-1) & mask` | $O(2^{\text{popcount}})$ |
| Count subarrays with XOR $K$ | prefix XOR + hash map | $O(n)$ expected |

The two bounds to remember: **$2^{20} \approx 10^6$** (safe), **$2^{25} \approx 3.3 \times 10^7$** (borderline), **$2^{30} \approx 10^9$** (too slow). When a statement says $n \le 20$, it is asking for a subset loop.

---

## 14. Unsolved problems

| # | Problem | Constraints |
|---|---|---|
| 1 | **Single Number II.** Every element appears three times except one, which appears once. Find it in $O(n)$ time and $O(1)$ space. | $n \le 2 \times 10^5$ |
| 2 | **Maximum XOR Pair.** Find $\max_{i \ne j} (a_i \oplus a_j)$. | $n \le 2 \times 10^5$, values $< 2^{31}$ |
| 3 | **Gray Code.** Given $n$, output the Gray code sequence of length $2^n$, where consecutive values differ in exactly one bit. | $n \le 16$ |
| 4 | **Divide Without Dividing.** Given integers $a$ and $b$, compute $a / b$ truncated toward zero using only addition, subtraction and bit shifts. | $\lvert a \rvert, \lvert b \rvert \le 2^{31}-1$ |
| 5 | **Range XOR Queries.** Given an array and $q$ queries $(l, r)$, print the XOR of $a_l \ldots a_r$. | $n, q \le 2 \times 10^5$ |

### Hints, in increasing strength

1. Problem 1 is the per-bit idea from §8: count how many elements have each bit set, then reduce that count modulo 3. A bit survives if the remainder is 1.
2. Problem 2 builds the answer one bit at a time from the top. Keep the set of prefixes already seen; for a candidate answer `ans`, test whether some prefix equal to `(candidate ^ prefix)` exists in the set.
3. Problem 3 has a three-line construction: `g(i) = i ^ (i >> 1)`. Verify that consecutive values differ in one bit for $n = 3$ before trusting the pattern.
4. Problem 4 is shift-and-subtract: find the largest shift `k` with `b << k <= a`, subtract, and accumulate `1 << k`. The overflow case is `a = -2^31` with `b = -1`, whose true quotient does not fit in a 32-bit signed integer.
5. Problem 5 is a prefix XOR array and two lookups per query - the additive version of Problem 5 in the solved set, with no counting.

---

## Key Takeaways

- A subset of at most 20 items is an `int`; that is why bitmask problems come with $n \le 20$.
- `&` clears, `\|` sets, `^` toggles, `~` flips, and shifts move bits.
- `&`, `^` and `|` bind looser than `==`; parenthesise every mixed expression.
- `x ^ x = 0` makes XOR the cancellation operator - pairs disappear, the odd one out survives.
- `x & (x - 1)` clears the lowest set bit; loop with it to visit set bits in time proportional to their count.
- `x & -x` isolates the lowest set bit, and is the range length in a Fenwick tree.
- `x > 0 && (x & (x - 1)) == 0` tests for a power of two.
- `__builtin_popcount` is a single instruction; never hand-write the loop in a contest.
- Per-bit counting replaces per-pair enumeration whenever the quantity decomposes over bits.
- `ones * zeros` for bit $b$ counts the pairs differing at that bit, and overflows `int` at $n = 2 \times 10^5$.
- Prefix XOR turns a subarray XOR into a difference of two prefixes, exactly like prefix sums.
- `count[0] = 1` before scanning is mandatory for subarray counting with prefixes.
- `1 << 31` is undefined behaviour; use `1LL << k` above bit 30.
- Right-shifting negatives is implementation-defined before C++20; bit code belongs in `unsigned`.
- $2^{20}$ is fast, $2^{25}$ is borderline, $2^{30}$ is not an option.

---

## Common Mistakes & Pitfalls

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | `x & 1 == 0` | The condition is always false | `(x & 1) == 0` |
| 2 | `x & (x-1) == 0` without `x > 0` | Zero reported as a power of two | Add the guard |
| 3 | Hand-written popcount loop | Slower, and easy to get wrong | `__builtin_popcount` |
| 4 | `1 << 31` | Undefined behaviour, negative results | `1LL << k` |
| 5 | `int` for `ones * zeros` | Overflow at $n = 2 \times 10^5$ | `long long` |
| 6 | `n` instead of `n + 1` in bit counting | Small constant error | $0 \ldots n$ has $n+1$ values |
| 7 | `(1 << n)` with $n = 31$ | Negative loop bound | Check the constraint |
| 8 | Empty subset counted when it should not be | Off-by-one on $K = 0$ | Read the statement |
| 9 | Forgetting `count[0] = 1` in prefix XOR | Subarrays starting at 0 missed | Initialise before the loop |
| 10 | `int` answer for subarray counts | Overflow | `long long` |
| 11 | Right-shifting a negative number | Implementation-defined | Cast to `unsigned` |
| 12 | Assuming `~0` is a large positive number | Sign surprise | It is `-1` |
| 13 | Loop `bit <<= 1` without a bound | Infinite loop at 64-bit edges | Bound by `bit <= n` |
| 14 | Sorting or hashing when XOR cancels | Slow and unnecessary | XOR the array |

---

## Practice Problems

The five unsolved problems above, in a sensible order:

| Order | Problem | Why first |
|---|---|---|
| 1 | 5 - Range XOR Queries | Prefix XOR with no counting; pure construction |
| 2 | 3 - Gray Code | One formula, verified on small $n$ |
| 3 | 1 - Single Number II | Per-bit counting with a modulus |
| 4 | 4 - Divide Without Dividing | Shift-and-subtract, plus overflow care |
| 5 | 2 - Maximum XOR Pair | Greedy over bit prefixes; the hardest idea here |

For more, the CSES *Additional Problems* section contains *Xor Pyramid* and *Maximum Xor Subarray*, both of which are prefix-XOR exercises. The *Bitwise Operators* problems in most judge archives are exactly the four patterns in this topic.

---

## Next Up

→ [Topic 08 - Level 2 Solved Problem Set](Topic%2008%20-%20Level%202%20Solved%20Problem%20Set.md)

That closes the technique chapters of Level 2: searching and sorting, graphs, trees and bits. The last two topics are the problem sets - twenty solved problems that each need one of these techniques and say nothing about which, then fifty unsolved ones to work through on your own.
