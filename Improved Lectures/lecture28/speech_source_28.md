# Speech Source — Lecture 28: Sharing NumPy Data with Multiprocessing — Zero-Copy Design

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 27 we covered coordination mechanisms: the poison-pill pattern for clean shutdown, shared flags for cooperative early-exit, and the cost spectrum from Manager objects to raw shared memory. We established that multiprocessing.Value and multiprocessing.Array use direct shared memory at nanosecond latency. Today we apply that principle to the most common high-performance data structure in scientific and data-engineering Python: the NumPy array. We examine why sending NumPy arrays through a Queue is catastrophic, how to use multiprocessing.Array as backing storage for zero-copy NumPy views, and how the Python 3.8 multiprocessing.shared_memory module enables clean, named shared memory for large arrays.

---

## CONCEPT 1.1 — Why Sending NumPy Arrays Through Queue Is Catastrophic

**Problem it solves:**
NumPy arrays are the primary data structure for numerical computation in Python. In multiprocessing pipelines — image processing, signal analysis, data-parallel model inference — developers naturally reach for Queue.put(array) to send data to workers. This is the wrong approach for any non-trivial array size. When a NumPy array is sent through a Queue, pickle serializes it: the array's dtype, shape, and raw bytes are all written into a bytes buffer. The buffer is written through the kernel pipe. The receiving process loads the bytes and reconstructs the array from scratch. For a 100-megabyte array, this is three copies and multiple seconds of overhead — per task dispatch.

**Why invented:**
NumPy arrays have a simple memory layout: a contiguous block of typed bytes with shape and stride metadata. This layout is ideal for shared memory: you can map the same physical memory page into multiple processes' virtual address spaces, and each process can create a NumPy array object that uses that page as its data buffer without copying a single byte. This is what numpy.frombuffer and multiprocessing.shared_memory enable: zero-copy sharing of the same physical data across process boundaries.

**What happens without it:**
A four-stage data pipeline that processes 50-megabyte arrays — load, filter, transform, analyze — and passes the array between stages through Queues copies the array twelve times (three copies per stage transition, four transitions). The pipeline consumes 600 megabytes of peak memory for transit alone, on top of the working memory needed for actual computation. At 30 arrays per second, the serialization throughput requirement is 1.8 gigabytes per second of pickling throughput. No Python process can sustain that.

**Industry impact:**
Every scientific computing, ML inference, and computer vision pipeline that uses multiprocessing must solve this problem. The standard solutions are: shared memory arrays that all workers access via NumPy views (read-only workers), or partitioned shared memory where each worker writes to a non-overlapping slice (parallel in-place transforms). Both patterns appear in NumPy documentation, scipy-image, OpenCV Python bindings, and every major ML inference server.

---

## CONCEPT 2.1 — multiprocessing.Array as NumPy Backing Storage

**Problem it solves:**
multiprocessing.Array is a typed, fixed-size shared memory buffer compatible with ctypes. When you create a multiprocessing.Array of ctypes.c_double, you get a block of raw bytes in shared memory. numpy.frombuffer takes that block of bytes and wraps it in a NumPy array object, giving you full NumPy semantics — vectorized operations, slicing, broadcasting — backed by shared memory. Workers receive the Array reference (which is just a shared memory handle, not the data), create their own frombuffer view, and operate directly on the shared data without any copying.

**Why invented:**
numpy.frombuffer was designed for exactly this pattern: given any object that exposes a raw bytes buffer via the Python buffer protocol — a bytes object, a bytearray, a ctypes array, a shared memory block — create a NumPy array that uses those bytes as its data. multiprocessing.Array implements the buffer protocol, making this zero-copy wrapping possible. The result is that the data lives in shared memory, and each process's NumPy array is just a view — a lightweight metadata object — over the same bytes.

**What happens without it:**
Without frombuffer, you would need to manually copy data from the shared Array into a local NumPy array for processing, then copy results back. This reintroduces the copy overhead that shared memory was meant to eliminate. The whole point of the shared memory approach is that the data never moves — only process views of the same data change.

**Industry impact:**
This pattern is used in parallel image processing pipelines, parallel audio processing, real-time sensor data analysis, and any workload where the same large dataset is processed by multiple workers who each operate on different regions. The pattern scales naturally: adding more workers does not increase memory usage because all workers share the same underlying buffer.

---

## EXAMPLE 2.1 — Sharing NumPy array via multiprocessing.Array (zero-copy view)

Narration: We create a 1,000,000-element float64 array in shared memory using multiprocessing.Array. We fill it with random values using numpy.frombuffer in the main process. We then spawn four workers, each responsible for squaring a different chunk of the array in-place. Each worker creates its own frombuffer view of the shared array and writes the squared values directly into shared memory. After all workers join, the main process's frombuffer view immediately reflects the changes — no result transfer needed. This is zero-copy parallel in-place transformation.

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

---

## CONCEPT 3.1 — multiprocessing.shared_memory for Python 3.8+ Zero-Copy

**Problem it solves:**
multiprocessing.Array requires specifying a ctypes element type and is tightly coupled to the ctypes type system. For teams working exclusively with NumPy, this coupling is awkward: you think in terms of dtypes, not ctypes. multiprocessing.shared_memory, introduced in Python 3.8, provides a cleaner abstraction: a named block of raw bytes identified by a string name, accessible from any process that knows the name. You create the shared memory block in the main process, write a NumPy array into it, then pass the name string to workers. Workers open the block by name, create a NumPy ndarray view over it, and work directly on the shared bytes.

**Why invented:**
shared_memory was designed to solve three limitations of multiprocessing.Array: it is not coupled to ctypes types, it supports arrays of any dtype including structured dtypes, and it provides a name-based lookup mechanism that works across unrelated processes — not just processes in the same Pool. The name mechanism also enables the pattern where a server process pre-loads a large dataset into shared memory and multiple independent client processes attach to it by name, all reading from the same physical memory without any copying.

**What happens without it:**
Without shared_memory, sharing a complex NumPy array — a structured array, a multi-dimensional array with non-trivial strides — through multiprocessing.Array requires manually linearizing the array, choosing a compatible ctypes type, and reshaping on the receiving end. For structured arrays or complex dtypes, this is error-prone. shared_memory eliminates the type mismatch by treating the backing store as raw bytes and letting numpy.ndarray interpret the layout via its dtype and shape arguments.

**Industry impact:**
multiprocessing.shared_memory is the recommended modern approach for large-array sharing in Python. ML model serving systems load model weights into shared memory once at startup and serve requests from multiple worker processes that all read from the same shared weights without any copying. Feature engineering pipelines load a dataset into shared memory and process it with a pool of workers, each reading a different subset of columns. The shared_memory API is also used in real-time systems where data must be accessible to multiple processes with minimal latency.

---

## EXAMPLE 3.1 — multiprocessing.shared_memory for Python 3.8+ zero-copy

Narration: We load 2,000,000 float64 values into a named shared memory block. We create four worker processes, each responsible for analyzing a different chunk of the array — computing mean, standard deviation, and maximum. Each worker opens the shared memory block by name, creates a NumPy ndarray view over it, and computes its statistics directly from shared memory without copying. The worker sends only the small statistics dictionary back through a Queue — tiny IPC payload, all heavy data access through shared memory. After all workers join, we unlink the shared memory to release the OS resource.

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

---

## CONCEPT 4.1 — Reading vs. Writing: When Locks Are Needed

**Problem it solves:**
A common question when workers share memory is: do I always need a lock? The answer depends on the access pattern. For read-only workers — all workers read from the shared array but none write to it — no lock is needed. The OS memory mapping is read-only for all processes, and concurrent reads of the same memory location are safe. For write workers — workers that write to non-overlapping regions of the array, like the chunk-based squaring in Example 28.1 — no lock is needed because no two workers ever access the same memory location simultaneously. Locks are needed only when two or more workers write to overlapping regions, or when one worker reads a region that another worker is simultaneously writing.

**Why invented:**
The distinction between read-read (always safe), read-write (requires coordination), and write-write to different locations (safe if non-overlapping, unsafe if overlapping) is fundamental to concurrent programming and is well-established in both operating system design and parallel algorithm theory. Shared memory in multiprocessing inherits these rules directly from hardware memory access semantics.

**What happens without it:**
A system where multiple workers write to the same array location without a lock produces undefined results — each worker's write may partially overwrite another's, producing garbage values that are neither of the two intended results. This is a data race on shared memory, and it is one of the most insidious bugs in concurrent programming because the corruption is non-deterministic and may not appear in testing but manifests under load in production.

**Industry impact:**
The standard design in parallel numerical computing is partitioned writes: divide the data structure into non-overlapping regions and assign each region to one worker. This eliminates all write-write conflicts without any locks. For reductions — summing all workers' results — each worker writes to a separate result slot, and the main process performs the final aggregation after all workers join. This is the MapReduce pattern at the shared-memory level.

---

## CONCEPT 5.1 — Final Takeaway Lecture 28

Sending NumPy arrays through a Queue causes three data copies and can consume seconds of serialization time for large arrays. The solution is shared memory: the array's raw bytes live in a memory region shared by all participating processes, and each process wraps those bytes in a lightweight NumPy view using numpy.frombuffer or numpy.ndarray with buffer argument. multiprocessing.Array is the pre-3.8 approach: tight ctypes coupling, but widely compatible. multiprocessing.shared_memory is the modern approach: raw bytes, any dtype, name-based access across unrelated processes. Lock discipline: concurrent reads are always safe; partitioned writes to non-overlapping regions are safe; overlapping writes require locks. The IPC payload in a well-designed shared-memory pipeline is tiny: file names, chunk indices, statistics — not the arrays themselves.

In the next lecture we combine async and multiprocessing into a hybrid architecture: the async front-end handles thousands of concurrent I/O connections while a process pool back-end handles CPU computation, bridged by loop.run_in_executor and ProcessPoolExecutor.
