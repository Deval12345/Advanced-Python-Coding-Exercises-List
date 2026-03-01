# Code — Lecture 28: Sharing NumPy Data with Multiprocessing — Zero-Copy Design

---

## Example 28.1 — Sharing NumPy array via multiprocessing.Array (zero-copy view)

```python
# Example 28.1
import multiprocessing
import numpy as np
import time
import ctypes

def squareInPlace(sharedArray, length, workerId, numWorkers):
    arr = np.frombuffer(sharedArray, dtype=np.float64)  # zero-copy view
    chunkSize = length // numWorkers
    start = workerId * chunkSize
    end = start + chunkSize if workerId < numWorkers - 1 else length
    arr[start:end] = arr[start:end] ** 2

if __name__ == "__main__":
    size = 1_000_000
    sharedArray = multiprocessing.Array(ctypes.c_double, size)
    arr = np.frombuffer(sharedArray, dtype=np.float64)
    arr[:] = np.random.rand(size)
    originalSum = arr.sum()
    
    numWorkers = 4
    workers = [
        multiprocessing.Process(target=squareInPlace, args=(sharedArray, size, i, numWorkers))
        for i in range(numWorkers)
    ]
    start = time.perf_counter()
    for w in workers: w.start()
    for w in workers: w.join()
    elapsed = time.perf_counter() - start
    print(f"Shared memory squaring: {elapsed:.3f}s")
    print(f"Result sum: {arr.sum():.4f}")
```

**Line-by-line explanation:**

- **Line 7:** `squareInPlace` is the worker function. It receives the shared array handle — not the data. The handle is a small object (dozens of bytes) that identifies the shared memory region.
- **Line 8:** `np.frombuffer(sharedArray, dtype=np.float64)` — this is the zero-copy view creation. frombuffer does not allocate memory. It creates a NumPy array whose data pointer points directly into the bytes of `sharedArray`. The dtype must match exactly what was stored — here, float64 matches ctypes.c_double.
- **Lines 9-11:** Chunk boundary computation. Worker zero gets indices 0 to chunkSize. Worker numWorkers-1 gets the remainder (handles the case where size is not exactly divisible by numWorkers).
- **Line 12:** `arr[start:end] = arr[start:end] ** 2` — reads the chunk from shared memory, squares each element, writes back into shared memory. This is an in-place operation on shared bytes. No intermediate array allocation is needed here because NumPy can perform the operation in-place.
- **Line 16:** `multiprocessing.Array(ctypes.c_double, size)` — allocates `size * 8` bytes (8 bytes per double) in a memory-mapped shared region. ctypes.c_double corresponds to numpy's float64. The mapping is backed by the OS and shared with all child processes that fork from this parent.
- **Line 17:** `np.frombuffer(sharedArray, dtype=np.float64)` in the main process — the same zero-copy view creation as in the worker. The main process now has a NumPy array whose data is the shared memory.
- **Line 18:** `arr[:] = np.random.rand(size)` — fills the shared memory with random values. This write is through the NumPy view but the bytes are being written to shared memory. Workers created after this point will see these values.
- **Lines 21-26:** Workers are created with `sharedArray` as an argument. When spawned, the child processes inherit the shared memory mapping — they do not receive a copy of the data, only a handle to the same OS-managed shared region.
- **Lines 27-28:** Start all workers. All four run in parallel, each writing to its own non-overlapping chunk.
- **Line 29:** After join, all worker writes are complete and visible through the main process's `arr` view.
- **Line 31:** `arr.sum()` reads the result directly from shared memory — no IPC needed for result retrieval.

**Expected output:**
```
Shared memory squaring: 0.043s
Result sum: 333412.7812
```
The result sum is approximately size/3 because E[x^2] for x uniform on [0,1] is 1/3.

---

## Example 28.2 — multiprocessing.shared_memory for Python 3.8+ zero-copy

```python
# Example 28.2
import multiprocessing
import multiprocessing.shared_memory as shm
import numpy as np
import time

def analyzeChunk(sharedName, shape, dtype, chunkStart, chunkEnd, resultQueue):
    existing = shm.SharedMemory(name=sharedName)
    arr = np.ndarray(shape, dtype=dtype, buffer=existing.buf)
    chunk = arr[chunkStart:chunkEnd]
    result = {
        "start": chunkStart,
        "end": chunkEnd,
        "mean": float(chunk.mean()),
        "std": float(chunk.std()),
        "max": float(chunk.max())
    }
    resultQueue.put(result)
    existing.close()

if __name__ == "__main__":
    size = 2_000_000
    data = np.random.rand(size).astype(np.float64)
    
    sharedMem = shm.SharedMemory(create=True, size=data.nbytes)
    sharedArr = np.ndarray(data.shape, dtype=data.dtype, buffer=sharedMem.buf)
    sharedArr[:] = data
    
    resultQueue = multiprocessing.Queue()
    numWorkers = 4
    chunkSize = size // numWorkers
    
    workers = []
    for i in range(numWorkers):
        chunkStart = i * chunkSize
        chunkEnd = chunkStart + chunkSize if i < numWorkers - 1 else size
        p = multiprocessing.Process(
            target=analyzeChunk,
            args=(sharedMem.name, data.shape, data.dtype, chunkStart, chunkEnd, resultQueue)
        )
        workers.append(p)
    
    start = time.perf_counter()
    for w in workers: w.start()
    for w in workers: w.join()
    elapsed = time.perf_counter() - start
    
    results = [resultQueue.get() for _ in range(numWorkers)]
    sharedMem.close()
    sharedMem.unlink()
    
    print(f"Zero-copy analysis of {size:,} elements in {elapsed:.3f}s")
    for r in sorted(results, key=lambda x: x['start']):
        print(f"  Chunk {r['start']}-{r['end']}: mean={r['mean']:.4f}, std={r['std']:.4f}")
```

**Line-by-line explanation:**

- **Line 7:** `analyzeChunk` receives only: the shared memory name (a string), shape, dtype, chunk boundaries, and result queue. The actual data — 16 MB — is never sent through any IPC channel.
- **Line 8:** `shm.SharedMemory(name=sharedName)` — attaches to an existing shared memory block by name. This is a lookup in the OS shared memory registry; it does not copy any data. The `existing.buf` property is a memoryview of the shared bytes.
- **Line 9:** `np.ndarray(shape, dtype=dtype, buffer=existing.buf)` — creates a NumPy array using the shared memory bytes as its data buffer. The shape and dtype tell NumPy how to interpret the raw bytes. No allocation, no copy.
- **Line 10:** `chunk = arr[chunkStart:chunkEnd]` — creates a slice view of the NumPy array. This is also zero-copy: a slice creates a new NumPy array object pointing into the same data at the offset position.
- **Lines 11-17:** Compute statistics on the chunk. All computation reads directly from shared memory through the NumPy view. `float()` conversion ensures the values are plain Python floats, not NumPy scalars, which pickle more efficiently.
- **Line 18:** `resultQueue.put(result)` — puts the small statistics dictionary (around 150 bytes) through the Queue. This is the only IPC in the hot path — not the 16 MB array, just the tiny result.
- **Line 19:** `existing.close()` — releases the worker's mapping of the shared memory. This does not free the shared memory; the main process still holds it open. Calling close is good practice: it releases virtual address space in the worker process.
- **Line 24:** `shm.SharedMemory(create=True, size=data.nbytes)` — allocates a new shared memory block. `create=True` fails if a block with the same name already exists. `size=data.nbytes` is the exact byte count of the NumPy array.
- **Line 25:** `np.ndarray(data.shape, dtype=data.dtype, buffer=sharedMem.buf)` — wrap the shared memory as a NumPy array in the main process. Same pattern as in the worker.
- **Line 26:** `sharedArr[:] = data` — copy the data into shared memory once. After this, the original `data` array could be deleted; `sharedArr` (backed by shared memory) holds all the values.
- **Lines 32-41:** Create workers passing `sharedMem.name` — a short string — as the IPC payload for data access. The name is typically a system-generated identifier like `/psm_XXXXXXXX`.
- **Lines 47-48:** `sharedMem.close()` — release the main process's mapping. `sharedMem.unlink()` — destroy the OS resource. Unlink must be called exactly once, by the creator. If unlink is omitted, the OS keeps the shared memory allocated until the system restarts.

**Expected output:**
```
Zero-copy analysis of 2,000,000 elements in 0.078s
  Chunk 0-500000: mean=0.4998, std=0.2887
  Chunk 500000-1000000: mean=0.5001, std=0.2888
  Chunk 1000000-1500000: mean=0.5002, std=0.2886
  Chunk 1500000-2000000: mean=0.4997, std=0.2887
```
All chunks have mean close to 0.5 and standard deviation close to 0.2887 (theoretical standard deviation of uniform [0,1] distribution is 1/sqrt(12) ≈ 0.2887).
