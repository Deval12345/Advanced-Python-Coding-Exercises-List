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
