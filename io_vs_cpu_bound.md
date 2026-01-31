# I/O-Bound vs CPU-Bound Work in Python

This file explains **why different kinds of work require different concurrency models**, and why misunderstanding this distinction leads to slow, fragile systems.

This topic comes **after object memory costs** and **before concurrency models**, because concurrency only helps once you understand *what you are waiting for*.

(Reference: *High Performance Python*, Concurrency chapter; *Fluent Python*, Part V)

---

## 1. The Fundamental Performance Question (Problem First)

When a Python program is slow, engineers often ask:

> “Should I use threads? async? multiprocessing?”

This question is **too late**.

The real first question is:

> “Is my program waiting on **I/O** or on **the CPU**?”

---

## 2. Two Very Different Kinds of Work

### I/O-Bound Work
The program spends most of its time:
- waiting for network responses
- waiting for disk reads/writes
- waiting for APIs or databases

The CPU is mostly **idle**.

### CPU-Bound Work
The program spends most of its time:
- performing calculations
- transforming data
- running tight loops

The CPU is **busy**.

---

## 3. Coding Problem #1 — I/O-Bound Task

### Problem statement

Simulate a task that waits on external resources.
The CPU does almost nothing.

```python
import time

def fetch_data():
    time.sleep(2)  # simulate network delay
    return "data"
