# Code — Lecture 22: The Async Event Loop — How Cooperative Concurrency Works Under the Hood

---

## Example 22.1 — Visualizing the Event Loop Schedule

```python
# Example 22.1
import asyncio                                         # line 1
import time                                            # line 2

async def task(name, workDuration, ioDelay):           # line 3
    print(f"[{time.perf_counter():.2f}] {name}: starting CPU work")  # line 4
    # Simulate CPU work — no yield, monopolizes loop  # line 5
    end = time.perf_counter() + workDuration           # line 6
    while time.perf_counter() < end:                   # line 7
        pass                                           # line 8
    print(f"[{time.perf_counter():.2f}] {name}: yielding to I/O")  # line 9
    await asyncio.sleep(ioDelay)                       # line 10
    print(f"[{time.perf_counter():.2f}] {name}: I/O complete, resuming")  # line 11
    return f"{name} done"                              # line 12

async def main():                                      # line 13
    start = time.perf_counter()                        # line 14
    results = await asyncio.gather(                    # line 15
        task("A", workDuration=0.1, ioDelay=0.5),     # line 16
        task("B", workDuration=0.1, ioDelay=0.3),     # line 17
        task("C", workDuration=0.1, ioDelay=0.2),     # line 18
    )                                                  # line 19
    total = time.perf_counter() - start                # line 20
    print(f"All done in {total:.2f}s")                 # line 21
    print(results)                                     # line 22

asyncio.run(main())                                    # line 23
```

**Line-by-line explanation:**

- **Line 1:** Import `asyncio`, the standard library module that provides the event loop, coroutine scheduling, and all async I/O primitives. This single import brings in the entire async runtime.

- **Line 2:** Import `time` for `time.perf_counter()`, which provides a high-resolution clock suitable for measuring sub-second durations. This will be used to timestamp each event in the coroutine lifecycle.

- **Line 3:** Define `task` as a coroutine function with three parameters: `name` for identification in output, `workDuration` controlling how many seconds the CPU-busy loop runs, and `ioDelay` controlling how long the simulated I/O wait lasts. The `async def` keyword makes this a coroutine — calling `task(...)` returns a coroutine object without starting execution.

- **Line 4:** Print a timestamped entry message showing the exact moment this coroutine begins its CPU work. The `:.2f` format rounds the float to two decimal places, giving readable output like `[0.00] A: starting CPU work`.

- **Line 5:** This comment is the critical annotation. The busy-wait loop below contains NO await expression — which means the event loop cannot run anything else for the entire duration of this loop. This is intentional in the example so that students can see the sequential nature of CPU work before the coroutines reach their I/O phase.

- **Lines 6–8:** The CPU busy-wait loop. Line 6 computes the deadline: `time.perf_counter() + workDuration` gives the timestamp at which the loop should stop. Lines 7–8 spin in a tight loop doing nothing (`pass`) until the current time reaches or exceeds the deadline. This simulates CPU-bound computation and deliberately monopolizes the event loop thread for `workDuration` seconds. There is no `await` inside, so the event loop is completely blocked for this duration.

- **Line 9:** Print a timestamped message when the CPU work finishes and the coroutine is about to yield to the event loop via its I/O await. Observing this timestamp shows exactly when each coroutine transitions from CPU phase to I/O phase.

- **Line 10:** `await asyncio.sleep(ioDelay)` is the suspension point. This line tells the event loop: "Register this coroutine to be woken after `ioDelay` seconds, then go run something else." The coroutine is moved from the ready queue into the I/O registry. The event loop is now free to run other ready coroutines or poll for I/O completions. This is the key difference between `asyncio.sleep` (cooperative, non-blocking) and `time.sleep` (blocking, freezes the loop).

- **Lines 11–12:** After the sleep timer fires and the event loop resumes this coroutine, line 11 prints the completion timestamp. Line 12 returns a result string, which `asyncio.gather` will collect and return as part of its results list.

- **Line 13:** Define the `main` coroutine, the entry point for the async program. It is itself a coroutine because it uses `await`; it will be run by `asyncio.run()` on line 23.

- **Line 14:** Record the wall-clock start time before launching all tasks, so we can measure total elapsed time at the end.

- **Lines 15–19:** `asyncio.gather(...)` takes multiple coroutine objects and schedules them all to run concurrently. It returns a single coroutine that completes when ALL of the provided coroutines complete, and collects their return values into a list in the same order as the arguments. Because the CPU busy-wait loops run sequentially before any `await` is hit, tasks A, B, and C each run their CPU phase one after another — observable in the timestamps. Once all three hit their `await asyncio.sleep(...)` lines, they all enter the I/O registry simultaneously, and the I/O waits overlap.

- **Line 20:** Compute the total elapsed wall-clock time. Because the I/O waits overlap, the total time should be approximately: (0.1 + 0.1 + 0.1) seconds of sequential CPU work plus 0.5 seconds of overlapping I/O wait — roughly 0.8 seconds total, not 2.7 seconds (which would be the sequential sum of all work and I/O).

- **Lines 21–22:** Print the total elapsed time and the list of return values. The order of results in the list matches the argument order to `gather` — not the completion order — because `gather` preserves submission order.

- **Line 23:** `asyncio.run(main())` is the standard entry point for every async Python program. It creates a new event loop, calls `main()` to get a coroutine object, runs that coroutine to completion, then closes the loop and cleans up all resources. This must be called from synchronous code (not from inside another coroutine). Using `asyncio.run()` rather than manually creating and running a loop is the recommended modern practice as of Python 3.7+.

---

## Example 22.2 — Blocking vs Non-Blocking, and run_in_executor

```python
# Example 22.2
import asyncio                                         # line 1
import time                                            # line 2
import concurrent.futures                              # line 3

def blockingOperation(duration):                       # line 4
    time.sleep(duration)                               # line 5
    return f"Blocking result after {duration}s"        # line 6

async def wrongWay():                                  # line 7
    print("Wrong: calling blocking sleep directly")    # line 8
    time.sleep(1)                                      # line 9  BLOCKS EVENT LOOP
    print("Wrong: done")                               # line 10
    return "wrong result"                              # line 11

async def rightWay(loop, executor):                    # line 12
    print("Right: offloading blocking call to executor")  # line 13
    result = await loop.run_in_executor(               # line 14
        executor,                                      # line 15
        blockingOperation,                             # line 16
        1                                              # line 17
    )                                                  # line 18
    print(f"Right: got result: {result}")              # line 19
    return result                                      # line 20

async def main():                                      # line 21
    loop = asyncio.get_event_loop()                    # line 22
    with concurrent.futures.ThreadPoolExecutor() as executor:  # line 23
        results = await asyncio.gather(                # line 24
            rightWay(loop, executor),                  # line 25
            rightWay(loop, executor),                  # line 26
        )                                              # line 27
    print(f"Results: {results}")                       # line 28

asyncio.run(main())                                    # line 29
```

**Line-by-line explanation:**

- **Line 1:** Import `asyncio` for the event loop and coroutine infrastructure.

- **Line 2:** Import `time` for `time.sleep`, the blocking sleep function. This is the function we will demonstrate calling incorrectly, and then correctly via an executor.

- **Line 3:** Import `concurrent.futures` for `ThreadPoolExecutor`. This module provides the thread pool that `run_in_executor` will use to run blocking functions off the event loop thread.

- **Lines 4–6:** Define `blockingOperation` as a regular (non-async) function. It calls `time.sleep(duration)` — a true OS-level blocking call that puts the calling thread to sleep for the specified number of seconds. Because this is a regular function (not a coroutine), it will block whichever thread calls it. When called from within a coroutine directly, it blocks the event loop thread. When called via `run_in_executor`, it blocks a worker thread in the pool instead, leaving the event loop thread free.

- **Lines 7–11:** Define `wrongWay`, a coroutine that makes the critical mistake: it calls `time.sleep(1)` directly on line 9. This is annotated with a comment to make the error obvious. When this coroutine runs, `time.sleep(1)` blocks the event loop's thread for exactly one second. During that second, every other coroutine in the entire program is frozen — the event loop cannot poll for I/O completions, cannot run other ready coroutines, cannot process timers. If this were a production server, every active request would stall for one full second.

- **Line 12:** Define `rightWay` as a coroutine that accepts the event loop reference and a thread pool executor. Passing these in as parameters (rather than obtaining them inside the function) is a common pattern that makes the coroutine testable and explicit about its dependencies.

- **Line 13:** Print a message before the executor call, making it visible in output when this coroutine starts its work.

- **Lines 14–18:** `loop.run_in_executor(executor, blockingOperation, 1)` submits `blockingOperation(1)` to the thread pool for execution on a worker thread. This call returns a `Future` (an asyncio Future, wrapping the concurrent.futures Future). The `await` on line 14 suspends this coroutine and registers it in the event loop's I/O registry. The event loop is now completely free to run other coroutines, poll for I/O, and process timers. When the worker thread finishes `blockingOperation`, the Future is resolved, and the event loop resumes this coroutine with the return value as `result`. The arguments to `run_in_executor` are: the executor (line 15), the callable (line 16), and positional arguments to pass to the callable (line 17). Note that keyword arguments are not supported directly; use `functools.partial` to pass keyword arguments.

- **Lines 19–20:** Print the result received from the executor and return it so `asyncio.gather` can collect it.

- **Line 21:** Define `main`, the top-level coroutine that orchestrates the demonstration.

- **Line 22:** `asyncio.get_event_loop()` retrieves the currently running event loop. Inside an async context (a running coroutine), this returns the event loop that is currently driving execution. This reference is needed to call `run_in_executor`.

- **Line 23:** Create a `ThreadPoolExecutor` as a context manager. On exit from the `with` block, the executor calls `shutdown(wait=True)`, ensuring all submitted tasks complete and worker threads are joined before proceeding. Without the context manager, submitted tasks might be abandoned when the program exits.

- **Lines 24–27:** `asyncio.gather(rightWay(loop, executor), rightWay(loop, executor))` launches two concurrent instances of `rightWay`. Both coroutines start, both hit their `await loop.run_in_executor(...)` lines, and both submit their blocking operations to the thread pool. The thread pool runs both blocking sleeps on two separate worker threads — they execute in parallel. The event loop awaits both futures concurrently. Total elapsed time for both: approximately one second (the duration of one blocking sleep), not two seconds.

- **Line 28:** Print the results list after both `rightWay` coroutines have completed. Each element is the string returned by `blockingOperation`.

- **Line 29:** Start the event loop with `asyncio.run(main())`. The executor context manager inside `main` ensures worker threads are cleaned up before `asyncio.run` closes the event loop.

---

## Example 22.3 — Long Loop with Periodic Yield

```python
# Example 22.3
import asyncio                                         # line 1
import time                                            # line 2

async def longBatchProcessor(items, batchSize=1000):  # line 3
    results = []                                       # line 4
    for index, item in enumerate(items):               # line 5
        results.append(item * 2)                       # line 6
        if index % batchSize == 0:                     # line 7
            await asyncio.sleep(0)                     # line 8  yield to event loop
            print(f"Processed {index}/{len(items)}")   # line 9
    return results                                     # line 10

async def heartbeat():                                 # line 11
    while True:                                        # line 12
        print(f"[heartbeat] {time.perf_counter():.1f}")  # line 13
        await asyncio.sleep(0.1)                       # line 14

async def main():                                      # line 15
    items = list(range(5000))                          # line 16
    heartbeatTask = asyncio.create_task(heartbeat())   # line 17
    results = await longBatchProcessor(items)          # line 18
    heartbeatTask.cancel()                             # line 19
    print(f"Processed {len(results)} items")           # line 20

asyncio.run(main())                                    # line 21
```

**Line-by-line explanation:**

- **Line 1:** Import `asyncio` for coroutine scheduling and the `asyncio.sleep(0)` yield mechanism.

- **Line 2:** Import `time` for `time.perf_counter()`, used in the heartbeat to print timestamps that make the event loop's responsiveness visible.

- **Line 3:** Define `longBatchProcessor` as a coroutine that processes a list of items. The `batchSize` parameter controls how many items are processed between cooperative yield points. A smaller `batchSize` gives the event loop more frequent opportunities to run other coroutines but increases the overhead of context switching; a larger `batchSize` does more work between yields but may make the loop less responsive.

- **Line 4:** Initialize an empty `results` list that will accumulate the processed output. This list lives in the coroutine's local frame and is not shared with any other coroutine — there is no need for synchronization.

- **Lines 5–9:** The main processing loop. `enumerate(items)` yields `(index, item)` pairs, giving both the position and the value. Line 6 performs the actual computation — here just `item * 2` — and appends the result. Lines 7–9 are the cooperative yield mechanism: every `batchSize` items, the coroutine awaits `asyncio.sleep(0)`. This is the critical pattern. Line 8 — `await asyncio.sleep(0)` — is the explicit yield point. When this line executes, the coroutine suspends immediately, the event loop runs all other ready coroutines (including the heartbeat), and then resumes this coroutine on the very next event loop cycle. The zero-second delay means no real time is wasted; the yield is purely a scheduling gesture. Line 9 prints progress so you can observe the yield frequency.

- **Line 10:** After processing all items, return the full results list. The caller (`main`) will receive this list when it awaits the coroutine.

- **Lines 11–14:** Define `heartbeat` as a simple infinite-loop coroutine that prints a timestamp and then sleeps for 0.1 seconds. This coroutine simulates a background health monitor that needs to fire regularly. In a real system this might send a heartbeat to a monitoring service, flush metrics, or update a "last seen" timestamp. The `while True` loop combined with `await asyncio.sleep(0.1)` makes the event loop responsible for waking this coroutine every tenth of a second. Without the yield points in `longBatchProcessor`, the heartbeat will be starved — its 0.1-second timer will fire in the event loop's timer registry, but the loop will not be able to check it until `longBatchProcessor` finally yields.

- **Line 15:** Define `main`, which coordinates both the batch processor and the heartbeat.

- **Line 16:** Create the list of items to process: integers 0 through 4999. In a real application this might be a list of database records, file paths, or API response objects.

- **Line 17:** `asyncio.create_task(heartbeat())` schedules the heartbeat coroutine as an independent Task. A Task wraps a coroutine and runs it concurrently alongside the current coroutine — the event loop will run it whenever the current coroutine yields. Unlike passing a coroutine to `await` directly (which would block until it finished), `create_task` schedules it in the background. The returned `heartbeatTask` reference is needed to cancel the task later.

- **Line 18:** `await longBatchProcessor(items)` runs the batch processor coroutine. Because `longBatchProcessor` contains `await asyncio.sleep(0)` calls, the event loop gets control at those points and runs the heartbeat. Without `await` here, the batch would not run at all. The `await` keyword drives the coroutine to completion and collects its return value into `results`.

- **Line 19:** `heartbeatTask.cancel()` sends a cancellation request to the heartbeat task. The task receives a `CancelledError` at its next `await` point — which is the `await asyncio.sleep(0.1)` on line 14. Because the heartbeat does not catch `CancelledError`, the exception propagates up and the task terminates. The cancellation is not immediate synchronously; it is delivered at the next opportunity when the event loop drives the task. In this program, `asyncio.run` will clean up any remaining tasks when `main` returns.

- **Line 20:** Print the count of processed results. This should always equal the length of `items` (5000), confirming that no items were lost.

- **Line 21:** `asyncio.run(main())` creates the event loop, runs `main` to completion, and closes the loop. All tasks created during the run are cancelled and awaited during cleanup if they have not already finished.
