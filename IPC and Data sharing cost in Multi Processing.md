# IPC and Data-Sharing Cost in Multiprocessing

Multiprocessing gives true CPU parallelism.

But there is a hidden cost:

👉 **data must move between processes**

And moving data is expensive.

Sometimes:

> copying data costs more than computing it

This section explains why that happens and how to design around it.

---

## Mental Model: Processes Do NOT Share Memory

Each process has:

- its own memory
- its own Python interpreter
- its own objects

When you send data to another process:

👉 Python serializes it  
👉 copies it  
👉 reconstructs it

This is called:

> IPC — Inter-Process Communication

IPC is not free.

It involves:

- pickling
- copying
- kernel transfer
- reconstruction

That overhead grows with data size.

---

## Real-World Scenario: Video Analytics Pipeline

Imagine a video analytics system:

- camera streams 4K frames
- each frame = large NumPy array
- worker processes analyze frames

A developer writes:

> “Send full frame to worker”

Each frame is copied between processes.

If a frame is 50 MB:

```
50 MB × 30 fps = 1.5 GB per second
```

The system becomes memory-bound, not CPU-bound.

Parallelism slows the pipeline.

This is a real production failure pattern.

---

## Demonstration: Data Transfer vs Computation

We simulate:

- small computation
- large data transfer

And show IPC dominating runtime.

---

## Example: expensive data copy

```python
import multiprocessing as mp
import numpy as np
import time

def worker(arr):
    # tiny computation
    return arr.sum()

if __name__ == "__main__":
    big_array = np.random.rand(20_000_000)  # large data

    start = time.time()

    with mp.Pool(4) as pool:
        results = pool.map(worker, [big_array] * 4)

    print("Time:", round(time.time() - start, 2))
```

What happens:

- array is copied 4 times
- computation is tiny
- copy dominates runtime

You pay for data movement, not CPU.

---

## Control Experiment: compute without copying

```python
start = time.time()
result = worker(big_array)
print("Single process:", round(time.time() - start, 2))
```

Often this is faster.

Why?

No IPC cost.

---

## Key Insight

Multiprocessing is beneficial only when:

```
computation >> data transfer
```

If data transfer dominates:

parallelism hurts performance.

---

## Real Engineering Rule

Never send large objects repeatedly.

Instead:

- share memory
- send indices
- send metadata
- send small messages

Move computation to the data,
not data to computation.

---

## Shared Memory Optimization

We redesign the pipeline:

- allocate data once
- workers access shared memory
- no copying

---

## Example: shared NumPy array

```python
from multiprocessing import Process, shared_memory
import numpy as np

def worker(name, shape):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)

    print(arr.sum())

    shm.close()

if __name__ == "__main__":
    data = np.random.rand(20_000_000)

    shm = shared_memory.SharedMemory(create=True, size=data.nbytes)
    shared = np.ndarray(data.shape, dtype=data.dtype, buffer=shm.buf)
    shared[:] = data[:]

    p = Process(target=worker, args=(shm.name, data.shape))
    p.start()
    p.join()

    shm.close()
    shm.unlink()
```

Now:

- zero copying
- true parallel computation
- scalable architecture

---

## Real-World Systems That Hit This Problem

This issue appears in:

- video analytics pipelines
- ML training systems
- scientific simulations
- financial Monte Carlo engines
- satellite image processing
- big data ETL pipelines
- GPU preprocessing
- distributed analytics

Any system moving large arrays between processes.

---

## Design Strategy

Think in terms of:

> data locality

Good multiprocessing design:

✔ compute near the data  
✔ minimize copying  
✔ batch messages  
✔ share memory  
✔ send references, not payloads

Bad design:

❌ passing giant objects repeatedly

---

## Architecture Upgrade Pattern

Instead of:

```
main → send huge object → worker
```

Use:

```
shared memory → worker reads directly
```

Or:

```
worker owns data
main sends instructions only
```

This flips architecture from data-transfer-heavy
to compute-heavy.

That’s where multiprocessing shines.

---

## Final Principle

Parallel systems are limited by:

> bandwidth, not just CPU

Data movement is often the true bottleneck.

Professional performance engineering is about:

reducing data motion.

Not just adding cores.
