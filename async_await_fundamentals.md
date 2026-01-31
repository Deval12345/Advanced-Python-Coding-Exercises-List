# async / await Fundamentals in Python

This file explains **what `async` and `await` really mean**, why they exist,
and what problem they solve that threads alone cannot.

This topic builds on:
- I/O-bound vs CPU-bound work
- concurrency overview

It is a prerequisite for understanding:
- event loops
- asyncio APIs
- modern async frameworks

(Reference: *Fluent Python*, Part V – asyncio fundamentals)

---

## 1. Why `async / await` Exists (Problem First)

Thread-based concurrency works well — up to a point.

As systems grow, threads introduce problems:
- high memory overhead per thread
- context switching costs
- difficulty managing thousands of concurrent tasks
- subtle race conditions

### Core problem

> How can we handle **large numbers of I/O-bound tasks**
> efficiently **without creating large numbers of threads**?

Python’s answer: **cooperative multitasking with `async / await`**.

---

## 2. What `async` Really Means

Declaring a function with `async`:

```python
async def fetch_data():
    ...
