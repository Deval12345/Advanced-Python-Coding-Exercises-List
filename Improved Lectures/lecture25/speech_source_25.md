# Lecture 25: Multiprocessing — True Parallelism Beyond the GIL
## Speech Source (Master Synchronization File)

---

## CONCEPT 0.1 — Transition

We have spent the last four lectures living inside async/await — single-threaded cooperative concurrency, event loops, coroutines, task groups, semaphores. Before that, we studied threading for I/O-bound work, and we studied the GIL to understand why threads fail on CPU-bound work. The GIL lecture ended with a promise: the correct fix for CPU-bound parallelism is multiprocessing. Today we cash that promise in. Today we go deep on multiprocessing — what it actually does at the OS level, how to communicate between processes correctly, and how the high-level Pool API makes it practical for real workloads.

---

## CONCEPT 1.1 — The Process Model: Isolation by Design

**Problem it solves:**
Threads share memory. In CPython, that shared memory is protected by the GIL, which means only one thread can execute Python bytecode at a time. For CPU-bound work — numerical computation, image processing, data transformation in pure Python — the GIL prevents any real parallelism. We need a concurrency model that gets around the GIL entirely. The answer: do not share memory at all.

**Why invented:**
Python's multiprocessing module, introduced in Python 2.6 and refined significantly through Python 3, creates entirely separate OS processes — each with its own Python interpreter, its own GIL, its own memory space, and its own set of loaded modules. Because each process is a completely independent interpreter, there is no GIL to contend over. The OS can schedule each process on a separate CPU core simultaneously. True parallelism — not concurrent waiting, but concurrent computation.

**What happens without it:**
Consider a media company's image processing pipeline: 10,000 photos need to be resized for a web CDN. A pure Python thread approach puts all four CPU cores to work — except the GIL ensures only one core is ever running Python bytecode at any moment. Three cores sit idle. The pipeline processes every image serially, one after another. With multiprocessing, four processes each take 2,500 images. Each runs on its own core. Total runtime drops by approximately 4x — a real, measurable, production speedup.

**Industry impact:**
Multiprocessing is the backbone of CPU-bound batch computation in Python: NumPy/SciPy preprocessing pipelines, video transcoding, machine learning feature engineering, scientific simulation, and any workload that is embarrassingly parallel — where tasks are independent and can be split across workers with no communication needed during computation. The multiprocessing module ships in the Python standard library; no third-party dependency is required for this level of parallelism.

---

## CONCEPT 2.1 — multiprocessing.Process: Spawning Individual Processes

**Problem it solves:**
The most direct way to achieve true parallelism is to create individual processes explicitly — one process per task or per logical worker. Python's multiprocessing.Process class is the low-level primitive that makes this possible.

**Why invented:**
multiprocessing.Process mirrors threading.Thread in its API deliberately. The familiar pattern — create, configure, start, join — transfers directly from the threading world. The internal mechanism is completely different: instead of creating an OS thread within the current process, Process creates a new OS process with its own interpreter.

**What happens without it:**
Process start methods matter significantly by platform. On Linux, the default is fork: the child process gets a complete copy of the parent's memory space through copy-on-write semantics — fast startup, but any open file descriptors, locks, and sockets are duplicated, which can cause subtle bugs. On macOS (since Python 3.8) and Windows, the default is spawn: a fresh Python interpreter starts, imports the target module from scratch, and calls the target function. Spawn is safer but slower — roughly 50-200ms per process. On all platforms with spawn, the if __name__ == "__main__" guard is mandatory. Without it, each spawned child reimports the module and re-executes the top level, which spawns more children, which spawn more children — an infinite fork bomb that will consume all system resources.

**Industry impact:**
Daemon processes (daemon=True) are a critical production pattern — they automatically terminate when the parent process exits, making them appropriate for background monitoring workers or heartbeat threads that should not hold the program open. However, daemon processes cannot spawn their own child processes, and they may be killed before completing critical cleanup. Process objects are heavyweight — 50 to 200ms startup time and 30 to 100MB of memory per process on typical workloads. Creating one process per tiny task defeats the purpose; process pools amortize this startup cost across many tasks.

---

## EXAMPLE 2.1 — Basic multiprocessing.Process with shared Manager list

**Narration:**
Let's watch four processes run in genuine parallel. Each worker is assigned an ID and a numeric input, does two million multiplications — real CPU work — and writes its result into a shared Manager list. The Manager creates a server process that hosts the shared list; all four worker processes communicate with it through IPC. We record the elapsed time before and after. On a multi-core machine, all four computations overlap in time, and the total wall-clock duration is approximately the time for one computation, not four.

```python
# Example 25.1
import multiprocessing                                 # line 1
import time                                            # line 2
import os                                              # line 3

def cpuWorker(workerId, inputValue, resultList):       # line 4
    pid = os.getpid()                                  # line 5
    print(f"Worker {workerId} in PID {pid}")           # line 6
    total = sum(i * inputValue for i in range(2_000_000))  # line 7
    resultList.append((workerId, total))               # line 8
    print(f"Worker {workerId} done: {total}")          # line 9

if __name__ == "__main__":                             # line 10
    manager = multiprocessing.Manager()                # line 11
    resultList = manager.list()                        # line 12

    processes = []                                     # line 13
    start = time.perf_counter()                        # line 14

    for workerId in range(4):                          # line 15
        p = multiprocessing.Process(                   # line 16
            target=cpuWorker,                          # line 17
            args=(workerId, workerId + 1, resultList)  # line 18
        )                                              # line 19
        processes.append(p)                            # line 20
        p.start()                                      # line 21

    for p in processes:                                # line 22
        p.join()                                       # line 23

    elapsed = time.perf_counter() - start              # line 24
    print(f"All workers done in {elapsed:.2f}s")       # line 25
    print(f"Results: {list(resultList)}")              # line 26
```

---

## CONCEPT 3.1 — Inter-Process Communication (IPC): The Cost of Isolation

**Problem it solves:**
Processes cannot share memory directly. If one process computes a result, it cannot simply write to a variable that another process reads — those two variables live in completely separate memory spaces. Every piece of data that moves between processes must be explicitly serialized, transmitted through a kernel-mediated channel, and deserialized on the other side. This is Inter-Process Communication, and its cost shapes everything about how you design a multiprocessing system.

**Why invented:**
Python provides several IPC primitives tuned to different use cases:
1. multiprocessing.Queue: a process-safe FIFO backed by a pipe and internal locks. Any process can put items in, any process can get items out. Everything is pickled (serialized) before transmission and unpickled after. Suitable for task distribution and result collection.
2. multiprocessing.Pipe: a bidirectional channel between exactly two endpoints. Lower overhead than Queue for direct point-to-point communication. Returns two Connection objects; each can send and receive.
3. multiprocessing.Value and multiprocessing.Array: shared memory segments with optional lock protection. Suitable for small numeric data — counters, flags, progress indicators — that needs to be read and written by multiple processes without full pickling overhead.
4. multiprocessing.Manager: a dedicated server process that hosts shared Python objects — lists, dicts, Namespaces. Very flexible; any picklable Python object can be shared. Slowest option because every operation crosses a process boundary through IPC.

**What happens without it:**
The pickling cost is the silent killer of multiprocessing performance. Sending a 100MB NumPy array through a Queue means: pickle it (copy it once into bytes), write those bytes through a pipe (copy two), unpickle it (copy three). A workload that sends large data to workers and receives large results back can spend more time in IPC serialization than in actual computation. This is why the golden rule of multiprocessing is: minimize what crosses process boundaries. Send small task parameters to workers. Receive small results. Do all heavy data manipulation inside the worker process, entirely in its own memory space.

**Industry impact:**
Production systems built on multiprocessing structure their work carefully around this constraint. A video transcoding pipeline does not send raw video frames through a Queue — it sends file paths. Each worker reads its own file from disk. The inter-process messages are paths (a few bytes), not frames (megabytes). A machine learning preprocessing system does not send full datasets to workers — it sends row index ranges, and each worker reads its own slice from a memory-mapped file. Understanding pickling cost is what separates a multiprocessing design that scales from one that slows down compared to sequential execution.

---

## EXAMPLE 3.1 — multiprocessing.Queue producer-consumer across processes

**Narration:**
Here is the producer-consumer pattern extended to real processes. The main process (producer) puts prime-counting tasks — just integer limits — onto a task queue. Four worker processes each loop, pulling tasks off the queue, counting primes up to that limit, and putting results onto a result queue. When there are no more tasks, the main process sends a None sentinel for each worker. Each worker, upon receiving None, breaks its loop and exits cleanly. The main process collects all results from the result queue, then joins the workers. Notice how small the inter-process messages are: integers going in, tuples of two integers coming out. The heavy computation happens entirely inside each worker's private memory space.

```python
# Example 25.2
import multiprocessing                                 # line 1
import time                                            # line 2
import math                                            # line 3

def primeWorker(taskQueue, resultQueue):               # line 4
    while True:                                        # line 5
        task = taskQueue.get()                         # line 6
        if task is None:                               # line 7
            break                                      # line 8
        limit = task                                   # line 9
        count = sum(                                   # line 10
            1 for n in range(2, limit)                 # line 11
            if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1))  # line 12
        )                                              # line 13
        resultQueue.put((limit, count))                # line 14

if __name__ == "__main__":                             # line 15
    taskQueue = multiprocessing.Queue()                # line 16
    resultQueue = multiprocessing.Queue()              # line 17

    numWorkers = 4                                     # line 18
    workerLimits = [20000, 30000, 25000, 35000, 40000, 15000]  # line 19

    workers = [                                        # line 20
        multiprocessing.Process(target=primeWorker, args=(taskQueue, resultQueue))  # line 21
        for _ in range(numWorkers)                     # line 22
    ]                                                  # line 23
    for w in workers: w.start()                        # line 24

    for limit in workerLimits:                         # line 25
        taskQueue.put(limit)                           # line 26

    for _ in range(numWorkers):                        # line 27
        taskQueue.put(None)                            # line 28

    results = []                                       # line 29
    for _ in range(len(workerLimits)):                 # line 30
        results.append(resultQueue.get())              # line 31

    for w in workers: w.join()                         # line 32
    print("Results:", sorted(results))                 # line 33
```

---

## CONCEPT 4.1 — multiprocessing.Pool: The High-Level Process Pool

**Problem it solves:**
Manually managing Process objects, Queues, sentinels, and the producer-consumer protocol is verbose, error-prone, and has to be rewritten for every use case. The pattern — create N workers, distribute tasks, collect results, shut down — is so universal that Python provides it out of the box: multiprocessing.Pool.

**Why invented:**
Pool creates N worker processes at startup and keeps them alive for the duration of the pool's lifetime. This amortizes the 50-200ms process startup cost across potentially thousands of tasks. The API is deliberately modeled after the concurrent.futures interface that many students already know from threading work:
- pool.map(fn, iterable): distributes items across workers, collects results in submission order (blocking until all results arrive)
- pool.imap(fn, iterable): lazy iterator that yields results as they complete, saving memory when the input is a large sequence
- pool.apply_async(fn, args): submits a single task asynchronously, returns an AsyncResult object whose .get() method blocks until the result is available
- Used as a context manager with the with statement, Pool shuts down its workers cleanly when the block exits
The chunksize argument for pool.map groups items into batches that are sent to each worker in a single IPC message, dramatically reducing queue overhead when processing many small tasks.

**What happens without it:**
Without Pool, every batch job requires writing the full process lifecycle management: Queue setup, sentinel design, worker loops, result collection, join logic. With Pool, a parallel map over 10,000 images is three lines. The developer focuses on the computation, not the infrastructure. The trade-off is that Pool is less flexible than raw Process management — it is designed for the common case of independent tasks (no inter-worker communication), not for complex worker topologies.

**Industry impact:**
Pool.map is the entry point for most production multiprocessing usage in Python. Data engineering pipelines use it to process partitions of large datasets. Scientific computing uses it to parallelize simulations across parameter ranges. Web crawlers use it to process downloaded pages. The pattern is so clean and Pythonic that many engineers reach for it as their first tool when any workload is CPU-bound and embarrassingly parallel. For more sophisticated patterns — dynamic task generation, heterogeneous worker types, streaming results — raw Process and Queue give the necessary control. For the 80% case, Pool is exactly the right level of abstraction.

---

## EXAMPLE 4.1 — multiprocessing.Pool for parallel image processing simulation

**Narration:**
We simulate an image resizing pipeline — the canonical multiprocessing use case. Each image spec is a tuple: an ID, a width, and a height. The resize simulation computes per-pixel work proportional to the image's pixel count, then returns the new dimensions. First we run the pipeline sequentially: one image at a time in the main process. Then we run it with Pool using four worker processes. We time both, compute the speedup, and print a sample result to verify correctness. The Pool version distributes the eight images across four workers, so ideally two images per worker run in parallel. The actual speedup depends on process startup amortization and core availability.

```python
# Example 25.3
import multiprocessing                                 # line 1
import time                                            # line 2
import math                                            # line 3

def simulateImageResize(imageSpec):                    # line 4
    imageId, width, height = imageSpec                 # line 5
    pixelCount = width * height                        # line 6
    total = 0                                          # line 7
    for pixel in range(pixelCount // 100):             # line 8
        total += math.sqrt(pixel + 1)                  # line 9
    newWidth = width // 2                              # line 10
    newHeight = height // 2                            # line 11
    return (imageId, newWidth, newHeight, round(total, 2))  # line 12

if __name__ == "__main__":                             # line 13
    images = [(i, 1920, 1080) for i in range(8)]       # line 14

    start = time.perf_counter()                        # line 15
    sequentialResults = [simulateImageResize(img) for img in images]  # line 16
    seqTime = time.perf_counter() - start              # line 17
    print(f"Sequential: {seqTime:.2f}s")               # line 18

    start = time.perf_counter()                        # line 19
    with multiprocessing.Pool(processes=4) as pool:    # line 20
        poolResults = pool.map(simulateImageResize, images)  # line 21
    poolTime = time.perf_counter() - start             # line 22
    print(f"Pool (4 procs): {poolTime:.2f}s")          # line 23
    print(f"Speedup: {seqTime/poolTime:.1f}x")         # line 24
    print("Sample result:", poolResults[0])            # line 25
```

---

## CONCEPT 5.1 — Final Takeaway Lecture 25

Multiprocessing bypasses the GIL by running separate Python interpreter instances in separate OS processes. Each process has its own memory space — isolation prevents data races but requires explicit IPC for communication. All data crossing process boundaries is pickled (serialized and deserialized), so keep inter-process communication minimal: send small task parameters, receive small results, and do heavy computation entirely inside the worker.

multiprocessing.Queue and Pipe are the fundamental communication primitives. Value and Array enable fast shared memory for small numeric data. multiprocessing.Pool provides the high-level process pool API that most production code should reach for first: pool.map for blocking ordered results, pool.imap for lazy streaming, pool.apply_async for single async tasks.

Process creation is heavyweight — 50 to 200ms startup, 30 to 100MB per process. Never create one process per tiny task. Use pools to amortize startup cost across many tasks. The if __name__ == "__main__" guard is mandatory — never omit it. Always join your processes; unjoinable processes become zombies that hold system resources.

Next lecture: ProcessPoolExecutor from concurrent.futures as the high-level alternative to Pool, IPC cost analysis and when to prefer shared memory over queues, and a frank discussion of when multiprocessing hurts more than it helps — the scenarios where process overhead dominates computation time and sequential code wins.
