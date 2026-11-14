# Competitive Programming using C++

**The Complete Book** — a 13-chapter, 152-topic curriculum that takes you from `int main()` to segment tree beats.

This is not a syntax reference and it is not a problem archive. It is a **book you read in order**, written for the specific skill of winning contests: estimating cost before you code, recognising the intended solution, and not losing points to the six traps everyone hits.

---

## Start here

| If you are… | Begin at | Then |
|---|---|---|
| New to C++ and contests | [Chapter 1 — CPP for CP](CPP%20for%20CP/README.md) | [Chapter 2 — CP Level 1](CP%20Level%201/README.md) |
| Comfortable with C++, new to CP | [Chapter 2 — CP Level 1](CP%20Level%201/README.md) | Chapter 3 |
| Solving Div. 2 A–B, stuck at C | [Chapter 3 — CP Level 2](#chapter-map) | Chapter 5 |
| Preparing for ICPC regionals | [Chapter 6 — CP Level 5](#chapter-map) | Chapter 9 |

---

## What makes this different

Most CP material is one of two things: an API listing, or a solution dump. This book is a middle layer that is usually missing — **the reasoning that connects a problem statement to a data structure**.

Every topic file follows the same contract:

- **Why before how.** The mechanism is explained before the syntax, so you can reason about a case the author never wrote down.
- **CP-first framing throughout.** For each feature: when does this TLE? when does this overflow? what is the constant factor? what will a setter test?
- **The trap is part of the lesson.** Every file ends with a `Common Mistakes & Pitfalls` table — 15 to 26 rows of things that actually cost submissions, with the fix on the same line.
- **Complexity everywhere.** Tables for every container, every algorithm, every memory budget.
- **Practice, not vibes.** Each file ends with a graded problem table tied to the exact technique it taught.
- **Verified code.** In Chapter 1, all 510 code blocks are compilable C++17, and the core algorithms were stress-tested against brute force before publication.

---

## Chapter map

| # | Chapter | Folder | Topics | Status |
|---|---|---|---|---|
| 1 | CPP for CP | `CPP for CP/` | 15 | **Complete** |
| 2 | [CP Level 1](CP%20Level%201/README.md) | `CP Level 1/` | 11 | **Complete** |
| 3 | CP Level 2 | `CP Level 2/` | 9 | **In progress** (5/9) |
| 4 | CP Level 3 | `CP Level 3/` | 11 | Planned |
| 5 | CP Level 4 | `CP Level 4/` | 11 | Planned |
| 6 | CP Level 5 | `CP Level 5/` | 10 | Planned |
| 7 | CP Level 6 | `CP Level 6/` | 11 | Planned |
| 8 | CP Level 7 | `CP Level 7/` | 10 | Planned |
| 9 | CP Level 8 | `CP Level 8/` | 11 | Planned |
| 10 | CP Level 9 | `CP Level 9/` | 12 | Planned |
| 11 | CP Level 10 | `CP Level 10/` | 10 | Planned |
| 12 | CP Level 11 | `CP Level 11/` | 14 | Planned |
| 13 | Recap & Problems | `Recap & Problems/` | 17 | Planned |

**152 topics total.** The difficulty curve is deliberate: Chapter 1 is the language, Chapters 2–3 are the techniques that solve most Div. 2 and Div. 3 problems, and Chapters 4–12 climb to ICPC World Finals territory.

### What each band covers

| Chapters | Band | You can solve |
|---|---|---|
| 1 | Language | Nothing yet — you can compile and debug |
| 2–3 | Level 1–2 | Div. 3 A–F, Div. 2 A–C, CSES Introductory + Sorting |
| 4–5 | Level 3–4 | Div. 2 D–E: DP, number theory, graphs, DSU |
| 6–8 | Level 5–7 | Div. 1 C–D: trees, range queries, geometry, advanced DP |
| 9–11 | Level 8–10 | Div. 1 E–F: persistent structures, flows, FFT, suffix structures |
| 12 | Level 11 | The exotic end: matroids, game theory, fracturing search |
| 13 | Recap | Retention — mixed problem sets across everything |

---

## Repository layout

```
Competitive Programming using C++ Complete Book/
├── content.md                    ← the master syllabus (single source of truth)
├── README.md                     ← this file
└── CPP for CP/                   ← Chapter 1
    ├── README.md                 ← chapter guide and table of contents
    ├── Topic 01 - ....md
    ├── ...
    └── Topic 15 - ....md
```

### Naming conventions

- **One chapter = one folder**, named after the chapter title with the `CP Level N` prefix preserved.
- **One topic = one file**, named `Topic NN - Title.md`, zero-padded so `ls` sorts correctly.
- Illegal filename characters are replaced: `:` → `-`, `/` → `-`. So the syllabus line *"Essential Prebuilt Algorithms: Sort…"* becomes the file *"Essential Prebuilt Algorithms - Sort…"*.
- **Every chapter folder has its own `README.md`** with a table of contents, prerequisites and a study plan.

---

## Conventions used throughout

| Convention | Meaning |
|---|---|
| `> **Prerequisites:**` | Topics to read first, as links |
| `> **You will be able to:**` | Concrete outcomes for the file |
| *Contest story:* | An italic aside with a real failure mode and its cost |
| `Key Takeaways` | 8–14 one-line facts worth memorising |
| `Common Mistakes & Pitfalls` | A numbered table: mistake, why it bites, fix |
| `Practice Problems` | Graded table: type, judge, difficulty |
| `Next Up` | The link forward, with a one-paragraph bridge |
| $\mathcal{O}(n \log n)$ | Complexity, in proper notation |
| `text` fences | Program output, compiler messages, terminal transcripts |

**Code style:** C++17, `#include <bits/stdc++.h>`, `using namespace std;`, `int main()` with fast I/O. This is contest style on purpose — Chapter 1 §5 explains exactly why that is acceptable here and unacceptable in production.

---

## Toolchain

You need a C++17 compiler. Nothing else.

```bash
# Linux / macOS
g++ -std=c++17 -O2 -Wall -Wextra -o sol sol.cpp

# Windows (MinGW), from PowerShell
g++ -std=c++17 -O2 -Wall -Wextra -o sol.exe sol.cpp
```

Debug builds used throughout the book:

```bash
g++ -std=c++17 -g -fsanitize=address,undefined -D_GLIBCXX_DEBUG -o sol sol.cpp
```

> `-fsanitize` requires a real GCC/Clang, not MinGW.org's GCC 6.3. On Windows, install MSYS2/MinGW-w64 or use WSL.

---

## Progress

- [x] Chapter 1 — CPP for CP (15/15 topics, ~13,450 lines, 510 code blocks)
- [x] Chapter 2 — CP Level 1 (11/11 topics, ~7,370 lines, 140 code blocks)
- [ ] Chapter 3 - CP Level 2 - 5 of 9 topics
- [ ] Chapter 4 — CP Level 3
- [ ] Chapter 5 — CP Level 4
- [ ] Chapter 6 — CP Level 5
- [ ] Chapter 7 — CP Level 6
- [ ] Chapter 8 — CP Level 7
- [ ] Chapter 9 — CP Level 8
- [ ] Chapter 10 — CP Level 9
- [ ] Chapter 11 — CP Level 10
- [ ] Chapter 12 — CP Level 11
- [ ] Chapter 13 — Recap & Problems

---

## Contributing

Corrections are welcome, especially:

- **Wrong complexity claims** — cite the operation and the container.
- **A trap that is missing** from a `Common Mistakes` table, with a one-line reproduction.
- **A code block that does not compile** on your compiler, with the error text.

Please keep the house style: explanation before syntax, a table where a list would be denser, and no filler paragraphs. Every file must keep the four closing sections.

---

## Licence

Text and code are released under the MIT Licence unless stated otherwise in a chapter folder. Problem references are to publicly available judges (Codeforces, AtCoder, CSES, UVa, ICPC) and no problem statements are reproduced.