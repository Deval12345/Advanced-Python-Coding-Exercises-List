# Futures and Executors in Python

This file explains **what Futures are**, why they exist as an abstraction,
and how **executors decouple task submission from task execution**.

This topic sits between:
- low-level concurrency models (threads / processes)
- high-level async APIs

It explains the bridge between **synchronous code** and **concurrent execution**.

(Reference: *Fluent Python*, Part V; `concurrent.futures` documentation)

---

## 1. Why Futures Exist (Problem First)

Before Futures, concurrent code often looked like this:
- callbacks
- shared state
- manual synchronization
- error handling spread everywhere

This made code:
- hard to read
- hard to compose
- hard to reason about failures

### Core problem

> How can we represent “a result that will be available later”  
> in a clean, composable, and debuggable way?

Python’s answer: **Futures**.

---

## 2. What a Future Is (Conceptual)

A **Future** is:
- a placeholder for a result
- produced by submitting a task
- completed at some point in the future

It represents **one unit of concurrent work**.

Key idea:

> The task runs elsewhere.  
> The Future lives *here*.

---

## 3. Coding Problem #1 — Background Task Execution

### Problem statement

You want to:
- run work in the background
- continue doing other things
- retrieve the result later
- handle errors cleanly

---

## 4. Baseline Solution — ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor

def work(x):
    return x * x

with ThreadPoolExecutor() as executor:
    future = executor.submit(work, 10)
    print("Task submitted")
    result = future.result()
    print(result)
