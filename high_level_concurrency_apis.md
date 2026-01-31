# High-Level Concurrency APIs in Python

This file explains **why high-level concurrency APIs exist**, how they simplify
orchestration of concurrent work, and when you should prefer them over
low-level primitives.

This topic builds on:
- Futures and executors
- async / await fundamentals
- event loop mechanics

It represents the **“composition layer”** of Python concurrency.

(Reference: *Fluent Python*, Part V; `asyncio` and `concurrent.futures` APIs)

---

## 1. Why High-Level APIs Exist (Problem First)

Low-level concurrency primitives work, but they are hard to scale:

- manual thread management
- manual task tracking
- manual result aggregation
- manual error propagation

As systems grow, this leads to:
- complex control flow
- duplicated orchestration logic
- subtle bugs

### Core problem

> How can we express **common concurrency patterns**  
> without re-implementing coordination logic every time?

Python’s answer: **high-level concurrency APIs**.

---

## 2. The Idea of Structured Concurrency

High-level APIs encourage **structured concurrency**:
- tasks are created together
- tasks are awaited together
- failures propagate predictably
- lifetimes are scoped

Instead of “fire and forget”, you get **controlled concurrency**.

---

## 3. Coding Problem #1 — Fan-Out / Fan-In Pattern

### Problem statement

You need to:
- launch many independent tasks
- wait for all of them to finish
- collect results in one place

This is one of the most common concurrency patterns.

---

## 4. Solution — `asyncio.gather`

```python
import asyncio

async def fetch(i):
    await asyncio.sleep(1)
    return f"data-{i}"

async def main():
    results = await asyncio.gather(
        fetch(1),
        fetch(2),
        fetch(3),
    )
    print(results)

asyncio.run(main())
