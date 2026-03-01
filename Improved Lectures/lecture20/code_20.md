# Lecture 20: The Global Interpreter Lock
## Code Examples with Line-by-Line Explanations

---

## EXAMPLE 2.1 — sys.getrefcount Demonstration

**Purpose:** Make Python's reference counting mechanism visible. Demonstrates that Python tracks every reference to every object, and that the GIL protects these counts from concurrent corruption.

```python
# Example 20.1
import sys                                             # line 1

data = [1, 2, 3]                                      # line 2
print(sys.getrefcount(data))                          # line 3  → 2 (data + getrefcount arg)

alias = data                                          # line 4
print(sys.getrefcount(data))                          # line 5  → 3

container = [data, data]                              # line 6
print(sys.getrefcount(data))                          # line 7  → 5 (data + alias + 2 in container + getrefcount arg)

del alias                                             # line 8
print(sys.getrefcount(data))                          # line 9  → 4

del container                                         # line 10
print(sys.getrefcount(data))                          # line 11 → 2
```

### Line-by-Line Explanation

**Line 1 — `import sys`**
Imports the `sys` module, which exposes CPython interpreter internals. The `sys.getrefcount` function is specific to CPython's implementation of Python — it returns the current value of an object's `ob_refcnt` field, the reference count maintained by the GIL-protected memory manager. This function is not available in PyPy or other Python implementations that do not use reference counting.

**Line 2 — `data = [1, 2, 3]`**
Creates a new list object in heap memory and binds the name `data` in the current scope to it. At this point, exactly one name in the program refers to this list object. The list's internal `ob_refcnt` is 1.

**Line 3 — `print(sys.getrefcount(data))`**
Calls `sys.getrefcount`, passing `data` as the argument. This creates a temporary second reference — the function parameter — for the duration of the call. So `getrefcount` sees a count of 2: one for `data` (the module-level name) and one for the function argument being passed. After the function returns, the temporary argument reference is dropped and the count returns to 1. Output: `2`.

This "+1 from getrefcount itself" is the most common source of confusion with this function. Always subtract 1 from any result to get the true number of external references.

**Line 4 — `alias = data`**
Creates a second name, `alias`, that points to the same list object. No new list is created — both `data` and `alias` are two names for one object. The list's reference count is now 2.

**Line 5 — `print(sys.getrefcount(data))`**
Now `getrefcount` sees 3: `data` (1), `alias` (1), and the function argument (1). Output: `3`.

**Line 6 — `container = [data, data]`**
Creates a new list whose first and second elements are both the same list object. Each element slot in a Python list holds a reference. So `container[0]` is one reference and `container[1]` is another. The `data` list object now has 4 references: `data` name, `alias` name, `container[0]`, `container[1]`.

**Line 7 — `print(sys.getrefcount(data))`**
`getrefcount` sees 5: `data` (1), `alias` (1), `container[0]` (1), `container[1]` (1), and the function argument (1). Output: `5`.

This demonstrates a critical point: a single list element that appears once in a container counts as one reference. Both slots in `container` each hold a reference, so containing the same object twice adds two references, not one.

**Line 8 — `del alias`**
Removes the name `alias` from the current scope. This decrements the reference count by 1 (from 4 to 3 external references). The object is not freed — three other references still exist. The `del` statement removes the binding, not the object.

**Line 9 — `print(sys.getrefcount(data))`**
`getrefcount` sees 4: `data` (1), `container[0]` (1), `container[1]` (1), function argument (1). Output: `4`.

**Line 10 — `del container`**
Removes the name `container` from scope. This does not immediately destroy the contained list — but it decrements the reference count of the `container` list object. When `container`'s own reference count reaches zero, `container` itself is freed. During that freeing process, Python decrements the reference count of every object `container` held — both slots pointed to `data`, so `data`'s reference count is decremented twice. Count drops from 3 to 1.

**Line 11 — `print(sys.getrefcount(data))`**
Only the name `data` and the function argument remain. Output: `2`.

### Key Insight from This Example

If we were to now execute `del data`, the reference count would drop to 0 (accounting for the getrefcount argument disappearing when the del completes), and the list `[1, 2, 3]` would be immediately freed. This is Python's reference-counting memory management in action — deterministic, immediate, and protected by the GIL from concurrent corruption.

---

## EXAMPLE 3.1 — Threads vs Processes on CPU-Bound Work

**Purpose:** Empirically demonstrate the GIL's impact on CPU-bound parallelism. Shows that threading provides no speedup (and may cause slowdown) for pure Python computation, while multiprocessing achieves near-linear speedup by using separate interpreters with separate GILs.

```python
# Example 20.2
import time                                           # line 1
import threading                                      # line 2
import multiprocessing                                # line 3

def cpuHeavyTask():                                   # line 4
    total = 0                                         # line 5
    for i in range(40_000_000):                       # line 6
        total += i                                    # line 7
    return total                                      # line 8

def runWithThreads(numThreads):                       # line 9
    threads = [threading.Thread(target=cpuHeavyTask) for _ in range(numThreads)]  # line 10
    start = time.perf_counter()                       # line 11
    for t in threads: t.start()                       # line 12
    for t in threads: t.join()                        # line 13
    return time.perf_counter() - start                # line 14

def runWithProcesses(numProcesses):                   # line 15
    procs = [multiprocessing.Process(target=cpuHeavyTask) for _ in range(numProcesses)]  # line 16
    start = time.perf_counter()                       # line 17
    for p in procs: p.start()                         # line 18
    for p in procs: p.join()                          # line 19
    return time.perf_counter() - start                # line 20

if __name__ == "__main__":                            # line 21
    seqStart = time.perf_counter()                    # line 22
    cpuHeavyTask()                                    # line 23
    seqTime = time.perf_counter() - seqStart          # line 24
    threadTime = runWithThreads(4)                    # line 25
    processTime = runWithProcesses(4)                 # line 26
    print(f"Sequential:    {seqTime:.2f}s")           # line 27
    print(f"4 Threads:     {threadTime:.2f}s (GIL bottleneck)")   # line 28
    print(f"4 Processes:   {processTime:.2f}s (true parallelism)")  # line 29
```

### Line-by-Line Explanation

**Line 1 — `import time`**
Imports the `time` module. We specifically need `time.perf_counter()`, which returns a high-resolution floating-point timestamp in seconds. It is the correct tool for benchmarking — it is not affected by system clock adjustments and has sub-microsecond resolution on modern hardware. `time.time()` is suitable for wall-clock time but less precise for short benchmarks.

**Line 2 — `import threading`**
Imports the `threading` module, which provides `threading.Thread` — the wrapper around OS-level threads. CPython threads are genuine operating system threads (pthreads on Unix, Win32 threads on Windows), not green threads or coroutines. The GIL is a CPython-level constraint on top of real OS threads.

**Line 3 — `import multiprocessing`**
Imports the `multiprocessing` module, which provides `multiprocessing.Process` — spawns a separate OS process, each running a fresh CPython interpreter. Because each process is a separate interpreter, each has its own GIL, and they can run truly in parallel on separate CPU cores.

**Lines 4–8 — `def cpuHeavyTask()`**
Defines the workload. This function does 40 million integer additions in a pure Python `for` loop.

- Line 5: `total = 0` — initializes a local integer variable
- Line 6: `for i in range(40_000_000)` — iterates 40 million times; `range()` is a lazy iterator, not a list, so this does not allocate 40 million integers upfront
- Line 7: `total += i` — adds the loop counter to the running total; each iteration executes several Python bytecode instructions (LOAD_FAST, BINARY_ADD, STORE_FAST), all of which require the GIL
- Line 8: `return total` — returns the accumulated sum

Critically, this function never calls `socket`, `file.read`, `time.sleep`, or any other I/O system call. It is pure computation inside the Python interpreter. The GIL is held for the entire duration of each thread's execution of this function — it is never released.

**Lines 9–14 — `def runWithThreads(numThreads)`**

- Line 10: Creates a list of `threading.Thread` objects, one per requested thread count. The list comprehension passes `cpuHeavyTask` as the `target` — the function each thread will call when started. At this point, no threads are running yet; they are configured but not started.
- Line 11: Records the start time using `perf_counter`. The timer starts before any thread is launched so we capture thread creation overhead as well.
- Line 12: `for t in threads: t.start()` — launches all threads. Each `t.start()` tells the OS to begin executing the thread. All threads are started before any `join()` call, meaning they all launch and begin competing for the GIL immediately.
- Line 13: `for t in threads: t.join()` — blocks the main thread until each worker thread finishes. The `join()` loop must come after all `start()` calls — if we called `start()` then immediately `join()` in the same loop, we would serialize execution (start thread 1, wait for thread 1, start thread 2, etc.).
- Line 14: Returns elapsed time. On a typical machine, this will be approximately equal to or slightly greater than the sequential time, because the GIL forces the four threads to take turns running their Python bytecode.

**Lines 15–20 — `def runWithProcesses(numProcesses)`**
Mirrors the threading implementation exactly, but with `multiprocessing.Process` instead of `threading.Thread`.

- Line 16: Creates `multiprocessing.Process` objects. On Unix/macOS, the default start method is `fork` — the child process gets a copy of the parent's memory space, including the function `cpuHeavyTask`. On Windows, the default is `spawn` — the child interpreter is started fresh and imports the module.
- Lines 17–19: The same start-then-join pattern. The difference: each process has a completely independent CPython interpreter and a completely independent GIL. Four processes can run `cpuHeavyTask` simultaneously on four cores with no GIL contention whatsoever.
- Line 20: Returns elapsed time. On a four-core machine, this will be approximately equal to `seqTime / 4` — near-linear speedup.

**Line 21 — `if __name__ == "__main__":`**
An essential guard for multiprocessing on all platforms. On Windows (and macOS with the `spawn` start method), when a child process is created, Python re-imports the main module to get the function definition. Without this guard, the import would re-execute the benchmark code in the child process, causing infinite process spawning. The guard ensures the benchmark code only runs in the original parent process.

**Lines 22–24 — Sequential benchmark**
Runs `cpuHeavyTask` once in the main thread to establish the baseline. This is the wall-clock time for a single-threaded execution. All subsequent times are compared against this.

**Line 25 — `threadTime = runWithThreads(4)`**
Runs the benchmark with four threads. Expected result: approximately equal to `seqTime` (the GIL serializes execution), or slightly worse (context-switch overhead with no parallelism benefit). In practice, many users observe 1.0x–1.1x of sequential time with four "parallel" threads.

**Line 26 — `processTime = runWithProcesses(4)`**
Runs the benchmark with four processes. Expected result: approximately `seqTime / 4` on a four-core machine. On a two-core machine, expect approximately `seqTime / 2`. The speedup is real and measurable — true parallelism.

**Lines 27–29 — Output**
Prints the three times with descriptive labels. The comments in the format string ("GIL bottleneck" and "true parallelism") make the benchmark self-documenting — the output itself teaches the lesson.

### Expected Output (approximate, on a modern 4-core machine)
```
Sequential:    2.80s
4 Threads:     2.95s (GIL bottleneck)
4 Processes:   0.75s (true parallelism)
```

### Key Insight from This Example

The thread version is not just "not faster" — it can be marginally slower. The OS is doing real work creating four threads, context-switching between them, and saving/restoring their state. All of that overhead is paid in full. The GIL then ensures that this overhead produces zero additional throughput. The process version pays its own overhead (process spawn, memory copy/setup) but then achieves genuine parallelism on separate cores, easily overcoming the startup cost for workloads of this duration.

For workloads shorter than ~100ms per task, process spawn overhead may dominate and `ProcessPoolExecutor` with pre-created workers is preferred over raw `multiprocessing.Process`.
