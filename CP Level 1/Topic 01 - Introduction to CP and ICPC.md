# Topic 01 — Introduction to CP and ICPC

Competitive programming is a sport with a compiler. You get a problem statement, a time limit, and a judge that runs your program against inputs you cannot see — and the whole skill is producing a correct, fast solution under pressure. Before you learn a single algorithm, you need to know the rules of the game: how contests are scored, what the verdicts mean, how difficulty is rated, and how to practise so that six months of effort actually compounds.

This chapter is the orientation. It will not make you faster today, but it will stop you from wasting the first three months.

> **Prerequisites:** [Chapter 1 — CPP for CP](../CPP%20for%20CP/README.md) (you can compile, debug and use the STL)
> **You will be able to:**
> - Explain how ICPC and Codeforces differ in format, scoring and pressure.
> - Read a constraints section and predict the intended complexity before coding.
> - Decode every verdict a judge can return, and know which one your bug is.
> - Run a practice regime with virtual contests, upsolving and ladders.
> - Read a problem statement the way a setter wrote it.

---

## 1. What competitive programming actually is

A contest gives you $8$–$13$ problems and $2$–$5$ hours. Each problem specifies an input format, an output format, and constraints — the maximum sizes of the input. You write a program that reads the input and prints the correct output. A judge runs your program on hidden test files, each with a time limit (usually $1$–$3$ seconds) and a memory limit (usually $256$ MB), and returns a verdict.

The loop is always the same:

$$\text{read} \;\to\; \text{model} \;\to\; \text{estimate cost} \;\to\; \text{implement} \;\to\; \text{test} \;\to\; \text{submit} \;\to\; \text{verdict}$$

Everything you will learn in this book is aimed at one of those seven steps. Chapter 1 covered *implement*. This chapter covers *read*, *model*, *estimate* and the *verdict* — plus the meta-skill of practising.

What separates a 1200-rated coder from a 1900-rated one is almost never knowledge of an exotic algorithm. It is:

| Skill | 1200 | 1900 |
|---|---|---|
| Reading constraints | skims them | derives complexity from them in 20 seconds |
| Modelling | codes the statement literally | identifies the underlying structure |
| Edge cases | finds them after WA | lists them before coding |
| Debugging | adds prints randomly | bisects the bug with a stress test |
| Problem choice | solves in given order | picks the cheapest problem first |
| Upsolving | moves on | re-solves every unsolved problem |

Every row is trainable, and every row is covered in this chapter or the next.

---

## 2. The contest landscape

Not all contests reward the same behaviour. Know which one you are in.

| Platform | Format | Duration | Scoring | Notes |
|---|---|---|---|---|
| **ICPC** | Teams of 3, one computer | 5 hours | Problems solved, then penalty time | The prestige event; regionals → World Finals |
| **IOI** | Individual, school students | 2 days × 5 hours | Subtasks, partial credit | The olympiad path |
| **Codeforces** | Individual, divisions | 2–2.5 hours | Points that decay with time | Weekly; the main training ground |
| **AtCoder** | Individual (ABC/ARC/AGC) | 100 min | Points, no penalty for wrong answers | Cleanest statements in the business |
| **CodeChef** | Individual, Div 1–4 | 2–3 hours | Points | Weekly "Starters" |
| **CSES** | Problem set, not a contest | — | — | 300 curated problems; the best structured practice |
| **LeetCode** | Individual, interview-focused | 1.5 hours | Points | Useful for interviews, weak for CP |

### What each one trains

- **Codeforces** trains speed and implementation under a decaying clock. Its rating system is the de facto standard, so "I'm 1400" is a shared unit of measurement.
- **AtCoder** trains clean reasoning. Statements are short, constraints are honest, and the difficulty curve is smooth. ABC A–D is the best beginner ladder that exists.
- **CSES** trains depth. It is organised by technique, not by contest, which makes it the right tool when you are learning a topic rather than competing.
- **ICPC** trains teamwork, judgement and stamina. A five-hour contest with one computer is a different sport from a two-hour solo sprint.

> **Practical advice for your first six months:** AtCoder ABC and CSES for learning, Codeforces for measuring, one virtual ICPC per month with two friends for the team skills.

---

## 3. ICPC in detail

The International Collegiate Programming Contest is the oldest and most prestigious team competition in the field, running since 1970.

### The format

| Rule | Detail |
|---|---|
| Team size | 3 students, 1 coach |
| Computers | **1** per team |
| Duration | 5 hours |
| Problems | Typically 10–13, lettered A onwards |
| Language | C, C++, Java, Python, Kotlin |
| Ranking | Most problems solved; ties broken by **penalty time** |

### Penalty time

For every problem you solve, you are charged:

$$\text{penalty} = (\text{minutes from contest start to your first AC}) + 20 \times (\text{rejected submissions before that AC})$$

Problems you never solve cost nothing, no matter how many times you submitted. This single rule drives ICPC strategy:

- **A wrong submission on a problem you eventually solve costs 20 minutes.** On a problem you abandon, it costs nothing.
- **Solving early is worth more than solving carefully.** A problem solved at minute 40 with 2 wrong answers scores 80. The same problem solved at minute 75 with none scores 75. Speed beats caution *once you are confident*.
- **Do not submit to "see what happens".** Every rejected submission is 20 minutes.

### The scoreboard freeze

In the final hour, the public scoreboard freezes. Teams can no longer see others' results, so the last hour is played blind. This creates the classic ICPC drama: a team that appears to be first may have three wrong answers on the last problem.

### The progression

```
Local / university contest
        ↓
Regional contest  (e.g. Amritapuri, Kanpur, Tehran, Cairo)
        ↓
National / super-regional
        ↓
ICPC World Finals  (~140 teams worldwide)
```

Each regional sends its top teams onward. Qualification is by rank at the regional, so a single bad contest can end the season.

### The three-person dynamic

This is the part solo competitors never learn, and it is genuinely a skill:

| Role | Responsibility |
|---|---|
| **Reader / strategist** | Reads problems, ranks them by expected cost, decides what the team attempts next |
| **Coder** | Types the current solution; owns the keyboard for that problem |
| **Tester / debugger** | Builds edge cases, checks the code, finds the counterexample |

In practice roles rotate — you must be able to do all three.

**The one-computer rule is the real constraint.** With one keyboard among three people, a team that argues for ten minutes loses ten minutes of typing. The winning habit is: *decide fast, write it down, hand over the keyboard.*

> *Contest story: a team spent 70 minutes on problem D because two members disagreed about the greedy exchange argument. Meanwhile problem F was a 15-line simulation nobody had opened. They finished with 3 problems; the team that solved F first finished with 5. The lesson is not "avoid arguments" — it is "the reader's job is to keep the queue full, so nobody argues about a problem that is not worth arguing about."*

---

## 4. How a contest actually runs

A well-run contest has a shape. Ignoring it is how you end up with one solved problem and 40 minutes left.

### The first ten minutes

1. **Read everything.** Skim all problem statements before writing a line. You cannot rank problems you have not read.
2. **Classify by expected cost**, not by difficulty label. "This is a simulation with fiddly output formatting" is more informative than "this is problem C".
3. **Find the cheapest problem** — usually the one with the smallest, most explicit statement. Solve that first.
4. **Note the constraints** on each. They tell you the intended complexity, which tells you whether it is a 20-line or 100-line solution.

### The middle

- **Never have an empty queue.** If you are stuck for more than 15 minutes with no progress, switch. Come back later with fresh eyes.
- **Write the test before you submit** when the problem has nasty edge cases (n = 1, all equal, negatives, maximum values).
- **Do not optimise prematurely.** $n \le 200$ and an $\mathcal{O}(n^3)$ solution is correct — submit it.

### The last 30 minutes

- **Stop starting new hard problems.** Finish what is half-done, or verify what is submitted.
- **Re-read the statement of every solved problem.** Output format errors are the most common late-contest WA.
- **In ICPC, check the freeze maths.** Know whether you are ahead on penalty or need one more problem.

### A realistic problem order

| Position | Typical problem | What it tests |
|---|---|---|
| A, B | Warm-up | Reading comprehension, off-by-one |
| C, D | Core | A standard technique applied once |
| E, F | Mid | Two ideas combined, or a non-obvious model |
| G+ | Hard | The real contest; often a known technique disguised |

Problems are **not** ordered by difficulty within a contest, though they usually trend upward. Always read the constraints before assuming which is which.

---

## 5. Verdicts and what they mean

The judge returns a short code. Learn to read it as a diagnosis, not an insult.

| Verdict | Meaning | Usually caused by |
|---|---|---|
| **AC** | Accepted | Correct on every test |
| **WA** | Wrong Answer | Logic error, unhandled edge case, wrong output format |
| **TLE** | Time Limit Exceeded | Complexity too high, or a large constant factor |
| **MLE** | Memory Limit Exceeded | Array too large, or recursion without freeing |
| **RE** | Runtime Error | Segfault, division by zero, out-of-bounds, stack overflow |
| **CE** | Compilation Error | Syntax, missing `;`, wrong `#include` |
| **PE** | Presentation Error | Output correct but formatted wrongly (rare on modern judges) |
| **OLE** | Output Limit Exceeded | Infinite loop printing, or a missing termination condition |
| **ILE** | Idle Limit Exceeded | Interactive problem: your program stopped responding |
| **RF** | Restricted Function | You called something the judge forbids (`system`, `fork`, `exec`) |
| **SJ** | Special Judge | Not a verdict — the problem uses a custom checker for multiple valid answers |

### Diagnosing from the verdict

| Verdict | First question to ask |
|---|---|
| WA on sample | You did not run the sample, or misread the output format |
| WA on tests 1–3 only | Edge cases: $n=1$, empty, all-equal, negatives |
| WA on later tests | Overflow, or a wrong assumption on large input |
| TLE on the last few tests | Constant factor, not complexity — you are close |
| TLE on almost all tests | Wrong complexity class |
| RE on test 1 | Missing `&` in `scanf`, or an uninitialised pointer |
| RE on large tests | Stack overflow from deep recursion, or an out-of-bounds write |
| MLE | Table too big — recompute the memory budget |

> **The most useful habit in this table:** when you get TLE, check *which* tests failed. If it passes $80\%$ of tests and dies on the largest ones, your algorithm is right and your constant factor is wrong — a faster I/O or a smaller array fixes it. If it dies on test 3, the complexity class is wrong and no amount of micro-optimisation will save you.

---

## 6. Ratings and diffs

Every platform has a difficulty scale. They are not comparable, but they are each internally consistent.

### Codeforces

| Rating | Title | Roughly means |
|---|---|---|
| 800–1000 | Newbie | Can implement a loop and a condition |
| 1100–1300 | Pupil | Can implement a standard technique when named |
| 1400–1600 | Specialist | Can choose the technique from the statement |
| 1700–1900 | Expert | Can combine two techniques |
| 2000–2200 | Candidate Master | Can find the non-obvious model |
| 2300–2500 | Master | Reliable on Div. 1 C–D |
| 2600+ | Grandmaster | Contests are a different game |

Codeforces shows two numbers: your **rating** (a running Elo-like score) and each problem's **rating**. A problem's rating is set after the contest from the people who solved it. Solving a 1400 problem consistently means you are around 1400.

### AtCoder

ABC problems are labelled A–F (occasionally G), with an estimated difficulty once the contest ends. Roughly: A ≈ 100, B ≈ 250, C ≈ 550, D ≈ 1000, E ≈ 1500, F ≈ 2000.

### The honest calibration

| If you can reliably solve… | You are roughly |
|---|---|
| Div. 3 A–C / ABC A–C | 1000–1200 |
| Div. 2 A–C / ABC D | 1300–1500 |
| Div. 2 D / ABC E | 1600–1800 |
| Div. 1 C / ABC F | 1900–2100 |

**Do not chase rating.** Rating is a lagging indicator of skill; the way to raise it is to raise the skill, and the way to raise the skill is to solve problems slightly above your comfort level and upsolve everything you fail.

---

## 7. The single most important beginner skill: reading constraints

The constraints are the setter telling you the intended complexity. This is not a hint — it is close to a specification.

| Constraint | Intended complexity | Typical technique |
|---|---|---|
| $n \le 10$ | $\mathcal{O}(n!)$, $\mathcal{O}(n^n)$ | Brute-force permutation |
| $n \le 20$ | $\mathcal{O}(2^n \cdot n)$ | Bitmask DP, meet in the middle |
| $n \le 25$ | $\mathcal{O}(2^n)$ | Subset enumeration |
| $n \le 100$ | $\mathcal{O}(n^4)$ | 4 nested loops, Floyd–Warshall variant |
| $n \le 500$ | $\mathcal{O}(n^3)$ | Floyd–Warshall, interval DP |
| $n \le 5000$ | $\mathcal{O}(n^2)$ | Pairwise DP, LCS |
| $n \le 10^5$ | $\mathcal{O}(n \log n)$ | Sort, binary search, segment tree |
| $n \le 10^6$ | $\mathcal{O}(n)$ or $\mathcal{O}(n \log n)$ | Prefix sums, sieve, two pointers |
| $n \le 10^8$ | $\mathcal{O}(n)$ with a tiny constant | Single pass, no allocation |
| $n \le 10^{18}$ | $\mathcal{O}(\log n)$ or $\mathcal{O}(1)$ | Binary exponentiation, digit DP |
| $n \le 10^{1000}$ | $\mathcal{O}(\text{digits})$ | Big-integer arithmetic, strings |

Read this table backwards when you are stuck: *the problem says $n \le 2 \cdot 10^5$, so an $\mathcal{O}(n \log n)$ solution exists. What data structure gives me $\log n$ per query?*

### The other half of reading a statement

| Element | What to look for |
|---|---|
| **Constraints** | The intended complexity, and the overflow risk |
| **Input format** | How many test cases? Is there a leading $T$? |
| **Output format** | Exact wording, trailing newline, "print $-1$ if impossible" |
| **Samples** | Weak; they never include the hard cases |
| **Keywords** | "minimum/maximum" → greedy or DP; "count" → combinatorics or DP; "if possible" → feasibility check |
| **Multiple test cases** | A $T$ at the top means you must clear global arrays between tests |

The "if impossible, print $-1$" clause is worth a dedicated search of the statement. It is the single most common source of WA on otherwise-correct solutions.

---

## 8. How to practise so it compounds

Most people practise by solving random problems, which is the slowest possible method. Here is what actually works.

### The weekly loop

| When | What | Why |
|---|---|---|
| 2–3 contests per week | Live Codeforces / AtCoder | Trains speed and decision-making under a clock |
| After each contest | **Upsolve** every unsolved problem | The learning happens here, not in the contest |
| 3–5 problems per day | Topic ladder (CSES / ABC by tag) | Builds technique breadth |
| Once a week | Re-solve an old problem you failed | Tests retention, exposes fake understanding |

### The 30-minute rule

When you are stuck on a problem, set a timer. Give it 30–45 minutes of genuine effort — write the brute force, draw the small cases, try to find the pattern. *Then* read the editorial.

The reason is not purity. It is that the struggle is what encodes the idea. Reading an editorial after 5 minutes produces the feeling of understanding and none of the substance; you will fail the same problem type next month.

### Upsolving is non-negotiable

A contest where you solve 3 and upsolve 2 teaches you 5 problems. A contest where you solve 3 and close the tab teaches you 3. Over a year that is the difference between 1500 and 2000.

**Upsolve protocol:**

1. Re-read the problem cold, no editorial.
2. If still stuck after 20 minutes, read the editorial's *first line* only.
3. Implement it yourself. Do not copy.
4. If you copied, re-implement from scratch the next day.

### Virtual contests

Codeforces and AtCoder both let you run a past contest on the original clock. This is the single best simulation of contest pressure available for free.

| Setting | Recommendation |
|---|---|
| Frequency | 1–2 per week |
| Difficulty | One division below your comfort zone, so you finish |
| Rules | No editorials, no pausing, no looking up syntax |
| Afterwards | Upsolve everything |

### What not to do

- **Do not grind 800-rated problems.** You learn nothing from problems you can already solve instantly.
- **Do not read editorials as your primary learning method.** They are for unblocking, not for study.
- **Do not skip implementation.** "I understand the idea" and "I can code it in 20 minutes without bugs" are different skills.
- **Do not switch topics every week.** Depth beats breadth until around 1800.

---

## 9. The first contest checklist

Print this. Use it for your first ten contests.

**Before the contest**

- [ ] Template ready and tested (fast I/O, `#define int long long` decision made)
- [ ] Compiler flags known (`-std=c++17 -O2 -Wall`)
- [ ] A local test file open, with the sample input pasted
- [ ] Water, and 90 minutes of uninterrupted time

**First 10 minutes**

- [ ] Read every problem, or at least every problem title and constraint
- [ ] Rank by expected cost
- [ ] Start with the cheapest

**During**

- [ ] Write the brute force first if the problem has small constraints
- [ ] List edge cases before coding: $n=1$, all equal, negatives, maximum values
- [ ] Run the sample locally before submitting
- [ ] If stuck 15 minutes with no progress, switch problems

**After**

- [ ] Note which problems you failed and why (not complexity, edge case, idea)
- [ ] Upsolve at least one unsolved problem within 48 hours
- [ ] Write one line about what you learned

---

## 10. The mental game

Three things derail beginners more than any algorithm.

### Rating anxiety

Rating is noisy. A 1500-rated coder will swing ±100 between contests based on problem selection luck. Watching the number after every round is a reliable way to stop improving, because it pushes you toward safe, unrewarding problem choices.

**Fix:** set a process goal per month ("upsolve every Div. 2 D") instead of a rating goal.

### The plateau

Everyone plateaus. Around 1400 it is usually "I can implement techniques but cannot choose them". Around 1800 it is "I know the technique but the model is hidden". Plateaus are not a sign of a ceiling; they are a sign that your current practice method has stopped producing new information.

**Fix:** change the input. Move to a harder judge, do a topic you have avoided (usually DP or graphs), or start solving problems by tag instead of by contest.

### Tilt

Submitting the same wrong solution five times, or rage-coding after a WA, costs more rating than the original bug.

**Fix:** after two WAs on the same problem, stop. Write down what you believe the code does, then read the statement again. The bug is almost always a mismatch between those two.

---

## Key Takeaways

- The loop is read → model → estimate → implement → test → submit. Most lost points come from the first three, not the last three.
- ICPC: 3 people, 1 computer, 5 hours, 20-minute penalty per rejected submission **on problems you eventually solve**. Wrong answers on abandoned problems are free.
- In ICPC, speed beats caution once you are confident: solving at minute 40 with 2 wrong answers beats minute 75 with none.
- The scoreboard freezes in the last hour; know your penalty situation before it does.
- Learn the verdicts as diagnoses. TLE on the last few tests means constant factor; TLE on test 3 means wrong complexity class.
- **The constraints are the specification.** $n \le 10^5$ means "find the $\mathcal{O}(n \log n)$ solution". Read the table backwards when stuck.
- A problem's rating is set from who solved it, not from its position. Read all problems before choosing.
- 30-minute rule: genuine effort before the editorial, every time.
- Upsolving is where learning happens. A contest you upsolve teaches 5 problems; one you abandon teaches 3.
- Change your practice method at a plateau, not your effort level.
- Two WAs on the same problem → stop and re-read the statement. The bug is a mismatch between the code and the statement.

---

## Common Mistakes & Pitfalls

| # | Mistake | Why it bites | Fix |
|---|---|---|---|
| 1 | Coding before reading all problems | You solve a 1900 while a 1200 sits unopened | Read everything first |
| 2 | Ignoring the constraints | You write $\mathcal{O}(n^2)$ for $n = 10^5$ and TLE | Derive complexity from constraints first |
| 3 | Not running the sample | The most common avoidable WA | Paste the sample locally every time |
| 4 | Missing "print $-1$ if impossible" | Correct algorithm, wrong answer | Search the statement for output clauses |
| 5 | Forgetting to clear globals between test cases | Passes test 1, fails test 2 onwards | Re-initialise per test, or declare inside the loop |
| 6 | Submitting to "see what happens" in ICPC | Every rejected submission is 20 penalty minutes | Only submit when you believe it is correct |
| 7 | Grinding easy problems | No new information, rating inflates then stalls | Solve slightly above your level |
| 8 | Reading the editorial after 5 minutes | Produces false understanding | 30–45 minutes of real effort first |
| 9 | Never upsolving | Same problem type fails again next month | Upsolve within 48 hours |
| 10 | Chasing rating, not skill | Encourages safe, unrewarding choices | Set process goals per month |
| 11 | Debugging by adding random prints | Slow, and often misses the bug | Stress test against a brute force |
| 12 | Optimising before correctness | Wastes time on problems that are still wrong | Make it correct, then fast |
| 13 | Assuming problem order equals difficulty | Time lost on a hard problem while an easy one waits | Rank by expected cost |
| 14 | Not knowing the language's limits | `cin` without sync-off TLEs at $10^6$ | See [Chapter 1, Topic 03](../CPP%20for%20CP/Topic%2003%20-%20Input,%20Output%20and%20Efficient%20I-O%20specially%20for%20CP%20(VERY%20IMPORTANT).md) |
| 15 | Practising alone forever | Missing the ICPC-specific team skills | One team contest per month |

---

## Practice Problems

This topic has no algorithmic content, so the "problems" are drills.

| # | Drill | Source | Difficulty |
|---|---|---|---|
| 1 | Run one virtual contest and write down your problem ranking before starting | Codeforces / AtCoder | Easy |
| 2 | Take five problems you solved, and write the intended complexity from the constraints alone | Any judge | Easy |
| 3 | Deliberately submit a wrong answer and identify the verdict from the judge's message | Any judge | Easy |
| 4 | Upsolve two problems from your last contest, without the editorial | Your own history | Medium |
| 5 | Do a 5-hour virtual ICPC with two friends, one computer | Codeforces Gym | Medium |
| 6 | Re-solve one problem you failed a month ago, cold | Your own history | Medium |
| 7 | For ten problems, predict the WA reason before submitting, then check | Any judge | Medium |

---

## Next Up

→ [Topic 02 — Time Complexity (Basics to Advanced)](Topic%2002%20-%20Time%20Complexity%20(Basics%20to%20Advanced).md)

You now know that constraints encode the intended complexity. Topic 02 teaches you to compute that complexity from your own code — counting loops, recognising halving as $\log n$, reasoning about nested structures, and using recurrences for recursive solutions. It is the most load-bearing topic in the book: almost every TLE you will ever get is a complexity mistake you could have caught on paper.