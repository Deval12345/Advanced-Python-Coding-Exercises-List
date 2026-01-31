# Advanced Python – Internals, Design & Performance

This repository is a **self-guided, mechanism-first learning path** for Advanced Python.
It is inspired by:

- *Fluent Python* (Luciano Ramalho)
- MIT Python / CS playlists
- Real-world production design patterns

The goal is to explain **why Python works the way it does**, not just *how to use it*.

---

## How to Use This Repository

- Each `.md` file focuses on **one core Python mechanism**
- Every file contains:
  - A **clear problem statement**
  - **Correct, minimal Python code**
  - Explanation of *why the mechanism exists*
  - Learning outcomes

You should be able to understand each topic **without external explanation**.

---

## Recommended Learning Order (Pre-GIL)

Read the files in this order for best conceptual flow:

1. `attribute_access.md`  
2. `descriptors.md`  
3. `abcs.md`  
4. `protocols_vs_abcs.md`  
5. `slots_memory.md`  
6. `object_memory_overhead.md`  
7. `io_vs_cpu_bound.md`  
8. `concurrency_overview.md`  
9. `concurrency_exercises.md`  
10. `design_tradeoffs.md`

> Topics after the Global Interpreter Lock (GIL) are intentionally excluded here.

---

## Design Philosophy

- **Behavior before syntax**
- **Mechanisms before patterns**
- **Costs before optimization**
- **Constraints before escape hatches**

This mirrors how Python itself evolved.

---

## Intended Audience

- Intermediate Python users moving to **advanced internals**
- Engineers preparing for **system-level Python interviews**
- Educators teaching **Fluent Python–style courses**

---

## How This Repo Should Feel

If done right, a reader should think:

> “Oh — *that’s* why Python works this way.”

That’s the goal.
