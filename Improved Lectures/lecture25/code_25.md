# Code — Lecture 25: Multiprocessing — True Parallelism Beyond the GIL

---

## Example 25.1 — Basic multiprocessing.Process with shared Manager list

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

**Line-by-line explanation:**

- **Line 3:** `os.getpid()` — each worker prints its own OS process ID, making it visible that these are genuinely different processes, not threads.
- **Line 4:** `cpuWorker` is defined at module level, not inside `if __name__ == "__main__"`. Worker functions must be picklable to be passed to child processes; nested or lambda functions are not picklable.
- **Line 7:** `sum(i * inputValue for i in range(2_000_000))` — two million multiplications. This is genuine CPU work that the GIL would serialize across threads but that multiple processes can run simultaneously.
- **Line 8:** `resultList.append(...)` — writing to a Manager list. The Manager hosts the list in a dedicated server process. This call crosses a process boundary via IPC, which is why Manager lists are slower than regular lists but work safely across processes.
- **Line 10:** The mandatory `if __name__ == "__main__"` guard. Without this on macOS and Windows, the spawn start method re-imports this script in each child, re-executes the loop, and spawns infinite children.
- **Line 11:** `multiprocessing.Manager()` creates a dedicated server process. This process hosts the shared list and coordinates access from all four workers.
- **Line 16-21:** Creating and starting processes. `p.start()` returns immediately — the process starts asynchronously. The main process does not wait here; it continues to the next loop iteration and starts the next process.
- **Lines 22-23:** `p.join()` — wait for each process to finish. Without join, the main process could exit before workers complete, leaving orphan processes.
- **Line 24-26:** Timing and results. The elapsed time should be close to the time for ONE computation (not four), confirming genuine parallelism.

---

## Example 25.2 — multiprocessing.Queue producer-consumer across processes

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

**Line-by-line explanation:**

- **Lines 4-14:** `primeWorker` runs in each child process. It loops indefinitely, pulling tasks from the task queue. Notice the task is just an integer — a few bytes — not a large data structure.
- **Lines 6-8:** The sentinel check. `taskQueue.get()` blocks until a task is available. When it receives `None`, the worker breaks out of the loop and the function returns, causing the process to exit cleanly.
- **Lines 10-13:** The actual computation — prime counting using trial division. This runs entirely inside the worker's private memory. No data about the intermediate results crosses process boundaries.
- **Line 14:** `resultQueue.put((limit, count))` — the result is a tuple of two integers: tiny IPC payload. The entire worker loop sends small parameters in and small results out.
- **Line 16-17:** Two separate queues: one for task distribution (main → workers) and one for result collection (workers → main). This is the canonical producer-consumer topology.
- **Lines 25-26:** The main process fills the task queue. `taskQueue.put()` is non-blocking here because the queue is unbounded. Workers will pull tasks as they become available.
- **Lines 27-28:** One `None` sentinel per worker. If a worker gets `None`, it exits. Since there are four workers, we need four sentinels. Order of sentinel consumption is non-deterministic — each worker takes whichever sentinel it gets.
- **Lines 29-31:** Collect results from the result queue. We know exactly how many results to expect: one per task (six tasks). Calling `resultQueue.get()` six times collects all results.
- **Lines 32:** `w.join()` for all workers — wait for clean shutdown before the main process exits.

---

## Example 25.3 — multiprocessing.Pool for parallel image processing simulation

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

**Line-by-line explanation:**

- **Line 4:** `simulateImageResize` takes a single tuple argument. `pool.map` passes one item from the iterable to the function. Functions used with Pool must be module-level (picklable) and accept a single argument, or you must use `pool.starmap` for multiple arguments.
- **Lines 5-6:** Unpack the image specification and compute pixel count.
- **Lines 7-9:** The CPU simulation: take the square root of `pixelCount / 100` numbers. For a 1920×1080 image that is 20,736 square root operations — genuine CPU computation that benefits from parallelism.
- **Lines 10-12:** Compute resized dimensions and return a result tuple. The return value travels back through IPC, so keep it small.
- **Lines 15-17:** Sequential baseline: process all 8 images in the main process, one at a time.
- **Line 20:** `with multiprocessing.Pool(processes=4) as pool:` — creates 4 worker processes and ensures clean shutdown (calling `pool.terminate()` and `pool.join()`) when the `with` block exits, even if an exception occurs.
- **Line 21:** `pool.map(simulateImageResize, images)` — distributes 8 images across 4 workers. Each worker gets 2 images. `map` returns results in submission order (image 0 first, then 1, then 2...) regardless of completion order. This blocks until all 8 are done.
- **Lines 22-24:** Timing the pool run and computing speedup. On a 4-core machine with no other load, expect close to 4× speedup. Real speedup is less due to process startup amortization and IPC overhead for the result tuples.
- **Line 25:** Verify correctness — the pool result must equal the sequential result (same images, same algorithm).
