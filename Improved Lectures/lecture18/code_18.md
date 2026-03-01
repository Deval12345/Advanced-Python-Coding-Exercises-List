# Lecture 18 — Concurrency Models Overview: I/O-Bound vs CPU-Bound Work
## Code Examples

---

## Example 18.1 — Sequential I/O Simulation

This example establishes the baseline problem. Five simulated API calls run one after another. Each call blocks for one second, and the next call cannot begin until the previous one completes. The CPU is idle for the entire duration of each sleep. This is the I/O-bound problem in its clearest form.

```python
import time                                          # 1: standard library time module

def simulateFetch(taskId):                           # 2: simulates one API call or I/O operation
    time.sleep(1)                                    # 3: sleep(1) represents the I/O wait — CPU is idle here
    return f"result_{taskId}"                        # 4: return a dummy result string

def runSequential(count):                            # 5: runs all fetches sequentially, one at a time
    start = time.perf_counter()                      # 6: perf_counter gives high-resolution wall-clock time
    results = [simulateFetch(i) for i in range(count)]  # 7: list comprehension calls each fetch in order
    elapsed = time.perf_counter() - start            # 8: subtract start time to get total elapsed seconds
    print(f"Sequential: {len(results)} tasks in {elapsed:.2f}s")  # 9: report task count and wall time
    return results                                   # 10: return list of result strings

runSequential(5)                                     # 11: invoke with 5 tasks — expect output: ~5.00s
```

**Line-by-line explanation:**

- **Line 1**: Imports the `time` module. We use `time.sleep` to simulate I/O latency and `time.perf_counter` for high-resolution timing.
- **Line 2**: Defines `simulateFetch`, which represents one unit of I/O work — an API call, a database query, or a disk read. The `taskId` parameter identifies which task this is.
- **Line 3**: `time.sleep(1)` is the critical line. This simulates a one-second I/O wait. During this sleep, the CPU is idle — it has nothing to compute. This is exactly what makes a program I/O-bound.
- **Line 4**: Returns a placeholder result. In real code this would be the response body, the database row, or the file contents.
- **Line 5**: Defines `runSequential`, which runs `count` fetches sequentially.
- **Line 6**: `time.perf_counter()` returns the current value of a high-resolution performance counter in seconds. It is the correct timer for measuring elapsed wall time in Python — more precise than `time.time()`.
- **Line 7**: The list comprehension calls `simulateFetch(i)` for each value of `i` from 0 to `count - 1`. Because this is a sequential list comprehension, each call blocks until the previous one returns. Total time = count x 1 second.
- **Line 8**: Subtracts the start time from the current time to get elapsed seconds.
- **Line 9**: Prints the result. For 5 tasks you will see approximately `Sequential: 5 tasks in 5.00s`.
- **Line 10**: Returns the collected results.
- **Line 11**: Invokes the function. Expected output: `Sequential: 5 tasks in 5.00s`. This is the problem we solve in Example 18.2.

**Expected output:**
```
Sequential: 5 tasks in 5.00s
```

---

## Example 18.2 — Threaded I/O Simulation

This example solves the problem from Example 18.1 using threads. All five tasks are started simultaneously. Each thread sleeps for one second independently. When they all sleep at the same time, the total wall time drops from five seconds to approximately one second. A lock protects the shared results list from concurrent modification.

```python
import time                                          # 1: standard library time module
import threading                                     # 2: standard library threading module

results = []                                         # 3: module-level shared list — all threads write here
lock = threading.Lock()                              # 4: Lock object — only one thread may hold it at a time

def fetchWithThread(taskId):                         # 5: the function each thread will execute
    time.sleep(1)                                    # 6: simulate I/O wait — GIL is released during sleep
    with lock:                                       # 7: acquire the lock before writing to shared state
        results.append(f"result_{taskId}")           # 8: safely append to shared list while lock is held

def runThreaded(count):                              # 9: creates and runs count threads concurrently
    global results                                   # 10: declare that we are modifying the module-level list
    results = []                                     # 11: reset the list for a fresh run
    start = time.perf_counter()                      # 12: high-resolution start timestamp
    threads = [                                      # 13: create a Thread object for each task
        threading.Thread(target=fetchWithThread, args=(i,))
        for i in range(count)
    ]
    for t in threads:                                # 14: start all threads — they all begin running now
        t.start()
    for t in threads:                                # 15: join — wait for every thread to finish
        t.join()
    elapsed = time.perf_counter() - start            # 16: compute total elapsed wall time
    print(f"Threaded: {len(results)} tasks in {elapsed:.2f}s")  # 17: report result
    return results                                   # 18: return the collected results

runThreaded(5)                                       # 19: invoke with 5 tasks — expect output: ~1.00s
```

**Line-by-line explanation:**

- **Line 1-2**: Import `time` and `threading`. The threading module is part of Python's standard library and requires no installation.
- **Line 3**: `results` is a module-level list shared by all threads. Multiple threads will write to this list, so it must be protected.
- **Line 4**: `threading.Lock()` creates a mutex. Only one thread can hold the lock at a time. Any thread that calls `lock.acquire()` while another thread holds the lock will block until the lock is released. The `with` statement handles acquire and release automatically.
- **Line 5**: `fetchWithThread` is the function each thread will run. This is called the thread's target function.
- **Line 6**: `time.sleep(1)` simulates one second of I/O waiting. Critically, CPython releases the GIL during `time.sleep` because it is a system call, not Python bytecode execution. This means all five threads can sleep simultaneously — they do not take turns.
- **Line 7**: `with lock:` acquires the lock. If another thread is currently inside this block, the current thread waits. This ensures that only one thread modifies `results` at a time, preventing race conditions.
- **Line 8**: Appends the result to the shared list while the lock is held. The lock is released automatically when the `with` block exits.
- **Line 9**: `runThreaded` orchestrates the entire threaded execution.
- **Line 10**: `global results` tells Python that the assignment on line 11 refers to the module-level variable, not a new local variable.
- **Line 11**: Resets the results list before each run so that repeated calls to `runThreaded` start fresh.
- **Line 12**: Records the start time.
- **Line 13**: Creates one `Thread` object per task. `target=fetchWithThread` specifies the function to run. `args=(i,)` passes the task ID as a positional argument. Note the comma — `(i,)` creates a tuple, which is required by the `args` parameter.
- **Line 14**: Calling `t.start()` on each thread launches it. After this loop, all five threads are running simultaneously. Each is sleeping for one second independently.
- **Line 15**: `t.join()` blocks until the thread finishes. The outer loop waits for all five threads. Because they all run simultaneously, the join loop finishes approximately one second after the start loop, not five seconds.
- **Line 16**: Computes elapsed time.
- **Line 17**: Prints the result. Expected: `Threaded: 5 tasks in 1.00s`.
- **Line 18**: Returns the results.
- **Line 19**: Invokes the function. The 5x speedup compared to sequential execution demonstrates why threads are the correct model for I/O-bound work.

**Expected output:**
```
Threaded: 5 tasks in 1.00s
```

**Key comparison:** Sequential took ~5.00s. Threaded takes ~1.00s. Same work, same I/O cost, 5x faster because the waiting periods overlap instead of stacking.

---

## Example 18.3 — CPU-Bound Work: Threads Do Not Help

This example demonstrates that the threading speedup from Example 18.2 does not apply to CPU-bound work. The GIL prevents multiple threads from executing Python bytecode simultaneously. A CPU-bound workload run sequentially and run with threads takes approximately the same amount of time — because only one thread can actually run at any moment.

```python
import time                                          # 1: standard library time module
import threading                                     # 2: standard library threading module

def computeHeavy(n):                                 # 3: CPU-bound function — pure computation, no I/O
    total = 0                                        # 4: initialize accumulator
    for i in range(n):                               # 5: tight loop — Python bytecode, GIL never released
        total += i * i                               # 6: arithmetic in every iteration
    return total                                     # 7: return final sum

def runCpuSequential():                              # 8: runs four computations sequentially
    start = time.perf_counter()                      # 9: start timestamp
    results = [computeHeavy(2_000_000) for _ in range(4)]  # 10: four sequential calls
    elapsed = time.perf_counter() - start            # 11: compute elapsed time
    print(f"CPU sequential: {elapsed:.2f}s")         # 12: report time

def runCpuThreaded():                                # 13: runs four computations in threads
    start = time.perf_counter()                      # 14: start timestamp
    threads = [                                      # 15: create four Thread objects
        threading.Thread(target=computeHeavy, args=(2_000_000,))
        for _ in range(4)
    ]
    for t in threads:                                # 16: start all four threads
        t.start()
    for t in threads:                                # 17: wait for all four threads to finish
        t.join()
    elapsed = time.perf_counter() - start            # 18: compute elapsed time
    print(f"CPU threaded (GIL blocks): {elapsed:.2f}s")  # 19: report time

runCpuSequential()                                   # 20: baseline — pure sequential CPU work
runCpuThreaded()                                     # 21: threaded — expect similar or worse time
# Both will take roughly the same time — threads do not help CPU-bound work in Python
```

**Line-by-line explanation:**

- **Line 1-2**: Import `time` and `threading` as before.
- **Line 3**: `computeHeavy` is a purely CPU-bound function. It performs no I/O, no file access, no network calls — only arithmetic. The GIL will never be released during its execution.
- **Line 4**: Initializes a running total.
- **Line 5**: Iterates `n` times. This is a tight Python loop — every iteration executes Python bytecode, so the GIL is held continuously.
- **Line 6**: Multiplies `i` by itself and adds to the total. Pure CPU arithmetic.
- **Line 7**: Returns the result. (Note: in `runCpuThreaded` this return value is not collected — the point of the example is timing, not results.)
- **Line 8**: `runCpuSequential` runs four computations one at a time.
- **Line 9**: Records start time.
- **Line 10**: List comprehension calls `computeHeavy(2_000_000)` four times sequentially. The `_` variable name signals that we do not use the loop variable. `2_000_000` uses Python's underscore separator for readability — it is the integer two million.
- **Line 11**: Computes elapsed time.
- **Line 12**: Prints the sequential time baseline.
- **Line 13**: `runCpuThreaded` attempts to parallelize the same work with threads.
- **Line 14**: Records start time.
- **Line 15**: Creates four Thread objects, each targeting `computeHeavy` with two million iterations.
- **Line 16**: Starts all four threads. They are now "running" — but the GIL ensures only one can actually execute Python bytecode at any moment.
- **Line 17**: Joins all four threads — waits for each to complete.
- **Line 18**: Computes elapsed time.
- **Line 19**: Prints the threaded time. You will observe that it is approximately equal to — or sometimes slightly slower than — the sequential time.
- **Line 20**: Calls the sequential baseline.
- **Line 21**: Calls the threaded version. The GIL prevents any parallel speedup.

**Expected output (approximate, varies by machine):**
```
CPU sequential: 1.85s
CPU threaded (GIL blocks): 1.92s
```

**Why are they the same?** Because `computeHeavy` never releases the GIL. Although four threads exist, only one runs at a time. The threads effectively take turns on the interpreter. There is no parallelism. The thread overhead may even make the threaded version slightly slower.

**The diagnostic conclusion:** If you see no speedup from adding threads to a Python program, the program is CPU-bound. The correct fix is multiprocessing — separate processes with separate GILs — which will be covered in a later lecture. Adding more threads to a CPU-bound Python program will never produce a speedup. This is the GIL's defining limitation.

**The contrast with Example 18.2:** In Example 18.2, `time.sleep(1)` releases the GIL during the system call, allowing all threads to sleep simultaneously — producing a 5x speedup. In this example, the tight computation loop never releases the GIL, so threads provide no benefit. The difference between these two examples is the entire reason the I/O-bound vs CPU-bound diagnosis exists.
