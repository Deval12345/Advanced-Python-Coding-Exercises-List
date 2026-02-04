
# Sharing NumPy Data in Multiprocessing — Zero Copy Design

## Overview

Multiprocessing gives true parallel CPU execution.

But NumPy arrays are large.

Naive multiprocessing copies arrays between processes.

Copying dominates runtime.

Parallel becomes slower than serial.

This module explains zero-copy shared NumPy architecture.

The goal:

parallel compute without data duplication.

---

## Section 1 — Why Copying Breaks Performance

Each process has its own memory.

Objects passed to workers are serialized and copied.

Large NumPy arrays multiply memory usage.

Memory bandwidth becomes the bottleneck.

Rule:

compute must outweigh copy cost.

Large arrays violate this rule.

---

## Demonstration — Copy Overhead

```python
import multiprocessing as mp
import numpy as np
import time

def worker(arr):
    return arr.sum()

if __name__ == "__main__":
    data = np.random.rand(50_000_000)

    start = time.time()
    with mp.Pool(4) as pool:
        pool.map(worker, [data] * 4)

    print("With copying:", round(time.time() - start, 2))
```

Runtime dominated by data transfer.

---

## Section 2 — Shared Memory Solution

Python provides:

multiprocessing.shared_memory

Allocate array once.

Workers attach to same memory.

No copies.

True parallelism.

---

## Shared NumPy Example

```python
from multiprocessing import Process, shared_memory
import numpy as np

def worker(name, shape):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)

    arr += 1

    shm.close()

if __name__ == "__main__":
    data = np.zeros((5_000_000,), dtype=np.float64)

    shm = shared_memory.SharedMemory(create=True, size=data.nbytes)
    shared = np.ndarray(data.shape, dtype=data.dtype, buffer=shm.buf)
    shared[:] = data[:]

    procs = [Process(target=worker, args=(shm.name, data.shape)) for _ in range(4)]

    for p in procs: p.start()
    for p in procs: p.join()

    print(shared[:10])

    shm.close()
    shm.unlink()
```

Zero serialization.
One memory copy.
All workers operate in-place.

---

## Section 3 — Slice Ownership Pattern

Avoid race conditions via partitioning.

Each worker owns a slice.

```python
def worker(name, shape, start, end):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)

    arr[start:end] *= 2

    shm.close()
```

No overlap.
No locks required.

Embarrassingly parallel.

---

## Safety Rules

Shared memory is powerful but dangerous.

Follow:

- disjoint slices
- read-only sharing when possible
- avoid overlapping writes
- locks only as last resort

High-performance systems avoid synchronization by design.

---

## When Not to Use Shared Memory

If arrays are small → copying is cheaper

If tasks are short → overhead dominates

If correctness > speed → message passing safer

Engineering is trade-offs.

---

## Final Takeaways

Multiprocessing speed is limited by data movement.

The fastest systems:

- minimize copying
- maximize locality
- operate in-place
- partition memory
- avoid synchronization

Compute is cheap.
Memory bandwidth is expensive.

Design for memory first.
