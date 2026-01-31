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
