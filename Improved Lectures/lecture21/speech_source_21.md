# Speech Source — Lecture 21: Async/Await Fundamentals — Cooperative Concurrency with a Single Thread

---

## CONCEPT 0.1 — Transition from Previous Lecture

In our last lecture we covered threading: running multiple OS threads so that I/O operations overlap in real time. Threading works, but every thread consumes roughly one megabyte of stack space and the OS scheduler must juggle all of them. When connection counts climb into the tens of thousands, thread-per-connection stops scaling. That scalability wall is exactly what async/await was designed to break through: a single-threaded, cooperative concurrency model where coroutines voluntarily yield control at each I/O boundary, letting one event loop manage thousands of in-flight operations for the cost of a handful of OS threads.

---

## CONCEPT 1.1 — The Scalability Problem with Threads

**Problem it solves:** Thread-per-connection servers run out of memory and scheduler capacity long before they run out of network bandwidth. A thread blocked on a network read is consuming OS resources while doing nothing productive.

**Why invented:** Node.js demonstrated in 2009 that a single-threaded event loop could handle 50,000+ concurrent connections with far less memory. Python's asyncio (introduced in 3.4, native async/await syntax in 3.5) imports the same architectural idea into Python.

**What happens without it:** A web server handling 10,000 concurrent connections with one thread per connection needs 10 GB just for thread stacks. Context-switching overhead becomes measurable. The OS scheduler becomes a bottleneck, and latency climbs even when the CPU is largely idle.

**Industry impact:** Every major async Python framework — FastAPI, aiohttp, Starlette, Sanic — is built on this model. Async Python has become the default architecture for high-concurrency API servers, websocket servers, and data pipeline ingestion layers.

---

## CONCEPT 1.2 — Coroutines: Functions That Can Pause

**Problem it solves:** Normal functions run to completion without interruption. When one of them blocks on I/O, the whole thread blocks. Coroutines solve this by introducing a suspension point — the await expression — where the coroutine voluntarily hands control back to the event loop.

**Why invented:** Generator-based coroutines existed in Python before asyncio, but the syntax was awkward. The async def / await keywords introduced in Python 3.5 (PEP 492) gave coroutines first-class syntax that reads almost like ordinary sequential code.

**What happens without it:** Without coroutines you must use callbacks — a style that fragments sequential logic across many small functions and makes error handling notoriously difficult. Coroutines let you write I/O-concurrent code that reads top-to-bottom like synchronous code.

**Industry impact:** Python coroutines are the foundation of every async library in the ecosystem. Understanding the await suspension model is the prerequisite for using any of them correctly.

---

## EXAMPLE 1.1 — Basic coroutines and asyncio.gather

Narration: We define two coroutines — `fetchData` and `main`. Notice that calling `fetchData("A", 1.0)` does not start execution; it returns a coroutine object. Execution begins only when the event loop drives it. Inside `fetchData`, the line `await asyncio.sleep(delaySeconds)` is the suspension point. The event loop sees that await, notes "this coroutine is paused for one second," and immediately switches to the next ready coroutine. The `asyncio.gather` call in `main` schedules all three fetch coroutines simultaneously. Because they all suspend at their sleep lines, the event loop runs all three concurrently. The total elapsed time is roughly the longest individual delay — one second — rather than the sum of all delays, which would be 2.3 seconds.

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

---

## CONCEPT 2.1 — asyncio.gather, asyncio.create_task, and Task Scheduling

**Problem it solves:** Sometimes you want to fire off a coroutine and immediately continue doing other work, collecting the result later. asyncio.gather is convenient when you have a fixed list of coroutines and want all their results. create_task is better when you want to schedule work and continue immediately.

**Why invented:** The distinction mirrors the difference between launching a thread and waiting for it versus launching it and joining later. create_task gives the programmer explicit control over when to collect results.

**What happens without it:** Without explicit task creation, coroutines only run when awaited directly or passed to gather. You cannot easily implement patterns like "start ten downloads, do local processing, then collect whatever finished" without tasks.

**Industry impact:** Production async services use create_task constantly: health-check tasks, background metric flushers, periodic cleanup jobs — all scheduled as independent tasks that run alongside the main request-handling coroutines.

---

## EXAMPLE 2.1 — Tasks with create_task and cancellation

Narration: We create three tasks immediately with `create_task`. All three begin running in the background. Then `asyncio.wait` watches them with a timeout of 1.2 seconds. After 1.2 seconds, tasks B and C have finished (they sleep for 0.5 and 1.0 seconds), but task A (sleeping 2.0 seconds) is still pending. We cancel the pending tasks explicitly. The `CancelledError` is raised at the `await asyncio.sleep` line inside the task — that's the nearest suspension point, and it's where the event loop delivers cancellation. The task catches it, prints a message, and re-raises, which is the correct pattern: always re-raise CancelledError after cleanup.

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

---

## CONCEPT 3.1 — async for and async with

**Problem it solves:** Python's with statement and for loop are synchronous by design. When opening a database connection or iterating a streaming API response requires I/O at each step, the synchronous versions would block the event loop.

**Why invented:** PEP 492 and PEP 525 extended the context manager and iteration protocols with async versions. This lets you write `async with db_conn:` and `async for row in cursor:` — familiar syntax that now correctly suspends the coroutine at each I/O step rather than blocking.

**What happens without it:** Without async context managers you would have to manually call open/close coroutines and wrap them in try/finally — error-prone boilerplate. Without async iteration you cannot stream large result sets without loading everything into memory first.

**Industry impact:** Every async database driver (asyncpg, aiosqlite, motor for MongoDB), every async HTTP client (aiohttp, httpx), and every async file library uses these protocols. Async iteration is especially important for streaming: it lets you process gigabytes of data one chunk at a time with constant memory usage.

---

## EXAMPLE 3.1 — Async context manager and async generator

Narration: `AsyncDatabaseConnection` implements `__aenter__` and `__aexit__` as async methods. When we enter the `async with` block, the event loop awaits `__aenter__` — simulating an actual network handshake with a small sleep. The `fetchRows` method is an async generator: it uses `yield` inside an `async def`, and each iteration awaits a small delay simulating row-by-row network retrieval. The `async for` loop in `main` suspends at each `__anext__` call, giving the event loop an opportunity to run other coroutines between rows. When the `async with` block exits, `__aexit__` is awaited, cleanly closing the simulated connection.

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

---

## CONCEPT 4.1 — When to Use Async vs Threads

**Problem it solves:** Developers frequently choose the wrong concurrency model and pay the price in correctness bugs or performance degradation. Knowing the tradeoffs lets you make the right architectural choice from the start.

**Why invented:** Async was invented specifically for the high-concurrency, I/O-dominated workload. Threads were invented for true parallel execution. Both exist because no single model wins in every scenario.

**What happens without it:** Choosing threads for a 10,000-connection server causes memory exhaustion. Choosing async when you have synchronous-only database drivers causes subtle event-loop-blocking bugs that are difficult to diagnose because the server appears to slow down under load rather than crashing.

**Industry impact:** Modern Python services commonly combine both: async for the request-handling layer, with run_in_executor to offload synchronous library calls (legacy database drivers, encryption libraries, PDF generation) to a thread pool — getting the best of both worlds.

---

## CONCEPT 5.1 — Final Takeaway Lecture 21

Async/await provides cooperative concurrency in a single thread. `async def` defines a coroutine; `await` suspends it until the awaited operation completes. `asyncio.gather()` runs multiple coroutines concurrently; `asyncio.create_task()` schedules independent tasks that run alongside your current coroutine. `async with` and `async for` extend Python's familiar context manager and iteration protocols to the async world. The single-threaded model means no race conditions for shared variables between await points — but any blocking call freezes the entire event loop. Use async for high-concurrency I/O where you control the stack and can use async libraries. Use threads for simpler concurrent I/O against synchronous libraries or legacy code. In Lecture 22 we open the engine compartment and see how the event loop itself works.
