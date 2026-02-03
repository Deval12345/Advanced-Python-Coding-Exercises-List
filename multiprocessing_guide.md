
# Multiprocessing — True Parallelism Beyond the GIL

## Overview

Python uses processes — not threads — to achieve real CPU parallelism.

This is a deliberate architectural choice.

Instead of complex shared-memory locking, Python prefers:

✔ isolated memory  
✔ independent interpreters  
✔ safe parallel execution  

Multiprocessing bypasses the GIL by running multiple Python interpreters in
separate processes.

This module explains multiprocessing as a system design decision.

You will learn:

✔ Why multiprocessing exists  
✔ GIL-safe parallelism  
✔ Process isolation model  
✔ Inter-process communication (IPC) cost  
✔ Process pools and executors  
✔ CPU-bound architecture patterns  

---

## Section 1 — Why Processes, Not Threads

Threads share memory.

Processes do not.

Sharing memory requires fine-grained locking and corruption risk.

Python chose:

> Isolation over complexity.

Each process has:

✔ its own GIL  
✔ its own memory space  
✔ independent interpreter state  

This guarantees correctness.

Parallelism becomes safe by design.

---

## Section 2 — True CPU Parallelism

Processes run on separate cores simultaneously.

Threads cannot bypass the GIL.
Processes can.

### Exercise — Parallel CPU Work

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

Run with different core counts and observe speedup.

---

## Section 3 — Isolation Has a Cost

Processes do not share memory.

Data must be copied or serialized.

IPC (inter-process communication) is expensive.

Trade-off:

✔ Safe parallelism  
✘ Data sharing overhead  

Multiprocessing favors heavy computation over heavy communication.

---

## Section 4 — Process Pools and Executors

High-level APIs simplify process management.

### Example — ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor

def compute(x):
    return x * x

with ProcessPoolExecutor() as executor:
    results = executor.map(compute, range(10))

print(list(results))
```

You get:

✔ parallel execution  
✔ task abstraction  
✔ automatic worker management  

No manual process tracking required.

---

## Section 5 — Real-World Systems

Multiprocessing powers:

✔ ML training pipelines  
✔ simulation engines  
✔ batch analytics  
✔ video/image processing  
✔ distributed workers (Celery style)  
✔ scientific computation  

Any serious CPU-heavy Python system uses this model.

---

## Architectural Rule of Thumb

I/O-bound → threads or async  
CPU-bound → processes  
Mixed workloads → async + process pools  

Choosing the wrong model wastes hardware.

Understanding multiprocessing is architecture, not optimization.

---

## Final Takeaways

Multiprocessing exists to:

✔ bypass the GIL safely  
✔ achieve real parallelism  
✔ isolate memory  
✔ protect correctness  
✔ scale CPU-heavy workloads  

Isolation is a design feature, not a limitation.

---

## Suggested Extensions

1. Measure IPC overhead with large data
2. Compare pool sizes vs performance
3. Combine multiprocessing with async
4. Build parallel data pipeline
5. Benchmark CPU scaling on your machine

---

End of module.
