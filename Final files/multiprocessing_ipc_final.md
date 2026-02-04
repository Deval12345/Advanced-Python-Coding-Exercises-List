
# Multiprocessing & Inter-Process Communication (IPC)

## Overview

The GIL prevents true CPU parallelism inside a single Python process.

Multiprocessing exists to:

- bypass the GIL safely
- use multiple CPU cores
- isolate memory for correctness
- achieve reliable parallelism

Python deliberately chose process-based parallelism over complex shared-memory threading.

Isolation is a design feature.

This module explains multiprocessing as an architectural model.

---

## Section 1 — Why Python Uses Processes

Threads share memory.
Processes do not.

Shared memory requires fine-grained locks and corruption risk.

Python chose:

isolation over complexity.

Each process has:

- its own interpreter
- its own GIL
- its own memory space

This guarantees safe parallel execution.

---

## Section 2 — True CPU Parallelism

Processes run on separate cores simultaneously.

Threads cannot bypass the GIL.
Processes can.

### Exercise — CPU Parallelism

```python
from multiprocessing import Pool
import time
import math

def heavy(n):
    return sum(math.sqrt(i) for i in range(n))

N = 10_000_00

start = time.perf_counter()

with Pool() as pool:
    results = pool.map(heavy, [N] * 4)

print("Results:", results)
print("Time:", round(time.perf_counter() - start, 2))
```

Observe scaling with CPU cores.

---

## Section 3 — IPC Has a Cost

Processes cannot share memory freely.

Data must be:

- copied
- serialized
- transmitted

IPC overhead is real.

Multiprocessing favors:

heavy computation + minimal communication

Architecture matters more than syntax.

---

## Section 4 — Process Pools and Executors

High-level abstractions simplify multiprocessing.

```python
from concurrent.futures import ProcessPoolExecutor

def compute(x):
    return x * x

with ProcessPoolExecutor() as executor:
    results = executor.map(compute, range(10))

print(list(results))
```

You get:

- automatic worker management
- parallel execution
- task abstraction

---

## Section 5 — Queue-Based Worker Model

Producer-consumer architecture.

Workers never talk directly.
Main process coordinates.

```python
from multiprocessing import Process, Queue

def worker(tasks):
    while not tasks.empty():
        print("Working on", tasks.get())

if __name__ == "__main__":
    tasks = Queue()
    for i in range(5):
        tasks.put(i)

    procs = [Process(target=worker, args=(tasks,)) for _ in range(2)]
    for p in procs: p.start()
    for p in procs: p.join()
```

This model scales safely.

---

## Architectural Rule of Thumb

I/O-bound → threads or async  
CPU-bound → processes  
Mixed → async + process pool

Choosing wrong model wastes hardware.

---

## Final Takeaways

Multiprocessing exists to:

- bypass the GIL
- enable real parallelism
- isolate memory
- protect correctness
- scale CPU-heavy workloads

Isolation is intentional engineering.

