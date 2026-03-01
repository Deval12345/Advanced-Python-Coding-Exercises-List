# Lecture 23: Futures and Executors — The Bridge Between Sync and Async
## code_23.md — Complete Code Examples with Line-by-Line Explanations

---

## Example 23.1 — CPU-Bound Tasks with ProcessPoolExecutor

```python
# Example 23.1
import concurrent.futures                              # line 1
import time                                            # line 2
import math                                            # line 3

def computePrimeCount(limit):                          # line 4
    count = 0                                          # line 5
    for n in range(2, limit):                          # line 6
        if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1)):  # line 7
            count += 1                                 # line 8
    return count                                       # line 9

def runSequential(limits):                             # line 10
    return [computePrimeCount(limit) for limit in limits]  # line 11

def runWithProcessPool(limits, maxWorkers):            # line 12
    with concurrent.futures.ProcessPoolExecutor(max_workers=maxWorkers) as executor:  # line 13
        results = list(executor.map(computePrimeCount, limits))  # line 14
    return results                                     # line 15

if __name__ == "__main__":                             # line 16
    workloads = [50000, 50000, 50000, 50000]           # line 17

    start = time.perf_counter()                        # line 18
    seqResults = runSequential(workloads)               # line 19
    seqTime = time.perf_counter() - start              # line 20
    print(f"Sequential: {seqTime:.2f}s → {seqResults}")  # line 21

    start = time.perf_counter()                        # line 22
    poolResults = runWithProcessPool(workloads, maxWorkers=4)  # line 23
    poolTime = time.perf_counter() - start             # line 24
    print(f"4 Processes: {poolTime:.2f}s → {poolResults}")  # line 25
    print(f"Speedup: {seqTime/poolTime:.1f}x")         # line 26
```

### Line-by-Line Explanation

**Line 1** — `import concurrent.futures`: Imports the module containing both ThreadPoolExecutor and ProcessPoolExecutor, along with Future, as_completed, and wait utilities. One import gives access to the entire high-level concurrency API.

**Line 2** — `import time`: Used for perf_counter timing. perf_counter is preferred over time.time() for performance measurement because it uses the highest-resolution clock available on the platform.

**Line 3** — `import math`: Used for math.sqrt() in the primality check. Computing the square root bounds the inner loop — a number n is composite only if it has a factor up to sqrt(n).

**Line 4** — `def computePrimeCount(limit):`: Defines the CPU-heavy worker function. This must be defined at module level (not inside a function or class) for ProcessPoolExecutor to pickle it and send it to worker processes. Lambda functions and locally-defined functions cannot be pickled.

**Line 5** — `count = 0`: Initializes the counter for primes found. Each worker process has its own memory space, so there is no sharing or race condition on this variable.

**Line 6** — `for n in range(2, limit):`: Iterates over every candidate number from 2 to limit-1. This is the outer loop that makes this CPU-bound: for a limit of 50,000, this runs nearly 50,000 iterations.

**Line 7** — `if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1)):`: Trial division primality test. For each candidate n, checks whether any integer from 2 to sqrt(n) divides it evenly. If none do, n is prime. The all() short-circuits on the first divisor found, which is an optimization.

**Line 8** — `count += 1`: Increments the local prime count. No lock needed — each process owns its own count variable.

**Line 9** — `return count`: Returns the result to the calling process. This value is serialized (pickled) by the worker process and sent back through the inter-process communication channel to the main process.

**Line 10** — `def runSequential(limits):`: Defines the baseline sequential function. Runs computePrimeCount for each limit one after another on the main process — no parallelism, no IPC overhead.

**Line 11** — `return [computePrimeCount(limit) for limit in limits]`: List comprehension that calls the prime counter for each limit in sequence. With four limits of 50,000 each, total time is approximately 4x the single-task time.

**Line 12** — `def runWithProcessPool(limits, maxWorkers):`: Defines the parallel version. Accepts max_workers as a parameter so the caller can experiment with different pool sizes.

**Line 13** — `with concurrent.futures.ProcessPoolExecutor(max_workers=maxWorkers) as executor:`: Creates a pool of worker processes. The with-block ensures all processes are cleanly shut down when the block exits — pending tasks complete, worker processes terminate, resources are released. max_workers controls how many processes run concurrently; values beyond the number of physical CPU cores rarely help and can hurt.

**Line 14** — `results = list(executor.map(computePrimeCount, limits))`: Distributes each item in limits as a separate task across the process pool. executor.map() returns a lazy iterator — wrapping in list() forces evaluation, blocking until all tasks complete. Results are returned in submission order, not completion order. Under the hood: each (function, argument) pair is pickled, sent to a worker process, deserialized, executed, and the result pickled and returned.

**Line 15** — `return results`: Returns the collected results list to the caller.

**Line 16** — `if __name__ == "__main__":`: The critical guard. On macOS and Windows, Python spawns worker processes by importing the main module. Without this guard, each spawned process imports the script and hits line 13, creating another ProcessPoolExecutor, which spawns more processes — an infinite loop that crashes the system. On Linux (which uses fork rather than spawn), this guard is less critical but is still best practice for portability.

**Line 17** — `workloads = [50000, 50000, 50000, 50000]`: Four identical workloads. Identical tasks make the speedup calculation clean — we expect roughly 4x on a 4-core machine, though real speedup is always less due to IPC overhead and OS scheduling.

**Lines 18-21** — Sequential timing block: perf_counter captures a high-resolution timestamp before and after sequential execution. The difference is the wall-clock time for the sequential run.

**Lines 22-25** — Parallel timing block: Same structure as the sequential timing. The parallel run should complete in roughly one-quarter the time on a 4-core machine.

**Line 26** — `print(f"Speedup: {seqTime/poolTime:.1f}x")`: Computes the empirical speedup ratio. Theoretical maximum for 4 tasks on 4 cores is 4x. Real speedup is typically 2.5x-3.5x due to pickle overhead, OS process management, and non-uniform core speeds.

---

## Example 23.2 — Futures with Callbacks and Error Handling

```python
# Example 23.2
import concurrent.futures                              # line 1
import time                                            # line 2
import random                                          # line 3

def riskyTask(taskId):                                 # line 4
    time.sleep(random.uniform(0.1, 0.5))               # line 5
    if random.random() < 0.3:                          # line 6
        raise ValueError(f"Task {taskId} failed randomly")  # line 7
    return f"Task {taskId} succeeded"                  # line 8

def onComplete(future):                                # line 9
    if future.exception():                             # line 10
        print(f"CALLBACK: caught exception: {future.exception()}")  # line 11
    else:                                              # line 12
        print(f"CALLBACK: result = {future.result()}")  # line 13

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:  # line 14
    futures = []                                       # line 15
    for taskId in range(8):                            # line 16
        future = executor.submit(riskyTask, taskId)    # line 17
        future.add_done_callback(onComplete)           # line 18
        futures.append(future)                         # line 19

    print("\n--- Collecting results with error handling ---")  # line 20
    successCount = 0                                   # line 21
    failCount = 0                                      # line 22
    for future in concurrent.futures.as_completed(futures):  # line 23
        try:                                           # line 24
            result = future.result()                   # line 25
            successCount += 1                          # line 26
        except ValueError as e:                        # line 27
            failCount += 1                             # line 28
            print(f"Handled error: {e}")               # line 29

    print(f"Done: {successCount} succeeded, {failCount} failed")  # line 30
```

### Line-by-Line Explanation

**Line 1** — `import concurrent.futures`: Provides ThreadPoolExecutor and the as_completed function used later.

**Line 2** — `import time`: Used for time.sleep() in the worker to simulate variable-duration I/O-bound tasks.

**Line 3** — `import random`: Used to simulate non-deterministic task durations and random failures — conditions representative of real-world tasks like network calls.

**Line 4** — `def riskyTask(taskId):`: The worker function. Accepts a task identifier so log messages are traceable. Simulates a task that sometimes fails — like a network request to an unreliable endpoint.

**Line 5** — `time.sleep(random.uniform(0.1, 0.5))`: Sleeps between 100ms and 500ms, simulating variable I/O latency. Because this is a ThreadPoolExecutor, the sleep releases the GIL and other threads run freely during this time.

**Line 6** — `if random.random() < 0.3:`: Generates a random float between 0.0 and 1.0. A 30% failure rate creates a realistic scenario — some tasks fail, most succeed. This models network timeouts, API errors, and disk failures.

**Line 7** — `raise ValueError(f"Task {taskId} failed randomly")`: Raises an exception inside the worker thread. This exception is caught by the executor internally, stored in the Future object, and will be re-raised when future.result() is called. The worker thread itself does not crash — it simply completes with an exception state.

**Line 8** — `return f"Task {taskId} succeeded"`: The happy path. This string is stored in the Future as the result value.

**Line 9** — `def onComplete(future):`: Defines the callback function. Callbacks must accept exactly one argument — the completed Future. The callback is called by an internal executor thread, not the main thread; avoid doing slow or blocking work inside callbacks.

**Line 10** — `if future.exception():`: Checks whether the future completed with an exception. This call never blocks because the future is guaranteed to be done when the callback fires. Returns the exception object if one exists, or None for a successful result.

**Line 11** — `print(f"CALLBACK: caught exception: {future.exception()}")`: Handles the error case in the callback. Calling future.exception() here returns the exception object without raising it. Calling future.result() instead would re-raise it, requiring a try/except.

**Line 12-13** — `else: print(f"CALLBACK: result = {future.result()}")`: Handles the success case. Calling future.result() on a completed successful future returns the value immediately without blocking.

**Line 14** — `with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:`: Creates a thread pool with 5 workers. With 8 tasks, the first 5 start immediately; the remaining 3 wait for a worker thread to become free.

**Line 15** — `futures = []`: Collects all submitted futures for later iteration with as_completed. Order in this list matches submission order.

**Lines 16-19** — Submission loop: submits 8 tasks and immediately registers a callback on each. executor.submit() is non-blocking — it queues the task and returns a Future right away. add_done_callback() registers onComplete before the task may have even started. If the task completes before this line is reached, the callback fires immediately on this thread.

**Line 17** — `future = executor.submit(riskyTask, taskId)`: Submits the task. The future starts in PENDING state, transitions to RUNNING when a thread picks it up, and transitions to DONE when it finishes or raises.

**Line 18** — `future.add_done_callback(onComplete)`: Registers the callback. The callback fires on a thread from the executor's thread pool — typically the thread that ran the task. Multiple callbacks can be registered; they fire in registration order.

**Line 19** — `futures.append(future)`: Saves the future reference. Without this, we lose the ability to iterate with as_completed or collect results.

**Line 20** — `print("\n--- Collecting results with error handling ---")`: Output separator. Note that by this point, some callbacks may have already fired — the submission loop and the callback system run concurrently.

**Lines 21-22** — `successCount = 0` / `failCount = 0`: Counters for the summary. These are only accessed from the main thread in this loop, so no synchronization is needed.

**Line 23** — `for future in concurrent.futures.as_completed(futures):`: as_completed takes the list of all 8 futures and yields them one by one as they complete. The loop blocks waiting for the next completion; when a future finishes, it is yielded. This is fundamentally different from iterating futures in list order — if the 8th submitted task finishes first, it is yielded first.

**Line 24-25** — `try: result = future.result()`: Each yielded future is done, so future.result() either returns immediately with the value or raises the stored exception.

**Line 26** — `successCount += 1`: Increments only on the successful path — after future.result() returns without raising.

**Lines 27-29** — `except ValueError as e:`: Catches the specific exception type raised by the worker. This is ordinary sequential exception handling — the exception was raised in a thread, stored in the future, and re-raised here cleanly. The fail counter increments and the error is reported.

**Line 30** — `print(f"Done: {successCount} succeeded, {failCount} failed")`: Final summary. With 8 tasks and a 30% failure rate, expect approximately 5-6 successes and 2-3 failures per run.

---

## Example 23.3 — asyncio.Future and Wrapping Thread-Pool Futures

```python
# Example 23.3
import asyncio                                         # line 1
import concurrent.futures                              # line 2
import time                                            # line 3

def legacyBlockingCompute(value):                      # line 4
    time.sleep(0.2)                                    # line 5
    return value * value                               # line 6

async def asyncWrapper(executor, value):               # line 7
    loop = asyncio.get_event_loop()                    # line 8
    result = await loop.run_in_executor(               # line 9
        executor,                                      # line 10
        legacyBlockingCompute,                         # line 11
        value                                          # line 12
    )                                                  # line 13
    return result                                      # line 14

async def main():                                      # line 15
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:  # line 16
        start = time.perf_counter()                    # line 17
        results = await asyncio.gather(                # line 18
            asyncWrapper(executor, 10),                # line 19
            asyncWrapper(executor, 20),                # line 20
            asyncWrapper(executor, 30),                # line 21
            asyncWrapper(executor, 40),                # line 22
        )                                              # line 23
        elapsed = time.perf_counter() - start          # line 24
        print(f"Results: {results}")                   # line 25
        print(f"Time: {elapsed:.2f}s (should be ~0.2s, not 0.8s)")  # line 26

asyncio.run(main())                                    # line 27
```

### Line-by-Line Explanation

**Line 1** — `import asyncio`: Provides the event loop, gather, get_event_loop, and run_in_executor. This is the async side of the hybrid architecture.

**Line 2** — `import concurrent.futures`: Provides ThreadPoolExecutor — the thread pool that will run the blocking function. The event loop itself does not execute blocking code; the thread pool does.

**Line 3** — `import time`: Used for both time.sleep() (blocking — runs in a thread) and time.perf_counter() (timing — runs in the main coroutine).

**Line 4** — `def legacyBlockingCompute(value):`: A regular synchronous function — not a coroutine. Represents a legacy library call: an old image processing library, a synchronous database driver, or any third-party code that does not support async. It is defined without async def because it genuinely blocks.

**Line 5** — `time.sleep(0.2)`: Simulates 200ms of blocking work. If this were called directly inside a coroutine (without run_in_executor), it would block the entire event loop for 200ms — freezing all other coroutines. Run in a thread, it blocks only that thread; the event loop continues running other coroutines on the main thread.

**Line 6** — `return value * value`: The computed result. This return value is pickled (actually, for threads it's just a Python object reference) and made available to the awaiting coroutine through the asyncio.Future mechanism.

**Line 7** — `async def asyncWrapper(executor, value):`: A coroutine that wraps the synchronous function for use in the async world. This is the adapter pattern — it bridges the synchronous and asynchronous APIs. It accepts the executor as a parameter so the same thread pool is reused across all calls.

**Line 8** — `loop = asyncio.get_event_loop()`: Gets a reference to the running event loop. This is the loop currently executing this coroutine. Starting with Python 3.10, asyncio.get_event_loop() in a running async context returns the current running loop. In newer code, asyncio.get_running_loop() is preferred as it's more explicit.

**Line 9-12** — `result = await loop.run_in_executor(executor, legacyBlockingCompute, value)`: This is the core bridge. run_in_executor submits legacyBlockingCompute(value) to the thread pool executor, then returns an asyncio.Future. The await keyword suspends this coroutine at this point — control returns to the event loop. The event loop can now run other coroutines while the thread executes the blocking function. When the thread finishes, the event loop resumes this coroutine with the result. The function is passed as a callable (line 11) and its argument is passed separately (line 12) — run_in_executor does not accept keyword arguments directly; use functools.partial for more complex argument passing.

**Line 13** — Closing parenthesis for run_in_executor. The await expression spans lines 9-13.

**Line 14** — `return result`: Returns the computed value (100, 400, 900, 1600 for the inputs 10, 20, 30, 40). This return value flows back through the asyncio.Future to whoever awaits asyncWrapper.

**Line 15** — `async def main():`: The top-level coroutine. asyncio.run() on line 27 creates a new event loop, runs this coroutine, and shuts down the loop when it completes.

**Line 16** — `with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:`: Creates the thread pool synchronously and makes it available to all coroutines via the executor variable. The with-block ensures the thread pool shuts down cleanly after all submitted tasks complete. max_workers=4 allows four simultaneous threads — one for each concurrent call in gather.

**Line 17** — `start = time.perf_counter()`: Captures the start time inside the coroutine, on the main thread. time.perf_counter() is safe to call in a coroutine — it's fast and non-blocking.

**Lines 18-23** — `results = await asyncio.gather(...)`: gather takes multiple awaitables and runs them concurrently. All four asyncWrapper coroutines are started simultaneously. Each immediately suspends at line 9 (the await run_in_executor), dispatching its blocking function to the thread pool. The thread pool has 4 workers, so all four blocking calls start immediately. The event loop waits for all four to complete. gather collects results in the order the awaitables were passed — not the order they complete. Total time: approximately 200ms (one task's duration), not 800ms (four tasks in sequence).

**Line 24** — `elapsed = time.perf_counter() - start`: Measures wall-clock time from submission to all results collected. Should be approximately 0.2 seconds — demonstrating that all four 200ms blocking calls ran concurrently.

**Line 25** — `print(f"Results: {results}")`: Prints [100, 400, 900, 1600] — the squares of 10, 20, 30, 40. gather preserves submission order in the results list even though the tasks ran concurrently.

**Line 26** — `print(f"Time: {elapsed:.2f}s (should be ~0.2s, not 0.8s)")`: The diagnostic message that demonstrates the performance benefit. If elapsed were 0.8 seconds, it would mean the blocking calls ran sequentially — indicating the calls were made directly in a coroutine without run_in_executor.

**Line 27** — `asyncio.run(main())`: Entry point. Creates a new event loop, runs the main coroutine until it completes, cleans up the loop. This is the correct way to run async code from synchronous context in Python 3.7+. It replaces the older pattern of manually calling loop.run_until_complete().
