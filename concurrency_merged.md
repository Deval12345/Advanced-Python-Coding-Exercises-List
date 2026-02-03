# Concurrency Exercises — Applying the Models

This file contains **integrated, real-world–style exercises** that demonstrate
*why* Python has multiple concurrency models and *when* each should be used.

These exercises intentionally combine concepts from:
- I/O-bound vs CPU-bound classification
- threads
- async / await
- process-based parallelism

The goal is not syntactic mastery, but **correct architectural choice**.

(Reference: *Fluent Python* Part V; *High Performance Python*)

---

## Exercise 1 — I/O-Bound API Aggregation (Threads)

### Problem statement

You are building a dashboard that:
- fetches data from multiple external APIs
- each call is slow and blocking
- results are independent
- total response time matters

Sequential execution wastes time waiting.

---

### Baseline (Sequential — Inefficient)

```python
import time

def fetch_api(i):
    time.sleep(1)  # simulate network delay
    return f"data-{i}"

results = []
for i in range(5):
    results.append(fetch_api(i))
```

---

# Concurrency Models in Python — A Conceptual Overview

This file explains **why Python has multiple concurrency models**, what problems each model is designed to solve, and how they evolved historically.

This topic comes **after understanding I/O-bound vs CPU-bound work** and **before diving into specific mechanisms (threads, async, processes)**.

(Reference: *High Performance Python*, *Fluent Python* Part V)

---

## 1. Why Concurrency Exists at All (Problem First)

Programs are slow primarily because they **wait**:
- waiting for I/O
- waiting for other systems
- waiting for resources

Concurrency exists to:
> **Make progress while waiting**

It is *not* primarily about doing more computation per second.

---

## 2. The Three Concurrency Models Python Supports

Python deliberately supports **three distinct models**:

1. **Thread-based concurrency**
2. **Event-driven (async) concurrency**
3. **Process-based parallelism**

Each solves a *different* problem.

---

## 3. Coding Problem #1 — Sequential Waiting Is Wasteful

### Problem statement

You need to perform several slow I/O operations.

```python
import time

def fetch(i):
    time.sleep(1)
    return i

for i in range(5):
    fetch(i)
```

---

# IPC and Data-Sharing Cost in Multiprocessing

Multiprocessing gives true CPU parallelism.

But there is a hidden cost:

👉 **data must move between processes**

And moving data is expensive.

Sometimes:

> copying data costs more than computing it

This section explains why that happens and how to design around it.

---

# Threads in CPython — Why They Matter for I/O

In CPython, threads do not provide true CPU parallelism because of the Global Interpreter Lock (GIL). However, threads are still extremely powerful for real systems dominated by waiting.

The key idea: most modern programs spend more time waiting on networks, disks, and databases than doing math.

Threads allow your program to overlap waiting time. While one thread waits for an external resource, another thread runs.

This reduces total wall-clock time and makes programs feel responsive and concurrent.

---

# I/O-Bound vs CPU-Bound Work in Python

This file explains **why different kinds of work require different concurrency models**, and why misunderstanding this distinction leads to slow, fragile systems.

---

# Multiprocessing in Real Systems — Practical Teaching Notes

Multiprocessing is not just about speed.

It is about **system design**.

You are building a group of independent workers that cooperate safely.
