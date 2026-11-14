# Topic 05 — String Manipulation Methods & Statements

Strings are the second most common input type in competitive programming after integers, and the most common source of silent quadratic time. A `substr` inside a loop, or an `erase` at the front, turns an $\mathcal{O}(n)$ algorithm into $\mathcal{O}(n^2)$ with no warning and no compiler error. This chapter covers `std::string` end to end: every method you will actually use, what each one costs, the `npos` trap, the `char` versus `char&` trap, and the parsing idioms that replace hand-rolled input code.

> **Prerequisites:** [Topic 04 — Number Systems, Variables and Datatypes](Topic%2004%20-%20Number%20Systems%2C%20Variables%20and%20Datatypes.md)
> **You will be able to:**
> - Use every `std::string` method you need in a contest, and know its complexity.
> - Avoid the `string::npos` and index-underflow traps.
> - Parse mixed input with `stringstream` and split lines on any delimiter.
> - Build strings of length $10^6$ without hitting quadratic behaviour.
> - Write six standard string recipes from memory: reverse words, count vowels, toggle case, run-length encode, rotation check, split.

---

## 1. `std::string` versus C-style `char[]`

A C string is a `char` array ending in `'\0'`. It carries no length, so every operation must scan for the terminator, and nothing stops you writing past the end. `std::string` is a class that owns a heap buffer, stores the length, and reallocates as needed.

| Feature | `std::string` | `char[]` |
|---|---|---|
| Length | Stored; `size()` is $\mathcal{O}(1)$ | `strlen` rescans, $\mathcal{O}(n)$ |
| Growth | Automatic, amortised $\mathcal{O}(1)$ per char | Fixed; overflow is UB |
| Assignment | `a = b` copies correctly | `strcpy`/`strncpy`, easy to get wrong |
| Comparison | `a < b` is lexicographic | `strcmp(a, b) < 0` |
| Memory | Heap (small-string optimisation for short values) | Stack or global |
| Speed on tiny strings | Slightly slower | Slightly faster |
| Used by | 95% of CP code | `scanf("%s")`, legacy APIs |

Use `std::string` unless you have a measured reason not to. The one place C strings still win is `scanf("%s", buf)` with a pre-sized buffer under a tight time limit — and even that is usually unnecessary once `cin` sync is off.

## 2. Creating a string

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string a;                       // empty
    string b = "hello";             // from a literal
    string c("hello");              // direct init
    string d(5, 'x');               // "xxxxx"
    string e(b);                    // copy
    string f(b, 1, 3);              // "ell"  (pos, len)
    string g = b + " world";        // concatenation
    string h(3, 'a');               // "aaa"

    cout << a.size() << " " << d << " " << f << " " << h << "\n";
    return 0;
}
```

```text
0 xxxxx ell aaa
```

`string d(5, 'x')` is the "repeat a character" constructor and is genuinely useful for building separators or padding. `string(b, 1, 3)` copies a slice without an intermediate temporary.

## 3. Accessing characters

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "codeforces";
    cout << s[0] << s[4] << " " << s.at(1) << " " << s.front() << s.back() << "\n";
    s[0] = 'C';                                    // unchecked, O(1)
    cout << s << "\n";                             // Codeforces
    try { cout << s.at(100); }                     // throws
    catch (const out_of_range&) { cout << "out_of_range\n"; }
    return 0;
}
```

```text
cf o ce
Codeforces
out_of_range
```

| Access | Bounds-checked | Cost | When |
|---|---|---|---|
| `s[i]` | No — UB past `size()` | $\mathcal{O}(1)$ | Inside loops where `i` is provably valid |
| `s.at(i)` | Yes, throws `out_of_range` | $\mathcal{O}(1)$ + check | Debugging, or untrusted indices |
| `s.front()` | No, UB when empty | $\mathcal{O}(1)$ | You know it is non-empty |
| `s.back()` | No, UB when empty | $\mathcal{O}(1)$ | Same |

In a contest, `s[i]` is correct and `at()` costs a branch. Use `s[i]`, but guard the index yourself. Note that `s[s.size()]` is legal and returns the null character in C++11 and later; anything beyond that is UB.

## 4. Size, length and emptiness

```cpp
string s = "abc";
cout << s.size() << " " << s.length() << " " << s.empty() << "\n";  // 3 3 0
```

`size()` and `length()` are synonyms, both $\mathcal{O}(1)$. `empty()` is clearer than `size() == 0` and equally fast.

The type is the trap: `s.size()` returns `size_t`, an **unsigned** 64-bit value. `s.size() - 1` on an empty string is `18446744073709551615`, not `-1`. Cast before arithmetic:

```cpp
for (int i = 0; i + 1 < (int)s.size(); ++i) { /* s[i], s[i+1] */ }
```

## 5. Growing a string

```cpp
string s = "ab";
s += 'c';              // abc
s += "de";             // abcde
s.push_back('f');      // abcdef
s.append(3, '!');      // abcdef!!!
s.pop_back();          // abcdef!!
```

*(fragment — `s` is a `std::string`)*

| Operation | Complexity | Note |
|---|---|---|
| `s += c` | amortised $\mathcal{O}(1)$ | May reallocate; doubling strategy |
| `s.push_back(c)` | amortised $\mathcal{O}(1)$ | Identical to `+=` |
| `s += t` | $\mathcal{O}(\lvert t \rvert)$ | Copies `t`'s characters |
| `s.append(t)` | $\mathcal{O}(\lvert t \rvert)$ | Same as `+=` |
| `s = s + t` | $\mathcal{O}(\lvert s \rvert + \lvert t \rvert)$ | Allocates a **new** buffer, copies both |
| `s.pop_back()` | $\mathcal{O}(1)$ | Undefined on an empty string |

The difference between `s += t` and `s = s + t` is the whole game in a loop: the first appends in place (amortised constant), the second allocates and copies the entire string every iteration. That is the difference between $\mathcal{O}(n)$ and $\mathcal{O}(n^2)$.

One compile-time trap:

```cpp
string bad  = "hello" + " world";          // ERROR: two pointers added
string good = string("hello") + " world";  // fine
```

Two string *literals* are arrays of `char`; adding them is pointer arithmetic, which the compiler rejects.

## 6. `substr(pos, len)`

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "competitive programming";
    cout << s.substr(12) << "\n";        // programming   (to the end)
    cout << s.substr(0, 11) << "\n";     // competitive
    cout << s.substr(5, 3) << "\n";      // tit
    cout << s.substr(s.size()).size() << "\n";   // 0  (pos == size() is legal)
    return 0;
}
```

```text
programming
competitive
tit
0
```

`substr` allocates a new string and copies `len` characters: $\mathcal{O}(\text{len})$, *not* $\mathcal{O}(1)$. That means `s.substr(i, 1)` inside a loop over $n$ is $\mathcal{O}(n)$ overall and fine, but `s.substr(i)` inside a loop is $\mathcal{O}(n^2)$ and fatal at $n = 10^5$.

If you only need to *look* at a slice, pass indices instead of building a string. Algorithms like Z-function and suffix arrays are built precisely to avoid `substr` in loops.

## 7. Searching: `find`, `rfind`, `find_first_of`, and `npos`

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "competitive programming";

    size_t p = s.find("program");        // 12
    cout << p << "\n";

    cout << s.find('m') << " " << s.rfind('m') << "\n";   // 2 19
    cout << s.find_first_of("aeiou") << "\n";              // 1  ('o')
    cout << s.find("zzz") << "\n";                         // npos, huge number

    if (s.find("program") != string::npos) cout << "found\n";
    return 0;
}
```

```text
12
2 19
1
18446744073709551615
found
```

`string::npos` is defined as `size_t(-1)`, i.e. `18446744073709551615`. It is **not** `-1` and it is **not** falsy-safe.

```cpp
if (s.find("comp"))          // BUG: returns 0 when found at position 0 -> false
if (s.find("comp") != string::npos)   // correct
```

This is the single most common string bug in competitive programming. A match at index 0 makes the `if` fail, so your code reports "not found" exactly when the pattern is at the start.

| Method | Finds | Complexity |
|---|---|---|
| `find(str)` | First occurrence of a substring | $\mathcal{O}(n \cdot m)$ worst case, $\mathcal{O}(n)$ typical |
| `find(ch)` | First occurrence of a character | $\mathcal{O}(n)$ |
| `rfind(str)` | Last occurrence | $\mathcal{O}(n \cdot m)$ |
| `find_first_of(set)` | First char that appears in `set` | $\mathcal{O}(n \cdot \lvert set \rvert)$ |
| `find_last_of(set)` | Last char in `set` | $\mathcal{O}(n \cdot \lvert set \rvert)$ |
| `find_first_not_of(set)` | First char **not** in `set` | $\mathcal{O}(n \cdot \lvert set \rvert)$ |

Storing the result in an `int` truncates the 64-bit `npos`, so always compare against `npos` before narrowing.

## 8. `insert`, `erase`, `replace` — the quadratic trap

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "abcdef";
    s.insert(2, "XY");          // abXYcdef
    s.erase(1, 3);              // acdef      (removes bXY)
    s.replace(0, 2, "ZZ");      // ZZdef
    cout << s << "\n";
    s.insert(s.begin(), '>');   // >ZZdef     (iterator overload)
    cout << s << "\n";
    return 0;
}
```

```text
ZZdef
>ZZdef
```

All three are $\mathcal{O}(n)$: everything after the edit point shifts. That is fine once, lethal in a loop.

```cpp
// QUADRATIC: ~n^2/2 character moves. At n = 1e5 that is 5e9 operations -> TLE.
string s;
for (int i = 0; i < 100000; ++i) s = char('a' + i % 26) + s;

// LINEAR: append, then reverse once at the end.
string t;
t.reserve(100000);
for (int i = 0; i < 100000; ++i) t.push_back(char('a' + i % 26));
reverse(t.begin(), t.end());
```

*Contest story: an "insert each character at the front" simulation looked like $\mathcal{O}(n)$ because each step is one character. At $n = 10^5$ it copied 5 billion bytes — 1.8 seconds over the limit. Reversing the build order fixed it with a two-line change.*

The same trap appears with `erase` in a loop over a string, e.g. removing all vowels one by one. Build a new string instead:

```cpp
string out;
out.reserve(s.size());
for (char c : s) if (!strchr("aeiouAEIOU", c)) out.push_back(c);
```

## 9. `compare` and lexicographic order

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string a = "apple", b = "apply";
    cout << (a < b) << " " << (a == b) << "\n";        // 1 0
    cout << (a.compare(b) < 0) << "\n";                 // 1  (differs at the last char)

    string x = "Zebra", y = "apple";
    cout << (x < y) << "\n";                            // 1  -> 'Z'(90) < 'a'(97)
    cout << (x.compare(0, 1, "A") > 0) << "\n";         // 1  'Z' > 'A'
    return 0;
}
```

```text
1 0
1
1
1
```

Lexicographic order compares character codes, not dictionary order. Uppercase letters all sit below lowercase, so `"Zebra" < "apple"` is true. Consequences you will meet in contests:

- Sorting strings puts `"ABC"` before `"abc"`, and `"Z"` before `"a"`.
- A prefix is smaller: `"abc" < "abcd"`.
- Comparing `const char*` with `<` compares **addresses**, not contents. `"abc" < "abd"` is not a string comparison — build `std::string` values first.

`compare` returns a negative number, zero, or a positive number — never assume it is exactly `-1` or `1`.

## 10. Reversing

```cpp
string s = "abcdef";
reverse(s.begin(), s.end());        // fedcba, in place
string t(s.rbegin(), s.rend());     // reversed copy, s unchanged
```

`reverse` is $\mathcal{O}(n)$ and in-place, from `<algorithm>`. The iterator-pair constructor is the clean way to get a reversed copy without touching the original — handy for palindrome tests: `s == string(s.rbegin(), s.rend())`.

## 11. Character classification and case conversion

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << (bool)isalpha('a') << (bool)isdigit('7') << (bool)isalnum('!') << "\n";  // 110
    cout << (bool)isspace(' ') << (bool)isupper('G') << "\n";                        // 11
    cout << char(tolower('Q')) << char(toupper('q')) << "\n";                        // qQ

    char c = 'G';                                    // manual, no function call
    cout << char(c - 'A' + 'a') << " " << char(c ^ 32) << "\n";   // g g
    return 0;
}
```

`isalpha` and friends return **non-zero**, not necessarily `1` — on glibc `isalpha('a')` is 1024 — so a test like `if (isalpha(c) == 1)` fails. Always use them in a boolean context.

| Function | True when | Notes |
|---|---|---|
| `isalpha(c)` | Letter | `a–z` or `A–Z` |
| `isdigit(c)` | `'0'`–`'9'` | Not `isnumber` |
| `isalnum(c)` | Letter or digit | |
| `isspace(c)` | Space, `\t`, `\n`, `\v`, `\f`, `\r` | |
| `isupper(c)` / `islower(c)` | Case | |
| `tolower(c)` / `toupper(c)` | Returns the converted char | Returns `int`; cast back |
| `c ^ 32` | Toggle case | **Only** valid for letters |
| `c - 'a' + 'A'` | Lower to upper | Assumes `c` is lowercase |

Two hazards. First, these functions take an `int` and are UB for negative values other than `EOF`, so cast: `isalpha((unsigned char)c)`. Second, `c ^ 32` on a non-letter produces garbage (`'0' ^ 32 == 'P'`), so guard with `isalpha` first.

## 12. Numbers ↔ strings: `to_string`, `stoi`, `stoll`, `stod`

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    cout << to_string(42) << " " << to_string(-7LL) << " " << to_string(2.5) << "\n";

    cout << stoi("  42abc") << "\n";          // 42  (skips whitespace, stops at 'a')
    cout << stoi("-17") << "\n";              // -17
    cout << stoi("ff", nullptr, 16) << "\n";  // 255 (explicit base)
    cout << stoll("9223372036854775807") << "\n";
    cout << stod("3.14159") << "\n";

    try { stoi("abc"); }
    catch (const invalid_argument&) { cout << "invalid\n"; }

    try { stoi("99999999999"); }
    catch (const out_of_range&) { cout << "out_of_range\n"; }
    return 0;
}
```

```text
42 -7 2.500000
42
-17
255
9223372036854775807
3.14159
invalid
out_of_range
```

| Call | Returns | Throws |
|---|---|---|
| `stoi(s)` | `int` | `invalid_argument`, `out_of_range` |
| `stol(s)` | `long` | same |
| `stoll(s)` | `long long` | same |
| `stoul`, `stoull` | unsigned | same |
| `stof`, `stod`, `stold` | `float`, `double`, `long double` | same |

Behaviour details worth knowing:

- **Leading whitespace is skipped.** `stoi("   5")` is `5`, but `stoi("5 ")` also returns `5` (it stops at the space).
- **Trailing junk is ignored.** `stoi("12abc")` is `12`. If you need the whole string to be a number, pass a `size_t pos` and check `pos == s.size()`:

```cpp
size_t pos = 0;
int v = stoi(s, &pos);
bool clean = (pos == s.size());
```

- **Empty or non-numeric input throws.** In a contest, an uncaught exception means a runtime error verdict, so validate with `isdigit` when input format is uncertain.
- `to_string(double)` prints with 6 decimals by default and does no rounding control; use `stringstream` or `printf` when the format matters.

### Converting by hand

```cpp
#include <bits/stdc++.h>
using namespace std;

string intToStr(long long x) {
    if (x == 0) return "0";
    bool neg = x < 0;
    if (neg) x = -x;
    string s;
    while (x > 0) { s.push_back(char('0' + x % 10)); x /= 10; }
    if (neg) s.push_back('-');
    reverse(s.begin(), s.end());
    return s;
}

long long strToInt(const string& s) {
    long long v = 0;
    for (char c : s) if (c >= '0' && c <= '9') v = v * 10 + (c - '0');
    return (!s.empty() && s[0] == '-') ? -v : v;
}

int main() {
    cout << intToStr(-1234) << " " << strToInt("-9876") << "\n";
    return 0;
}
```

```text
-1234 -9876
```

In contests, hand conversion is worth writing when you parse $10^6$ numbers: `getchar_unlocked` plus a digit loop beats `cin >>` and beats `stoi` on substrings, because it avoids building the string at all.

## 13. `stringstream`: parsing and splitting

`stringstream` is the general-purpose tokeniser. `>>` skips whitespace and converts to the target type; `getline(ss, tok, delim)` splits on any character.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    // (a) parse a mixed-type record
    string line = "Aarav 17 92.5";
    istringstream iss(line);
    string name; int age; double marks;
    iss >> name >> age >> marks;
    cout << name << " " << age << " " << marks << "\n";

    // (b) split on a delimiter
    string csv = "10,20,30,40";
    stringstream ss(csv);
    string tok;
    int sum = 0;
    while (getline(ss, tok, ',')) sum += stoi(tok);
    cout << sum << "\n";

    // (c) split a whole line into words (any whitespace)
    string sentence = "the quick  brown\tfox";
    istringstream words(sentence);
    string w;
    vector<string> all;
    while (words >> w) all.push_back(w);
    cout << all.size() << " " << all[2] << "\n";
    return 0;
}
```

```text
Aarav 17 92.5
100
4 brown
```

| Pattern | Code |
|---|---|
| Split on whitespace | `while (ss >> tok)` |
| Split on one char | `while (getline(ss, tok, ','))` |
| Parse a typed record | `iss >> a >> b >> c` |
| Reuse a stream | `ss.clear(); ss.str(newData);` |

Cost: a `stringstream` is roughly 3–10× slower than `scanf`, so do not put one inside a loop over $10^6$ elements. Use it for line-shaped input (a few thousand tokens) and use fast readers for bulk numbers — see [Topic 03](Topic%2003%20-%20Input%2C%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20%28VERY%20IMPORTANT%29.md). Also note `getline` drops a trailing empty field: `"1,2,"` yields two tokens, not three.

## 14. Iterating: `char` versus `char&`

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "hello";
    for (char c : s) c = toupper(c);              // BUG: c is a copy
    for (char& c : s) c = toupper(c);             // correct: reference
    cout << s << "\n";                             // HELLO

    for (int i = 0; i < (int)s.size(); ++i) s[i] = tolower(s[i]);
    cout << s << "\n";                             // hello
    return 0;
}
```

```text
HELLO
hello
```

`for (char c : s)` copies each character; modifying `c` changes the copy. Use `char&` to mutate, and `const char&` (or plain `char c`) to read. The same applies to `auto`: `for (auto c : s)` copies a `char`, `for (auto& c : s)` aliases it.

Iterators work too, and some algorithms demand them: `sort(s.begin(), s.end())` sorts the characters, `s.begin() + k` is a valid position for `insert`/`erase`.

## 15. Performance: amortised append, `reserve`, and $10^6$ characters

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    const int N = 1000000;

    string a;                       // no reserve: ~20 reallocations, each copying everything
    a.reserve(N);                   // one allocation, guaranteed capacity
    for (int i = 0; i < N; ++i) a += char('a' + i % 26);
    cout << a.size() << " " << a.capacity() << "\n";

    string b;
    for (int i = 0; i < N; ++i) b = b + char('a' + i % 26);   // O(n^2): never do this
    cout << b.size() << "\n";
    return 0;
}
```

```text
1000000 1000000
1000000
```

| Pattern | Cost for $n = 10^6$ |
|---|---|
| `s += c` with growth | $\mathcal{O}(n)$ total, ~20 reallocations |
| `s += c` after `reserve(n)` | $\mathcal{O}(n)$ total, 1 allocation |
| `s = s + c` | $\mathcal{O}(n^2)$ — about $5 \times 10^{11}$ byte copies |
| `s.push_back(c)` | Same as `+=` |

`reserve` does not change `size()`, only `capacity()`; it never shrinks the string. Use it when you know the final length — building an output string for $10^6$ answers is the standard case.

## 16. Reading strings: `cin >>` versus `getline`

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    int n;
    cin >> n;                       // leaves '\n' in the buffer

    string s;
    getline(cin, s);                // reads the leftover empty line  -> s == ""
    cout << "[" << s << "]\n";

    getline(cin, s);                // now reads the real line
    cout << "[" << s << "]\n";

    string word;
    cin >> word;                    // stops at whitespace
    cout << "[" << word << "]\n";
    return 0;
}
```

For input `3\nhello world\nfoo bar`:

```text
[]
[hello world]
[foo]
```

| Reader | Stops at | Keeps spaces | Use for |
|---|---|---|---|
| `cin >> s` | Any whitespace | No | Single tokens |
| `getline(cin, s)` | `'\n'` (consumed, not stored) | Yes | Whole lines |
| `scanf("%s", buf)` | Whitespace | No | C buffers, unbounded — dangerous |
| `cin.getline(buf, n)` | `'\n'` or `n-1` chars | Yes | Fixed buffers |

**The classic bug:** `cin >> n` leaves the newline, so the next `getline` returns an empty string. Fix it with `cin.ignore()` (or `cin.ignore(numeric_limits<streamsize>::max(), '\n')` to be safe) before the first `getline`. This single omission causes more wrong answers on line-based problems than any algorithm.

## 17. Frequency arrays, palindromes, anagrams

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "anagram";

    int freq[26] = {};
    for (char c : s) if (c >= 'a' && c <= 'z') ++freq[c - 'a'];
    cout << freq[0] << freq[13] << "\n";        // 3 (a) 1 (n)

    // palindrome, two pointers
    auto isPal = [](const string& t) {
        int i = 0, j = (int)t.size() - 1;
        while (i < j) if (t[i++] != t[j--]) return false;
        return true;
    };
    cout << isPal("racecar") << isPal("abca") << "\n";   // 10

    // anagram: same multiset of characters
    auto isAnagram = [](const string& a, const string& b) {
        if (a.size() != b.size()) return false;
        int f[26] = {};
        for (char c : a) ++f[c - 'a'];
        for (char c : b) if (--f[c - 'a'] < 0) return false;
        return true;
    };
    cout << isAnagram("listen", "silent") << "\n";        // 1
    return 0;
}
```

```text
31
10
1
```

The `int[26]` array is the canonical string tool: $\mathcal{O}(1)$ update, $\mathcal{O}(26)$ scan, no hashing overhead. For arbitrary byte values use `int[256]`. Palindromes are $\mathcal{O}(n)$ with two pointers and $\mathcal{O}(1)$ extra space; the `s == reversed(s)` version is also $\mathcal{O}(n)$ but allocates.

## 18. String hashing (teaser)

Comparing two substrings costs $\mathcal{O}(\text{len})$, so "are these two ranges of a string equal?" done $q$ times is $\mathcal{O}(nq)$. Hashing fixes this: assign each prefix a value $h[i] = (h[i-1] \cdot B + s[i]) \bmod M$ with a large base $B$ and prime $M \approx 10^9+7$, then any substring's hash is a difference of two prefix hashes, giving $\mathcal{O}(1)$ comparison after $\mathcal{O}(n)$ preprocessing. Collisions are possible, so contest code usually uses two moduli or a 64-bit unsigned hash; the full treatment belongs with the string-algorithms material, not here.

## 19. C-string pitfalls

A C string is `char buf[N]` plus a `'\0'`. `char word[] = "hi";` occupies **3** bytes, not 2, so a buffer for a string of length $n$ needs $n+1$ slots. `cin >> buf` has no limit — write `cin >> setw(sizeof(buf)) >> buf`, or use `std::string`.

| Pitfall | Why | Fix |
|---|---|---|
| Buffer too small | No null terminator left | Allocate `len + 1` |
| `scanf("%s", s)` | No width limit | `scanf("%7s", s)` or use `std::string` |
| `strlen` in a loop condition | $\mathcal{O}(n)$ per iteration | Compute once |
| `strcpy` overflow | No bound | `strncpy`/`snprintf` |
| Forgetting `'\0'` after manual fill | String runs past the end | Set `s[n] = '\0'` |
| `s == "abc"` on `char*` | Compares pointers | `strcmp(s, "abc") == 0` |

`char s[10]; cin >> s;` with input `abcdefghijkl` writes past the array — undefined behaviour that often passes locally and crashes on the judge.

## 20. Method complexity cheat-sheet

| Method | Complexity | Returns | Gotcha |
|---|---|---|---|
| `s[i]` | $\mathcal{O}(1)$ | `char&` | No bounds check |
| `s.at(i)` | $\mathcal{O}(1)$ | `char&` | Throws |
| `s.front()`, `s.back()` | $\mathcal{O}(1)$ | `char&` | UB when empty |
| `s.size()`, `s.length()` | $\mathcal{O}(1)$ | `size_t` | Unsigned |
| `s.empty()` | $\mathcal{O}(1)$ | `bool` | |
| `s.clear()` | $\mathcal{O}(1)$ | `void` | Size becomes 0, capacity stays |
| `s.push_back(c)` / `s += c` | amortised $\mathcal{O}(1)$ | | |
| `s += t` | $\mathcal{O}(\lvert t \rvert)$ | | |
| `s + t` | $\mathcal{O}(\lvert s \rvert + \lvert t \rvert)$ | new string | |
| `s.substr(p, l)` | $\mathcal{O}(l)$ | new string | Not free |
| `s.find(t)` | $\mathcal{O}(n \cdot m)$ worst | `size_t` | Compare with `npos` |
| `s.insert(p, t)` | $\mathcal{O}(n + \lvert t \rvert)$ | | Shifts the tail |
| `s.erase(p, l)` | $\mathcal{O}(n)$ | | Shifts the tail |
| `s.replace(p, l, t)` | $\mathcal{O}(n + \lvert t \rvert)$ | | |
| `s.compare(t)` | $\mathcal{O}(\min)$ | `int` | Sign only |
| `reverse(s.begin(), s.end())` | $\mathcal{O}(n)$ | `void` | Mutates |
| `sort(s.begin(), s.end())` | $\mathcal{O}(n \log n)$ | `void` | Mutates |
| `to_string(x)` | $\mathcal{O}(\text{digits})$ | `string` | 6 decimals for double |
| `stoi(s)` | $\mathcal{O}(\text{len})$ | `int` | Throws |
| `s.reserve(n)` | $\mathcal{O}(n)$ at most | `void` | Capacity only |

## 21. Six recipes you should know cold

```cpp
#include <bits/stdc++.h>
using namespace std;

string reverseWords(const string& s) {                 // 1
    istringstream iss(s); vector<string> w; string t;
    while (iss >> t) w.push_back(t);
    string out;
    for (int i = (int)w.size() - 1; i >= 0; --i) out += w[i] + (i ? " " : "");
    return out;
}

int countVowels(const string& s) {                     // 2
    int c = 0;
    for (char ch : s) { char l = (char)tolower((unsigned char)ch);
                        c += (l == 'a' || l == 'e' || l == 'i' || l == 'o' || l == 'u'); }
    return c;
}

string toggleCase(string s) {                          // 3
    for (char& c : s) {
        if (islower((unsigned char)c)) c = (char)toupper((unsigned char)c);
        else if (isupper((unsigned char)c)) c = (char)tolower((unsigned char)c);
    }
    return s;
}

string rle(const string& s) {                          // 4
    string out;
    for (int i = 0, n = (int)s.size(); i < n; ) {
        int j = i;
        while (j < n && s[j] == s[i]) ++j;
        out += s[i] + to_string(j - i);
        i = j;
    }
    return out;
}

bool isRotation(const string& a, const string& b) {    // 5
    return a.size() == b.size() && (a + a).find(b) != string::npos;
}

vector<string> split(const string& s, char d) {        // 6
    vector<string> out; string cur; istringstream iss(s);
    while (getline(iss, cur, d)) out.push_back(cur);
    return out;
}

int main() {
    cout << reverseWords("one two three") << "\n" << countVowels("competitive") << "\n";
    cout << toggleCase("HeLLo 42") << "\n" << rle("aaabbc") << "\n";
    cout << isRotation("abcde", "cdeab") << isRotation("abc", "acb") << "\n";
    for (const string& p : split("a,b,c", ',')) cout << p << "|";
    cout << "\n";
    return 0;
}
```

```text
three two one
5
hEllO 42
a3b2c1
10
a|b|c|
```

Each recipe is $\mathcal{O}(n)$ except `split` and `reverseWords`, which are $\mathcal{O}(n)$ plus allocation. The rotation trick `(a + a).find(b)` is $\mathcal{O}(n^2)$ worst case but $\mathcal{O}(n)$ in practice for random input, and it is short enough to write during a contest.

---

## Key Takeaways

- `s.size()` is $\mathcal{O}(1)$ but returns unsigned `size_t`; cast to `int` before subtracting.
- `s += c` is amortised $\mathcal{O}(1)$; `s = s + c` is $\mathcal{O}(n)$ and turns loops quadratic.
- `substr`, `insert`, `erase` and `replace` are all $\mathcal{O}(n)$ — never call them per element of a large string.
- Always compare `find` results with `string::npos`; a match at index 0 is falsy.
- Both `s.find(...)` and `npos` are `size_t`; narrowing to `int` loses information.
- Use `for (char& c : s)` to modify characters; `char c` is a copy.
- `isalpha` and friends return non-zero, not `1`, and are UB on negative `char` — cast to `unsigned char`.
- `stoi` skips leading whitespace, ignores trailing junk, and throws `invalid_argument` / `out_of_range`.
- `cin >> n` leaves the newline behind; call `cin.ignore()` before the next `getline`.
- `reserve(n)` plus `push_back` is the fast way to build a long string.
- Two string literals cannot be concatenated with `+`; make one a `std::string` first.
- Lexicographic order follows ASCII: `'Z' < 'a'`, so `"Zebra" < "apple"`.

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | `if (s.find(x))` | Match at index 0 returns 0 | `!= string::npos` |
| 2 | `s = c + s` in a loop | $\mathcal{O}(n^2)$ copies | `push_back`, reverse at the end |
| 3 | `s.substr(i)` inside a loop | Each call copies the whole tail | Pass indices, avoid `substr` |
| 4 | `for (char c : s) c = toupper(c);` | `c` is a copy | `char&` |
| 5 | `s.size() - 1` when `s` is empty | Unsigned underflow | Check `empty()` first |
| 6 | `getline` right after `cin >> n` | Newline still in the buffer | `cin.ignore()` |
| 7 | `strlen(s)` in a loop condition | $\mathcal{O}(n)$ per iteration | `int n = strlen(s)` once |
| 8 | `char buf[10]; cin >> buf;` | Buffer overflow | `setw` or `std::string` |
| 9 | `if (isalpha(c) == 1)` | Returns 1024 on glibc | Test truthiness |
| 10 | `"abc" + "def"` | Pointer arithmetic on literals | `string("abc") + "def"` |
| 11 | `int p = s.find(...)` | `npos` truncated to -1 ambiguously | Keep `size_t`, check `npos` |
| 12 | `stoi` with no try/catch | Uncaught exception → runtime error | Validate input first |
| 13 | `s.erase(i, 1)` for each unwanted char | $\mathcal{O}(n^2)$ | Build a filtered copy |
| 14 | Assuming `"apple" < "Banana"` | ASCII, not dictionary, order | Compare lowercase or use custom cmp |

## Practice Problems

| # | Problem type | Judge / source | Difficulty |
|---|---|---|---|
| 1 | Reverse a string / check palindrome | Any judge, warm-up | Easy |
| 2 | Count vowels, consonants, digits, spaces | Codeforces Div. 2 A | Easy |
| 3 | Toggle case of a line | CSES introductory / school level | Easy |
| 4 | Anagram detection for $n \le 10^5$ | Codeforces Div. 2 A | Easy |
| 5 | Find all occurrences of a pattern (naive then KMP) | CSES "String Matching" | Medium |
| 6 | Run-length encode and decode | AtCoder ABC B | Medium |
| 7 | Check whether one string is a rotation of another | Interview classic | Medium |
| 8 | Split and parse CSV-like input with `stringstream` | Codeforces Div. 2 B | Medium |
| 9 | Longest palindromic substring ($\mathcal{O}(n^2)$ then Manacher) | CSES "Palindrome Queries" family | Hard |
| 10 | Substring equality queries with rolling hash | Codeforces Div. 2 D / CSES | Hard |

## Next Up

→ [Topic 06 — Conditional Statements (if, else, else if, ternary)](Topic%2006%20-%20Conditional%20Statements%20%28If%2C%20else%2C%20else%20if%2C%20ternary%20operators%29.md)