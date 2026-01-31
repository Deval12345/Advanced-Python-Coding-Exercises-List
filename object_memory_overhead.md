# When Object Overhead Dominates Computation

This file explains **why Python programs sometimes become slow even when the logic is simple**, and how **object overhead and memory behavior** can dominate runtime more than algorithms themselves.

This topic builds directly on:
- attribute storage (`__dict__`, `__slots__`)
- before concurrency and parallelism

(Reference: *Fluent Python*, Object References & Mutability; *High Performance Python*, Chapter on memory)

---

## 1. The Hidden Performance Problem (Problem First)

A common misconception:

> “If my algorithm is simple, my program should be fast.”

In Python, this is often false.

Why?
Because:
- objects are heavy
- memory access is expensive
- cache misses dominate execution time
- garbage collection adds overhead

---

## 2. Core Problem Statement

You write code that:
- performs simple arithmetic
- iterates over large collections
- uses many small objects

Yet:
- CPU usage is low
- execution time is high
- performance does not scale as expected

### The real question

> Is the program slow because of computation —  
> or because of **object and memory overhead**?

---

## 3. Coding Problem #1 — Simple Logic, Many Objects

### Problem statement

You want to compute a simple aggregate over a large dataset.
Each data point is represented as a Python object.

```python
items = [{"x": i, "y": i * i} for i in range(1_000_000)]
total = sum(item["y"] for item in items)
