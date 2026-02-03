# GIL Motivation Through a Real CPU-Intensive Example

This example demonstrates why threads do not speed up CPU-heavy work in CPython.

When tasks are computation-heavy and do not wait on I/O, threads compete for the Global Interpreter Lock (GIL).

Only one thread runs Python bytecode at a time.

So CPU tasks execute sequentially even with many threads.

---

## Real-World Scenario: Image Rendering Pipeline

Imagine a graphics engine rendering high-resolution frames.

Each frame requires heavy pixel math with almost no I/O.

A developer tries to speed it up using threads — but performance does not improve.

Why?

Because the GIL forces threads to take turns using the CPU.

Multiprocessing fixes this by running separate interpreters on separate cores.

---

## CPU Task Definition

We simulate heavy pixel computation similar to rendering or scientific simulation.

```python
def cpu_heavy_task():
    x = 0
    for _ in range(40_000_000):
        x += 1
```

---

## Threaded Version — GIL Bottleneck

Even with 4 threads, only one executes Python code at a time.

Threads fight for the GIL.

Runtime is similar to running the task sequentially.

```python
import threading
import time

start = time.time()

threads = [threading.Thread(target=cpu_heavy_task) for _ in range(4)]

for t in threads:
    t.start()
for t in threads:
    t.join()

print("Threads runtime:", round(time.time() - start, 2))
```

---

## Multiprocessing Version — True Parallelism

Each process gets its own interpreter and its own GIL.

Tasks run on different CPU cores simultaneously.

Runtime drops close to:

> 1 / number_of_cores

```python
import multiprocessing
import time

start = time.time()

procs = [multiprocessing.Process(target=cpu_heavy_task) for _ in range(4)]

for p in procs:
    p.start()
for p in procs:
    p.join()

print("Processes runtime:", round(time.time() - start, 2))
```

---

## Key Insight

Threads are not broken.

They solve **I/O waiting**.

But the GIL intentionally prevents CPU-heavy thread parallelism.

This protects Python's memory model and keeps single-thread performance fast.

Correct tool choice:

- CPU scaling → multiprocessing
- I/O overlap → threads
