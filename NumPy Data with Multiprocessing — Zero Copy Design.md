# Sharing NumPy Data with Multiprocessing — Zero Copy Design

Multiprocessing gives true parallel CPU execution.

But NumPy arrays are large.

And large arrays expose a hidden performance trap:

👉 multiprocessing copies data by default

This section explains:

- why copying NumPy arrays kills performance
- when multiprocessing becomes slower than serial
- how to share arrays safely
- real-world architecture patterns

This is a critical concept in scientific and data systems.

---

## Mental Model: Each Process Has Its Own Memory

In multiprocessing:

- processes do NOT share Python objects
- every object sent to a worker is copied
- NumPy arrays are serialized and reconstructed

This is fine for small objects.

It is catastrophic for large matrices.

---

## Real-World Scenario: Medical Image Pipeline

Imagine a hospital imaging system:

- 3D MRI scans
- each scan = 800 MB NumPy array
- workers run segmentation algorithms
- 8 parallel processes

Naive design:

```
send full array to each process
```

Memory required:

```
800 MB × 8 = 6.4 GB
```

System crashes.

Even if memory survives:

copying dominates runtime.

Parallel becomes slower than serial.

This is a real production failure pattern.

---

## Demonstration: Copy Overhead

```python
import multiprocessing as mp
import numpy as np
import time

def worker(arr):
    return arr.sum()

if __name__ == "__main__":
    data = np.random.rand(50_000_000)  # large array

    start = time.time()
    with mp.Pool(4) as pool:
        pool.map(worker, [data] * 4)

    print("With copying:", round(time.time() - start, 2))
```

Even though the computation is small:

runtime is large.

Why?

The array is copied 4 times.

Data transfer dominates compute.

---

## Key Principle

Multiprocessing is efficient only when:

```
compute cost >> data copy cost
```

Large arrays violate this rule.

We must eliminate copying.

---

## Shared Memory: Zero-Copy NumPy

Python provides:

```
multiprocessing.shared_memory
```

We allocate array once.

Workers attach to the same memory block.

No copies.

True parallelism.

---

## Example: Shared NumPy Array

```python
from multiprocessing import Process, shared_memory
import numpy as np

def worker(name, shape):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)

    arr += 1  # modify shared data

    shm.close()

if __name__ == "__main__":
    data = np.zeros((5_000_000,), dtype=np.float64)

    shm = shared_memory.SharedMemory(create=True, size=data.nbytes)
    shared = np.ndarray(data.shape, dtype=data.dtype, buffer=shm.buf)
    shared[:] = data[:]

    processes = [Process(target=worker, args=(shm.name, data.shape)) for _ in range(4)]

    for p in processes: p.start()
    for p in processes: p.join()

    print(shared[:10])

    shm.close()
    shm.unlink()
```

Now:

- one copy in RAM
- all workers operate in-place
- zero serialization overhead

This is true shared memory parallelism.

---

## Architecture Pattern: Data Ownership

Better design:

```
main allocates data
workers operate on slices
workers never duplicate arrays
```

Example:

```python
def worker(name, shape, start, end):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)

    arr[start:end] *= 2

    shm.close()
```

Each worker touches only its slice.

No conflicts.

No locks needed.

This is embarrassingly parallel.

---

## Real Systems Using This Pattern

This design appears in:

- deep learning pipelines
- video analytics engines
- satellite image processing
- large-scale simulations
- scientific computing clusters
- financial Monte Carlo engines
- GPU preprocessing pipelines
- genomics analysis

Any system processing large matrices.

---

## Safety Considerations

Shared memory is powerful but dangerous.

Workers can overwrite each other.

Design rules:

- divide memory into disjoint slices
- never overlap writes
- read-only access is safest
- use locks only if unavoidable

Most high-performance systems avoid locks by design.

---

## When Not to Use Shared Memory

If arrays are small:

copying is cheaper than complexity.

If tasks are short:

overhead dominates.

If correctness matters more than speed:

prefer safe message passing.

Engineering is about tradeoffs.

---

## Final Insight

Multiprocessing speed is limited by:

> data movement

The fastest parallel systems:

- minimize copying
- maximize locality
- operate in-place
- avoid synchronization

Compute is cheap.

Memory bandwidth is expensive.

Design for memory first.
