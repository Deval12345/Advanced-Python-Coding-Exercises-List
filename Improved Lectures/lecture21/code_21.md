# Code Examples — Lecture 21: Async/Await Fundamentals — Cooperative Concurrency with a Single Thread

---

## Example 21.1 — Basic Coroutines and asyncio.gather

```python
# Example 21.1
import asyncio                                         # line 1
import time                                            # line 2

async def fetchData(sourceId, delaySeconds):           # line 3
    print(f"Starting fetch {sourceId}")                # line 4
    await asyncio.sleep(delaySeconds)                  # line 5
    result = f"Data from source {sourceId}"            # line 6
    print(f"Completed fetch {sourceId}")               # line 7
    return result                                      # line 8

async def main():                                      # line 9
    startTime = time.perf_counter()                    # line 10
    results = await asyncio.gather(                    # line 11
        fetchData("A", 1.0),                           # line 12
        fetchData("B", 0.5),                           # line 13
        fetchData("C", 0.8),                           # line 14
    )                                                  # line 15
    elapsed = time.perf_counter() - startTime          # line 16
    for result in results:                             # line 17
        print(result)                                  # line 18
    print(f"Total time: {elapsed:.2f}s (vs ~2.3s sequential)")  # line 19

asyncio.run(main())                                    # line 20
```

### Line-by-Line Explanation

**Line 1 — `import asyncio`**
Imports the standard library asyncio module, which provides the event loop, coroutine scheduling primitives (gather, create_task, wait), and async-compatible I/O utilities like asyncio.sleep.

**Line 2 — `import time`**
Imports the time module for high-resolution wall-clock timing. Used here to measure actual elapsed time and demonstrate that concurrent coroutines complete faster than sequential ones.

**Line 3 — `async def fetchData(sourceId, delaySeconds):`**
Defines a coroutine function using the `async def` keyword. Calling `fetchData("A", 1.0)` does NOT execute the body — it returns a coroutine object. The event loop must drive that object for execution to begin. The two parameters represent an identifier for the source and the simulated network delay in seconds.

**Line 4 — `print(f"Starting fetch {sourceId}")`**
Runs synchronously inside the coroutine — no I/O, no suspension here. Prints immediately when the event loop first starts driving this coroutine, showing that all three fetches begin before any of them completes.

**Line 5 — `await asyncio.sleep(delaySeconds)`**
This is the suspension point. `asyncio.sleep` is an async-compatible sleep that does not block the OS thread. The `await` keyword suspends this coroutine, returns control to the event loop, and schedules a wakeup after `delaySeconds` seconds. While this coroutine waits, the event loop is free to drive any other ready coroutine. This is the core mechanism that makes concurrent I/O possible in a single thread.

**Line 6 — `result = f"Data from source {sourceId}"`**
Constructs the return value after the sleep completes and the event loop resumes this coroutine. This represents the payload that would, in a real application, be the HTTP response body or database query result.

**Line 7 — `print(f"Completed fetch {sourceId}")`**
Prints completion in the actual order the coroutines finish, which depends on their delay. The B coroutine (0.5 s) will print first, then C (0.8 s), then A (1.0 s) — demonstrating that coroutines complete in delay order, not creation order.

**Line 8 — `return result`**
Returns the result string. asyncio.gather collects this return value and places it in the results list at the index corresponding to this coroutine's position in the gather call.

**Line 9 — `async def main():`**
The top-level entry-point coroutine. All async programs need at least one root coroutine that is handed to `asyncio.run()`. It is defined as `async def` because it must use `await` internally.

**Line 10 — `startTime = time.perf_counter()`**
Captures a high-resolution timestamp before any concurrent work begins. `time.perf_counter()` is the recommended clock for measuring short elapsed intervals; it is monotonic and has sub-microsecond resolution.

**Lines 11–15 — `results = await asyncio.gather(...)`**
`asyncio.gather` accepts an arbitrary number of awaitables (here, three coroutine objects). It schedules all of them concurrently — they all start running on the event loop immediately. `gather` itself is awaitable; awaiting it suspends `main` until every coroutine in the group has either returned or raised an exception. The return value is a list of results in the same order as the arguments, regardless of which coroutine finished first.

**Lines 12–14 — The three fetchData calls inside gather**
Each call produces a coroutine object (not a result). All three are handed to gather at once, so the event loop starts driving all three before any of them reaches its first await. This is why all three "Starting fetch" messages appear before any "Completed fetch" message.

**Line 16 — `elapsed = time.perf_counter() - startTime`**
Computes wall-clock elapsed time after gather returns. The expected result is approximately 1.0 second — the duration of the longest single coroutine — rather than 2.3 seconds (1.0 + 0.5 + 0.8), which would be the sequential total.

**Lines 17–18 — `for result in results: print(result)`**
Iterates the results list returned by gather. Results appear in argument order: A, then B, then C — even though B and C finished before A. gather preserves input order, making it easy to correlate results with their sources.

**Line 19 — `print(f"Total time: {elapsed:.2f}s (vs ~2.3s sequential)")`**
Quantifies the concurrency benefit. The `:.2f` format specifier rounds the float to two decimal places for a clean display.

**Line 20 — `asyncio.run(main())`**
The entry point for all asyncio programs. `asyncio.run` creates a new event loop, runs the given coroutine to completion, closes the loop, and returns the coroutine's return value (None here). It must be called from synchronous code — you cannot call `asyncio.run` from inside a running event loop.

---

## Example 21.2 — Tasks with create_task and Cancellation

```python
# Example 21.2
import asyncio                                         # line 1
import time                                            # line 2

async def longRunningJob(jobId, duration):             # line 3
    try:                                               # line 4
        print(f"Job {jobId} started")                  # line 5
        await asyncio.sleep(duration)                  # line 6
        print(f"Job {jobId} completed")                # line 7
        return f"Result_{jobId}"                       # line 8
    except asyncio.CancelledError:                     # line 9
        print(f"Job {jobId} was cancelled")            # line 10
        raise                                          # line 11

async def main():                                      # line 12
    task1 = asyncio.create_task(longRunningJob("A", 2.0))  # line 13
    task2 = asyncio.create_task(longRunningJob("B", 0.5))  # line 14
    task3 = asyncio.create_task(longRunningJob("C", 1.0))  # line 15

    done, pending = await asyncio.wait(                # line 16
        [task1, task2, task3],                         # line 17
        timeout=1.2                                    # line 18
    )

    print(f"Done: {len(done)}, Pending: {len(pending)}")  # line 19
    for task in pending:                               # line 20
        task.cancel()                                  # line 21
    await asyncio.gather(*pending, return_exceptions=True)  # line 22

asyncio.run(main())                                    # line 23
```

### Line-by-Line Explanation

**Line 1–2 — imports**
Same imports as Example 21.1. asyncio provides the task and wait primitives; time is available if needed for timing measurements.

**Line 3 — `async def longRunningJob(jobId, duration):`**
Defines a coroutine that simulates a long-running background job. Takes a job identifier string and a duration in seconds. The `try/except` block around the entire body is the standard pattern for a coroutine that must handle cancellation gracefully.

**Line 4 — `try:`**
Opens the try block that wraps all job logic. This is required because cancellation is delivered as an exception — `asyncio.CancelledError` — at the coroutine's nearest suspension point. Without a try block, the exception would propagate uncaught, which is fine for simple cases but prevents any cleanup from running.

**Line 5 — `print(f"Job {jobId} started")`**
Runs immediately when the event loop first drives this task. Since create_task is called three times before any await in main, the event loop will start all three jobs the next time it cycles — and all three "started" messages print before any job completes.

**Line 6 — `await asyncio.sleep(duration)`**
The suspension point. This is also where cancellation is delivered. When `task.cancel()` is called on a task, the event loop injects `CancelledError` at the task's current suspension point. If a task is sleeping here when cancelled, the CancelledError is raised from this line.

**Line 7 — `print(f"Job {jobId} completed")`**
Only executes if the sleep completes normally (i.e., the task is not cancelled during the sleep). Jobs B and C will reach this line; job A will not because it is cancelled before its 2-second sleep finishes.

**Line 8 — `return f"Result_{jobId}"`**
The return value collected by gather or accessible via `task.result()`. Only jobs that complete normally produce a return value; cancelled tasks raise `CancelledError` when their result is requested.

**Line 9 — `except asyncio.CancelledError:`**
Catches the cancellation exception specifically. `asyncio.CancelledError` is a subclass of `BaseException` (not `Exception`) in Python 3.8+, which means a bare `except Exception` will not catch it — you must name it explicitly. This is intentional: it prevents accidental suppression of cancellation.

**Line 10 — `print(f"Job {jobId} was cancelled")`**
Performs the cleanup action — in a real application this might close a file handle, roll back a partial database write, or log a metric. The cleanup runs before the exception is re-raised.

**Line 11 — `raise`**
Re-raises the `CancelledError` without modification. This is mandatory. If you do not re-raise, the task appears to complete normally, the event loop considers it done, and the `await asyncio.gather(*pending, ...)` on line 22 would not receive the cancellation signal. Always re-raise `CancelledError` after cleanup.

**Line 12 — `async def main():`**
The root coroutine. Creates three tasks, waits for them with a timeout, then cancels stragglers.

**Lines 13–15 — `asyncio.create_task(...)`**
Each call wraps the coroutine in a `Task` object and schedules it on the running event loop immediately. Crucially, the current coroutine (main) does not suspend here — it continues executing synchronously through all three create_task calls. The tasks are queued and begin running the next time main suspends (at line 16). This is the key difference from gather: create_task lets you start work and continue without waiting.

**Lines 16–18 — `done, pending = await asyncio.wait([task1, task2, task3], timeout=1.2)`**
`asyncio.wait` watches the given set of tasks. With `timeout=1.2`, it returns after 1.2 seconds regardless of whether all tasks are done. The return value is a tuple of two sets: `done` contains tasks that finished (returned, raised, or were cancelled) within the timeout; `pending` contains tasks that are still running. After 1.2 seconds, jobs B (0.5 s) and C (1.0 s) are done; job A (2.0 s) is still pending.

**Line 19 — `print(f"Done: {len(done)}, Pending: {len(pending)}")`**
Displays the count of finished vs still-running tasks. Expected output: "Done: 2, Pending: 1".

**Lines 20–21 — `for task in pending: task.cancel()`**
Iterates the pending set and calls `cancel()` on each. `task.cancel()` does not immediately stop the task — it schedules the delivery of `CancelledError` at the task's next suspension point. The task is not actually cancelled until the event loop delivers the exception and the task coroutine handles it.

**Line 22 — `await asyncio.gather(*pending, return_exceptions=True)`**
Awaits the cancelled tasks to give the event loop time to actually deliver the cancellation and let the tasks run their cleanup code (the except block). `return_exceptions=True` is required here: without it, gather re-raises the first exception it sees (which would be `CancelledError`), crashing main before all tasks have been properly drained. With `return_exceptions=True`, exceptions are returned as values in the results list rather than being raised.

**Line 23 — `asyncio.run(main())`**
Starts the event loop and runs the main coroutine to completion.

---

## Example 21.3 — Async Context Manager and Async Generator

```python
# Example 21.3
import asyncio                                         # line 1

class AsyncDatabaseConnection:                         # line 2
    def __init__(self, dbUrl):                         # line 3
        self.dbUrl = dbUrl                             # line 4
        self.isOpen = False                            # line 5

    async def __aenter__(self):                        # line 6
        await asyncio.sleep(0.01)                      # line 7
        self.isOpen = True                             # line 8
        print(f"Connected to {self.dbUrl}")            # line 9
        return self                                    # line 10

    async def __aexit__(self, exc_type, exc_val, exc_tb):  # line 11
        await asyncio.sleep(0.01)                      # line 12
        self.isOpen = False                            # line 13
        print(f"Disconnected from {self.dbUrl}")       # line 14
        return False                                   # line 15

    async def fetchRows(self, count):                  # line 16
        for i in range(count):                         # line 17
            await asyncio.sleep(0.005)                 # line 18
            yield {"id": i, "value": i * 10}           # line 19

async def main():                                      # line 20
    async with AsyncDatabaseConnection("db://prod") as conn:  # line 21
        async for row in conn.fetchRows(3):            # line 22
            print(f"Row: {row}")                       # line 23

asyncio.run(main())                                    # line 24
```

### Line-by-Line Explanation

**Line 1 — `import asyncio`**
Only asyncio is needed for this example. The class demonstrates async resource management and streaming data access patterns without requiring additional imports.

**Line 2 — `class AsyncDatabaseConnection:`**
Defines a class that implements the async context manager protocol. Any class with both `__aenter__` and `__aexit__` defined as `async def` methods can be used with `async with`. This mirrors the synchronous context manager protocol (`__enter__` and `__exit__`) but suspends the coroutine during setup and teardown.

**Lines 3–5 — `__init__`**
Standard synchronous constructor. Stores the database URL string and initializes `isOpen` to False. The constructor is always synchronous — async work happens in `__aenter__`, not `__init__`. This is a common pattern: construct the object synchronously, open the connection asynchronously.

**Line 6 — `async def __aenter__(self):`**
The async entry hook. The `async with` statement awaits this method before executing the block body. Defining it as `async def` allows it to perform I/O — such as establishing a network connection, running an authentication handshake, or acquiring a pooled connection — without blocking the event loop.

**Line 7 — `await asyncio.sleep(0.01)`**
Simulates the network latency of opening a real database connection. A real implementation might await `asyncpg.connect(...)` or `aiohttp.ClientSession().__aenter__()` here. This await suspends the coroutine for 10 ms; other coroutines can run during this time.

**Line 8 — `self.isOpen = True`**
Updates the connection state after the simulated handshake completes. In production code, this flag would guard against using a connection that was never successfully opened.

**Line 9 — `print(f"Connected to {self.dbUrl}")`**
Logs the successful connection. In a real driver this might log to a structured logger or record a metric.

**Line 10 — `return self`**
Returns the connection object itself. This becomes the value bound to the `conn` variable in `async with ... as conn`. Returning `self` is the standard pattern when the context manager and the resource are the same object.

**Line 11 — `async def __aexit__(self, exc_type, exc_val, exc_tb):`**
The async exit hook. Awaited by the `async with` statement when the block exits — whether normally or due to an exception. The three parameters carry exception information if an exception occurred inside the block, or are all None for normal exit. Like its synchronous counterpart, `__aexit__` can perform cleanup (closing connections, releasing locks) that itself requires I/O.

**Line 12 — `await asyncio.sleep(0.01)`**
Simulates the network round-trip required to gracefully close the connection. A real implementation might send a quit message to the database server or return a connection to a pool.

**Line 13 — `self.isOpen = False`**
Updates state to reflect that the connection is closed. Any further use of `conn` after the `async with` block would now correctly detect the closed state.

**Line 14 — `print(f"Disconnected from {self.dbUrl}")`**
Logs the teardown. This is always reached — even if an exception occurred inside the `async with` block — because `__aexit__` is guaranteed to run.

**Line 15 — `return False`**
Returning False from `__aexit__` means "do not suppress any exception that occurred inside the block." If `__aexit__` returns True, any exception is swallowed. Returning False is the correct default: let exceptions propagate naturally so callers can handle them.

**Line 16 — `async def fetchRows(self, count):`**
An async generator method. The combination of `async def` and `yield` makes this an asynchronous generator — it implements the async iteration protocol automatically. Each call to `__anext__` on the resulting object runs the generator body up to the next `yield` (or until the function returns, signalling StopAsyncIteration).

**Line 17 — `for i in range(count):`**
A standard synchronous for loop inside the async generator. This is fine — not every line in an async function needs to be asynchronous. The synchronous loop drives the iteration counter; the asynchronous behavior comes from the await on line 18.

**Line 18 — `await asyncio.sleep(0.005)`**
Simulates the per-row network latency of fetching a row from a remote database. In a real async database cursor, this might be `await cursor.fetchone()`. The await suspends the generator at each row, giving the event loop an opportunity to run other coroutines between rows — which is what makes async streaming memory-efficient and non-blocking.

**Line 19 — `yield {"id": i, "value": i * 10}`**
Yields one row as a dictionary. The async for loop on line 22 receives this value as `row`. After yielding, the generator is suspended until the `async for` loop requests the next item. The dictionary simulates a real database row; in production code this might be a named tuple, a dataclass, or an ORM model object.

**Line 20 — `async def main():`**
The root coroutine that exercises both the async context manager and the async generator in combination.

**Line 21 — `async with AsyncDatabaseConnection("db://prod") as conn:`**
Enters the async context manager. The event loop awaits `__aenter__`, which runs the simulated connection handshake. `conn` is bound to the return value of `__aenter__` (which is `self`, the connection object). The indented block that follows runs while the connection is open. When the block exits, `__aexit__` is awaited automatically.

**Line 22 — `async for row in conn.fetchRows(3):`**
Iterates the async generator. On each iteration, the event loop awaits `__anext__` on the generator, which runs the body of `fetchRows` up to the next `yield`. After each yield, the loop body runs (line 23), then the next iteration starts. The loop ends when the generator raises `StopAsyncIteration` — which happens automatically when the generator function returns.

**Line 23 — `print(f"Row: {row}")`**
Processes each row as it arrives. In a streaming context, this is where you would write to an output stream, aggregate statistics, or pass data to another pipeline stage — without ever holding the full result set in memory.

**Line 24 — `asyncio.run(main())`**
Runs the event loop. Expected output shows the connection message, three row prints interleaved with the event loop servicing other tasks (in a real concurrent program), and finally the disconnection message.
