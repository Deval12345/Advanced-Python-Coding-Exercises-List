# Speech Source — Lecture 26: Multiprocessing Part 2 — IPC Cost, Shared Memory, and When Parallelism Fails

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 25 we established the foundation: multiprocessing creates separate OS processes each with its own Python interpreter and GIL, giving true CPU parallelism. We covered multiprocessing.Process, multiprocessing.Queue, and multiprocessing.Pool. We introduced the golden rule — minimize what crosses process boundaries — but we were deliberately brief on the mechanics of why. Today we go deep on exactly that: what does it cost to move data between processes, when does that cost dominate the computation benefit, and what alternatives exist when raw pickling is too expensive. We also look at two high-level alternatives that make the Pool API feel even more Pythonic: concurrent.futures.ProcessPoolExecutor and the executor.map pattern you already know from the threading world.

---

## CONCEPT 1.1 — The True Cost of IPC: Three Data Copies

**Problem it solves:**
Most developers who are new to multiprocessing understand that IPC exists, but they do not have a concrete mental model of what it costs. They think of it as "a small overhead" and move on. The reality is more specific and more dangerous. When you send a Python object from one process to another through a Queue or a Pipe, the following sequence happens: pickle serializes the object into a bytes buffer in the sending process — that is copy one. The bytes are written through a kernel pipe — that is copy two. The receiving process reads the bytes and unpickle reconstructs the object in its own memory — that is copy three. A 100-megabyte NumPy array that crosses a process boundary has been copied three times, consuming 300 megabytes of peak memory in transit.

**Why invented:**
The isolation that makes processes safe — separate memory spaces — is precisely what necessitates this overhead. There is no free lunch. Shared memory mechanisms (multiprocessing.Value, multiprocessing.Array, mmap) exist specifically to provide a path where data does not need to be pickled at all — two or more processes can read and write the same physical memory page. But these require explicit lock discipline to remain safe.

**What happens without it:**
A video analytics pipeline designed to send raw 4K frames through a Queue processes 50-megabyte frames at 30 frames per second. That is 1.5 gigabytes per second of data transfer, all through serialization. The system becomes memory-bound before it becomes CPU-bound. Parallelism slows the pipeline compared to sequential processing because the workers spend more time deserializing frames than analyzing them.

**Industry impact:**
Understanding this cost is what separates a multiprocessing design that scales from one that regresses. The rule is architectural: design so that inter-process messages are tiny and computation is large. Send file paths, not file contents. Send array indices, not arrays. Send configuration parameters, not datasets. Do heavy data manipulation inside the worker in its own memory.

---

## EXAMPLE 1.1 — Measuring IPC cost vs computation cost

Narration: This benchmark isolates two variables: data payload size and computation work. We run four scenarios: small data with small computation, small data with large computation, large data with small computation, and large data with large computation. For each, we time sequential execution versus Pool with four workers. The result reveals the crossover point — where parallelism helps and where it hurts. Small data and large computation: processes win. Large data and small computation: sequential wins. The IPC cost is the x-axis of this decision; computation benefit is the y-axis.

```python
# Example 26.1
import multiprocessing                                 # line 1
import time                                            # line 2
import math                                            # line 3

def lightWorkSmallData(params):                        # line 4
    taskId, value = params                             # line 5
    return value * 2                                   # line 6  tiny compute, tiny data

def heavyWorkSmallData(params):                        # line 7
    taskId, value = params                             # line 8
    total = sum(math.sqrt(i) for i in range(value))   # line 9  heavy compute, tiny data
    return (taskId, total)                             # line 10

def lightWorkLargeData(params):                        # line 11
    taskId, largeList = params                         # line 12
    return sum(largeList)                              # line 13  tiny compute, huge IPC cost

def heavyWorkLargeData(params):                        # line 14
    taskId, largeList = params                         # line 15
    total = 0                                          # line 16
    for x in largeList:                                # line 17
        total += math.sqrt(x)                          # line 18  heavy compute, huge IPC cost
    return (taskId, total)                             # line 19

if __name__ == "__main__":                             # line 20
    smallTasks = [(i, 200_000) for i in range(8)]      # line 21
    largeTasks = [(i, list(range(50_000))) for i in range(8)]  # line 22

    with multiprocessing.Pool(processes=4) as pool:    # line 23
        for label, fn, tasks in [                      # line 24
            ("Heavy compute, small data", heavyWorkSmallData, smallTasks),  # line 25
            ("Light compute, small data", lightWorkSmallData, smallTasks),  # line 26
            ("Heavy compute, large data", heavyWorkLargeData, largeTasks),  # line 27
            ("Light compute, large data", lightWorkLargeData, largeTasks),  # line 28
        ]:                                             # line 29
            start = time.perf_counter()                # line 30
            seqResults = [fn(t) for t in tasks]        # line 31
            seqTime = time.perf_counter() - start      # line 32

            start = time.perf_counter()                # line 33
            poolResults = pool.map(fn, tasks)          # line 34
            poolTime = time.perf_counter() - start     # line 35

            ratio = seqTime / poolTime                 # line 36
            verdict = "FASTER" if ratio > 1.2 else "SLOWER or no gain"  # line 37
            print(f"{label}: seq={seqTime:.3f}s pool={poolTime:.3f}s speedup={ratio:.2f}x [{verdict}]")  # line 38
```

---

## CONCEPT 2.1 — concurrent.futures.ProcessPoolExecutor as the High-Level Alternative

**Problem it solves:**
multiprocessing.Pool is excellent but its API differs from the threading executor API — pool.map versus executor.map, pool.apply_async versus executor.submit, AsyncResult.get() versus future.result(). For teams that already use concurrent.futures for threading, this inconsistency creates cognitive friction. More importantly, multiprocessing.Pool does not return Future objects — it returns AsyncResult objects that do not compose well with the broader futures ecosystem.

**Why invented:**
concurrent.futures.ProcessPoolExecutor was designed to provide the exact same interface as ThreadPoolExecutor but backed by processes rather than threads. The same executor.submit, the same executor.map, the same Future objects, the same as_completed support. Switching from thread-based to process-based parallelism is a one-line change. The futures compose naturally with asyncio via asyncio.wrap_future, enabling the hybrid architectures we discussed in Lecture 23.

**What happens without it:**
ProcessPoolExecutor provides one capability that multiprocessing.Pool lacks: direct integration with the async ecosystem. pool.map is blocking — you cannot await it from inside a coroutine. executor.map returns an iterator of futures that can be wrapped and awaited. This is the correct choice when the process pool is part of a larger async system.

**Industry impact:**
ProcessPoolExecutor is the preferred choice when your process-based parallelism needs to compose with other concurrent.futures code or with asyncio. Use multiprocessing.Pool when you need more control: daemon processes, custom worker initializers, maxtasksperchild for memory leak mitigation, or pool.imap for true streaming results.

---

## EXAMPLE 2.1 — ProcessPoolExecutor with submit and as_completed

Narration: We submit eight computational tasks to a ProcessPoolExecutor with four workers. Each task computes a hash-like value for a range of integers. We use executor.submit to get Future objects back immediately, then iterate with as_completed to process results in the order they finish. Tasks with smaller ranges finish first and their results appear in the output before the longer tasks complete. We also demonstrate error handling: one task is deliberately given an invalid parameter and raises a ValueError. as_completed surfaces the exception through future.result(), which we catch with a normal try-except block.

```python
# Example 26.2
import concurrent.futures                              # line 1
import time                                            # line 2
import math                                            # line 3

def computeHash(rangeSize):                            # line 4
    if rangeSize < 0:                                  # line 5
        raise ValueError(f"Invalid range: {rangeSize}")  # line 6
    total = sum(                                       # line 7
        int(math.sqrt(i) * 1000) % 9973               # line 8  deterministic hash
        for i in range(rangeSize)                      # line 9
    )                                                  # line 10
    return (rangeSize, total)                          # line 11

if __name__ == "__main__":                             # line 12
    workloads = [50000, 80000, 30000, -1, 60000, 40000, 90000, 20000]  # line 13

    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:  # line 14
        futures = {executor.submit(computeHash, size): size for size in workloads}  # line 15

        successCount = 0                               # line 16
        errorCount = 0                                 # line 17

        for future in concurrent.futures.as_completed(futures):  # line 18
            rangeSize = futures[future]                # line 19
            try:                                       # line 20
                result = future.result()               # line 21
                print(f"Range {rangeSize}: hash={result[1]}")  # line 22
                successCount += 1                      # line 23
            except ValueError as e:                    # line 24
                print(f"Error for range {rangeSize}: {e}")  # line 25
                errorCount += 1                        # line 26

    print(f"Done: {successCount} succeeded, {errorCount} failed")  # line 27
```

---

## CONCEPT 3.1 — When Multiprocessing Makes Things Worse

**Problem it solves:**
The most dangerous misconception in multiprocessing is the assumption that parallel always means faster. There are three concrete scenarios where multiprocessing is slower than sequential code, and recognizing them before deployment is critical.

**Scenario 1 — Task duration less than process overhead:**
Process startup costs 50-200ms. If your task runs in 10ms, even with a warm process pool (no startup cost), the IPC round-trip for submitting the task and collecting the result can cost 5-20ms per task for pickling and unpickling. Ten thousand tasks at 10ms each take 100 seconds sequentially. With a pool of 4 workers and 15ms IPC overhead per task, 10,000 IPC round-trips add 150 seconds of overhead — total: 25 seconds of computation plus 150 of IPC — slower than sequential.

**Scenario 2 — Large data payload dominates:**
When the task's input data is large, pickling it consumes most of the wall-clock time. A task that processes a 100MB array in 200ms and takes 2 seconds to pickle sees 90% of its time in serialization. Four workers in parallel each process their array in 200ms, but each also spends 2 seconds pickling, for a total of about 2.2 seconds — barely faster than the 2.4 seconds of sequential execution, and with 4× the memory pressure.

**Scenario 3 — Inter-worker communication:**
Embarrassingly parallel tasks — tasks that are completely independent with no communication — benefit most from multiprocessing. Tasks that require frequent communication between workers through shared queues or Manager objects pay synchronization costs that grow with worker count. A design where workers constantly coordinate can be slower than a single carefully sequenced process.

**Industry impact:**
Profile first. Always measure sequential vs. parallel before committing to a multiprocessing architecture. If you cannot measure a meaningful speedup in isolated benchmarks, the production system will not have one either.

---

## EXAMPLE 3.1 — Profiling the crossover point between sequential and parallel

Narration: We systematically vary task duration from 1 millisecond to 1000 milliseconds and measure speedup at each level. For tiny tasks, sequential wins. As task duration increases, the process pool's advantage grows. We identify the crossover point — the minimum task duration at which parallel execution is reliably faster. This is the data-driven way to make the build-vs-sequential decision.

```python
# Example 26.3
import multiprocessing                                 # line 1
import time                                            # line 2
import math                                            # line 3

def simulateWork(params):                              # line 4
    taskId, computeMs = params                         # line 5
    iterations = int(computeMs * 50_000)               # line 6  scale iterations to target ms
    total = sum(math.sqrt(i + 1) for i in range(iterations))  # line 7
    return (taskId, round(total, 4))                   # line 8

if __name__ == "__main__":                             # line 9
    numTasks = 8                                       # line 10
    durationsMsToTest = [1, 5, 10, 50, 100, 500]       # line 11

    with multiprocessing.Pool(processes=4) as pool:    # line 12
        print(f"{'Duration (ms)':<15} {'Seq (s)':<10} {'Pool (s)':<10} {'Speedup':<10} {'Verdict'}")  # line 13
        for durationMs in durationsMsToTest:           # line 14
            tasks = [(i, durationMs) for i in range(numTasks)]  # line 15

            start = time.perf_counter()                # line 16
            [simulateWork(t) for t in tasks]           # line 17  sequential
            seqTime = time.perf_counter() - start      # line 18

            start = time.perf_counter()                # line 19
            pool.map(simulateWork, tasks)              # line 20  parallel
            poolTime = time.perf_counter() - start     # line 21

            speedup = seqTime / poolTime               # line 22
            verdict = "PARALLEL WINS" if speedup > 1.2 else "SEQUENTIAL WINS"  # line 23
            print(f"{durationMs:<15} {seqTime:<10.3f} {poolTime:<10.3f} {speedup:<10.2f} {verdict}")  # line 24
```

---

## CONCEPT 4.1 — Final Takeaway Lecture 26

The true cost of IPC is three data copies — serialize, transfer, deserialize. For a 100MB NumPy array, that is 300MB of peak memory pressure and seconds of serialization time. The three scenarios where multiprocessing fails: tasks shorter than IPC overhead, data payload larger than computation benefit, and high inter-worker communication frequency. concurrent.futures.ProcessPoolExecutor gives the same high-level interface as ThreadPoolExecutor — Future objects, as_completed, callback support — backed by processes instead of threads. Use Pool when you need fine-grained control (daemon workers, maxtasksperchild); use ProcessPoolExecutor when your process parallelism must compose with futures or asyncio. The universal rule: profile before committing. Measure sequential versus parallel in isolation. If you cannot see a speedup in the benchmark, the production system will not have one either.

In the next lecture we look at sharing memory across processes without pickling — using multiprocessing.Array, mmap, and the specific design for sharing NumPy arrays through shared memory that enables zero-copy data access across processes.
