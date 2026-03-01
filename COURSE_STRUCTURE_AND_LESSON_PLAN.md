# Advanced Python Course — Derived Lesson Plan and Course Structure

This document defines the **lecture order** used to generate all Interactive Delivery content. Concepts build step by step; no later topics are assumed before they are taught.

**Note:** No "Lesson Plan" folder or Excel was found in the repository. This order is derived from the README recommended learning path and the Final files, and is designed for **gradual progression** (7–8 minutes per lecture, with splitting where needed).

---

## Phase 1 — Python as a Framework (Objects and Syntax)

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 1 | Python Data Model Part 1 | Framework, __len__, __getitem__, __iter__, __contains__, custom containers | — |
| 2 | Python Data Model Part 2 | Operators (__add__), duck typing, strategy pattern | L1 |
| 3 | Protocols and informal interfaces | Behavior over inheritance, duck typing, protocol-based extensibility | L1, L2 |

*(Lectures 1–3: see `Interactive delievary/lecture1`, `lecture2`; L3 content is Protocols; L4 starts Attribute access in `lecture3`.)*

**Clarification:** Each row in the lesson plan need not be one full lecture. Long topics are split into multiple lectures (e.g. Data Model → Part 1 and Part 2). All lectures target **7–8 minutes**; concepts build step by step with no forward references.

---

## Phase 2 — Attribute-Level Machinery

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 4 | Attribute access control | Lookup order, __getattr__, __getattribute__, lazy/proxy use cases | L1–L3 |
| 5 | Descriptors | Descriptor protocol, __get__/__set__/__delete__, validated fields, reuse across classes | L4 |

---

## Phase 3 — Functions and Control Flow

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 6 | Functions as objects | First-class functions, callables, lambda, passing functions | L1–L5 |
| 7 | Decorators and closures | Closures, decorator syntax, decorators as wrappers | L6 |
| 8 | Context managers | with statement, __enter__/__exit__, resource lifecycle | L1, L6 |
| 9 | EAFP and exception model | Easier to ask forgiveness than permission, exception flow, design impact | L8 |

---

## Phase 4 — Iteration and Memory

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 10 | Iterators, iterables, generators | __iter__, __next__, yield, lazy sequences | L1, L6 |
| 11 | Memory layout and __slots__ | Object overhead, memory layout, __slots__ for fixed attributes | L1, L4 |

---

## Phase 5 — Concurrency (Concepts in Order)

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 12 | I/O vs CPU bound and GIL | When threads help, when they do not, GIL in one sentence | L1–L11 |
| 13 | Concurrency overview and threading | Threading for I/O-bound work, coordination | L12 |
| 14 | Multiprocessing and IPC | Processes, IPC cost, when to use multiprocessing | L12, L13 |

---

## Phase 6 — Consolidation

| # | Lecture | Main concepts | Depends on |
|---|--------|----------------|------------|
| 15 | Capstone project | One project tying data model, protocols, attributes, functions, decorators, context managers, iteration | L1–L14 |

---

**Interactive Delivery:** Full content for all 15 lectures exists under `Interactive delievary/lecture1` … `lecture15`. Lecture numbering in folders matches course order (lecture1 = L1, lecture2 = L2/Protocols, lecture3 = L4 Attribute access, … lecture15 = Capstone).

## Output Format (per lecture)

Each lecture folder contains:

- **speech_source_<n>.md** — Synchronisation file: CONCEPT X.Y and EXAMPLE X.Y blocks; maps speech to concepts and code.
- **speech_<n>.md** — Clean delivery speech only (no headers, no code, no symbols).
- **points_<n>.md** — Slide points (one point per CONCEPT; titles from theme).
- **code_<n>.md** — Demo code with line numbers, keyed by EXAMPLE.
- **homework_<n>.txt** — Optional; coding exercise when appropriate.
- **readme.txt** — Optional; short note for the lecture.

Rules from Agent Prompts (rules.txt, Agent 4, 5, 6) are followed: no code or symbols in speech, “special dunder method” wording, structural framing for code (“Here we define a class named…”), slides from CONCEPT blocks only.

---

## Lecture length

Target: **7–8 minutes** per lecture. Where a topic is long, it is split into multiple lectures (e.g. Data Model → Part 1 and Part 2). Concepts are not jumped; each lecture only uses material from earlier lectures.
