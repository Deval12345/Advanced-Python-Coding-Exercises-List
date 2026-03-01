# Code — Lecture 26: Multiprocessing Part 2 — IPC Cost, Shared Memory, and When Parallelism Fails

---

## Example 26.1 — Measuring IPC cost vs computation cost

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

**Line-by-line explanation:**

- **Lines 4-6:** `lightWorkSmallData` — multiplication (microseconds of compute) with a tiny IPC payload (two integers). IPC overhead dominates — expect no speedup in pool.
- **Lines 7-10:** `heavyWorkSmallData` — 200,000 square root operations (~10-50ms of compute) with the same tiny payload. Compute dominates — expect clear speedup in pool.
- **Lines 11-13:** `lightWorkLargeData` — sum of a list (microseconds) but the list itself (50,000 integers ≈ 400KB) must cross the process boundary. Serialization of the list dominates.
- **Lines 14-19:** `heavyWorkLargeData` — square root on each element (significant compute) but same large IPC cost. A borderline case — speedup depends on the ratio of compute time to serialization time.
- **Line 21:** `smallTasks` — each task carries just two integers. These are the "correct" style of IPC message: parameterization, not payload.
- **Line 22:** `largeTasks` — each task carries a list of 50,000 integers. This is the antipattern: sending data that the worker could instead compute or read from shared memory.
- **Lines 30-31:** Sequential baseline: no IPC, just pure function call overhead.
- **Lines 33-34:** Pool measurement: includes IPC round-trip for each task. `pool.map` is warm (pool already running from line 23), so no startup cost.
- **Lines 36-38:** The speedup ratio. A ratio < 1.2× means the parallelism benefit is masked by overhead.

**Expected output pattern:**
- Heavy compute, small data: speedup ≈ 3–4× (parallel wins clearly)
- Light compute, small data: speedup < 1× (IPC overhead dominates)
- Heavy compute, large data: speedup 1–2× (marginal; IPC partially cancels parallelism)
- Light compute, large data: speedup < 1× (IPC cost far exceeds any compute benefit)

---

## Example 26.2 — ProcessPoolExecutor with submit and as_completed

```python
# Example 26.2
import concurrent.futures                              # line 1
import time                                            # line 2
import math                                            # line 3

def computeHash(rangeSize):                            # line 4
    if rangeSize < 0:                                  # line 5
        raise ValueError(f"Invalid range: {rangeSize}")  # line 6
    total = sum(                                       # line 7
        int(math.sqrt(i) * 1000) % 9973               # line 8  deterministic hash-like value
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

**Line-by-line explanation:**

- **Line 4-11:** `computeHash` is a module-level function (required for pickling). It validates its input and raises `ValueError` for negative values — simulating a worker that can fail.
- **Line 13:** The workload list includes `-1` as a deliberately invalid input. This tests exception propagation through the Future.
- **Line 14:** `ProcessPoolExecutor(max_workers=4)` — identical syntax to `ThreadPoolExecutor`. The workers are OS processes, but the interface is unchanged.
- **Line 15:** Dictionary comprehension mapping each submitted Future to its input size. This `{future: input}` pattern is the standard way to correlate futures with their inputs in `as_completed` loops.
- **Lines 18-26:** `as_completed(futures)` yields futures as they finish — fastest first. The `rangeSize = futures[future]` lookup recovers which input this future corresponds to. `future.result()` either returns the value or re-raises any exception that occurred inside the worker process.
- **Lines 24-25:** Catching `ValueError` from the worker. The exception was raised in a subprocess, serialized, transmitted back, and re-raised here by `future.result()`. The calling code sees it as if it were raised locally — exactly the same behavior as with `ThreadPoolExecutor`.

---

## Example 26.3 — Profiling the crossover point between sequential and parallel

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
            [simulateWork(t) for t in tasks]           # line 17  sequential baseline
            seqTime = time.perf_counter() - start      # line 18

            start = time.perf_counter()                # line 19
            pool.map(simulateWork, tasks)              # line 20  parallel
            poolTime = time.perf_counter() - start     # line 21

            speedup = seqTime / poolTime               # line 22
            verdict = "PARALLEL WINS" if speedup > 1.2 else "SEQUENTIAL WINS"  # line 23
            print(f"{durationMs:<15} {seqTime:<10.3f} {poolTime:<10.3f} {speedup:<10.2f} {verdict}")  # line 24
```

**Line-by-line explanation:**

- **Lines 4-8:** `simulateWork` calibrates iterations to approximate a target computation time. `computeMs * 50_000` iterations of `math.sqrt` produces roughly that many milliseconds on typical hardware. The actual duration varies by machine.
- **Line 11:** `durationsMsToTest` spans three orders of magnitude — from 1ms (definitely too short for multiprocessing) to 500ms (definitely long enough to benefit).
- **Line 12:** Pool is created once and reused across all benchmark runs. This eliminates process startup cost from all measurements — we are measuring pure IPC overhead, not startup.
- **Lines 16-18:** Sequential run: just function call overhead, no pickling.
- **Lines 19-21:** Pool run: includes task pickling, IPC submission, parallel execution, result pickling, and IPC return.
- **Line 22-23:** Speedup ratio and verdict. The crossover — where verdict switches from "SEQUENTIAL WINS" to "PARALLEL WINS" — is the key output of this benchmark.

**Expected output on a 4-core machine:**
```
Duration (ms)   Seq (s)    Pool (s)   Speedup    Verdict
1               0.008      0.045      0.18x      SEQUENTIAL WINS
5               0.040      0.048      0.83x      SEQUENTIAL WINS
10              0.080      0.060      1.33x      PARALLEL WINS
50              0.400      0.130      3.08x      PARALLEL WINS
100             0.800      0.230      3.48x      PARALLEL WINS
500             4.000      1.050      3.81x      PARALLEL WINS
```
The crossover appears between 5ms and 10ms on this configuration. Your exact numbers will differ by machine, but the pattern is universal.
