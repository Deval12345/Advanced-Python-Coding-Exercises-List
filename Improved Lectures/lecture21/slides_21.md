# Slides — Lecture 21: Async/Await Fundamentals — Cooperative Concurrency with a Single Thread

---

## Slide 1: The Scalability Problem with Threads

- Each OS thread consumes ~1 MB of stack space regardless of whether it is doing useful work
- A server with 10,000 concurrent connections needs ~10 GB just for thread stacks
- The OS scheduler must context-switch between all threads, adding measurable CPU overhead
- Latency climbs under high connection counts even when the CPU is largely idle
- Network bandwidth is rarely the bottleneck — thread overhead is
- The thread-per-connection model hits a hard memory wall before it hits a hardware limit

---

## Slide 2: The Async/Await Solution — Cooperative Concurrency

- A single event loop manages thousands of in-flight operations using only a handful of OS threads
- Node.js (2009) demonstrated 50,000+ concurrent connections on a single-threaded loop
- Python's asyncio introduced in 3.4; native `async`/`await` syntax in Python 3.5 (PEP 492)
- Coroutines voluntarily yield control at each I/O boundary — called cooperative concurrency
- The OS never forcibly interrupts a coroutine; it suspends itself
- FastAPI, aiohttp, Starlette, and Sanic are all built on this model

---

## Slide 3: Coroutines — Functions That Can Pause

- Defined with `async def`; look like regular functions but can suspend at `await` expressions
- Calling a coroutine function does NOT start execution — it returns a coroutine object
- Execution begins only when the event loop drives the coroutine
- `await expr` suspends the current coroutine and yields control to the event loop
- The event loop resumes the coroutine when the awaited operation completes
- Replaces callback-based async code with readable top-to-bottom sequential logic

```python
async def fetchData(sourceId, delaySeconds):
    print(f"Starting fetch {sourceId}")
    await asyncio.sleep(delaySeconds)   # suspend here; run other coroutines
    return f"Data from source {sourceId}"
```

---

## Slide 4: asyncio.gather — Concurrent Fan-Out

- Schedules multiple coroutines to run concurrently and collects all their return values
- All coroutines begin immediately; each suspends independently at its own `await` points
- Total elapsed time ≈ the longest individual coroutine, not the sum of all delays
- Returns results in the same order as the input coroutines, regardless of completion order
- Raises the first exception by default; use `return_exceptions=True` to collect all errors
- Best used when you have a fixed, known set of coroutines and need all their results

```python
results = await asyncio.gather(
    fetchData("A", 1.0),
    fetchData("B", 0.5),
    fetchData("C", 0.8),
)
# Completes in ~1.0s, not ~2.3s
```

---

## Slide 5: asyncio.create_task — Background Task Scheduling

- Wraps a coroutine in a `Task` and schedules it to run immediately on the event loop
- Returns a `Task` object that can be awaited, inspected, or cancelled later
- Unlike `gather`, the current coroutine continues running without waiting for the task
- `asyncio.wait([task1, task2], timeout=N)` returns `(done_set, pending_set)` after the timeout
- Pending tasks that are no longer needed must be cancelled explicitly
- Used in production for health checks, background metric flushers, and periodic cleanup jobs

```python
task1 = asyncio.create_task(longRunningJob("A", 2.0))
task2 = asyncio.create_task(longRunningJob("B", 0.5))
done, pending = await asyncio.wait([task1, task2], timeout=1.2)
```

---

## Slide 6: Task Cancellation — The Correct Pattern

- Calling `task.cancel()` delivers a `CancelledError` at the task's nearest `await` point
- The `await` inside the task is where the exception is injected by the event loop
- Always catch `CancelledError`, perform cleanup, then re-raise — never swallow it
- Swallowing `CancelledError` leaves the event loop unable to confirm the task stopped
- After cancelling pending tasks, await them with `return_exceptions=True` to drain them safely
- Cancellation is the clean shutdown mechanism for long-running background tasks

```python
except asyncio.CancelledError:
    print(f"Job {jobId} was cancelled")
    raise   # always re-raise
```

---

## Slide 7: async with — Async Context Managers

- The standard `with` statement is synchronous; setup and teardown cannot suspend the coroutine
- `async with` allows `__aenter__` and `__aexit__` to be `async def` methods that can `await`
- Entering the block awaits `__aenter__`; exiting awaits `__aexit__` — even on exceptions
- Eliminates the need for manual `try/finally` around async open/close operations
- Used by every async database driver, HTTP client, and file library in the ecosystem
- Guarantees resource cleanup without blocking the event loop

```python
class AsyncDatabaseConnection:
    async def __aenter__(self): ...
    async def __aexit__(self, exc_type, exc_val, exc_tb): ...

async with AsyncDatabaseConnection("db://prod") as conn:
    ...
```

---

## Slide 8: async for — Async Iteration

- The standard `for` loop calls `__next__` synchronously; it blocks if each step requires I/O
- `async for` calls `__anext__` which is an `async def` method; the loop suspends at each step
- An async generator uses `yield` inside an `async def` to produce values one at a time
- Between every iteration, the event loop gets an opportunity to run other coroutines
- Essential for streaming: process gigabytes of data chunk-by-chunk with constant memory usage
- Used by async database cursors, async HTTP response streaming, and message queue consumers

```python
async def fetchRows(self, count):       # async generator
    for i in range(count):
        await asyncio.sleep(0.005)      # suspend; event loop can run elsewhere
        yield {"id": i, "value": i * 10}

async for row in conn.fetchRows(3):
    print(f"Row: {row}")
```

---

## Slide 9: The Single-Threaded Safety Guarantee

- Between two `await` points, a coroutine runs atomically — no other coroutine can interrupt it
- Shared in-memory state mutated between awaits is safe from race conditions by design
- This eliminates the need for locks around pure in-memory operations
- However: any blocking call (synchronous I/O, CPU computation) freezes the entire event loop
- Blocking the event loop stalls every other coroutine — the server appears to slow under load
- All libraries in the async stack must be async-compatible; one synchronous call breaks the model

---

## Slide 10: When to Use Async vs Threads

| Scenario | Use Async | Use Threads |
|---|---|---|
| 10,000+ concurrent connections | Yes | No — memory exhaustion |
| Async-compatible library ecosystem | Yes | Not needed |
| Synchronous-only legacy libraries | No | Yes — wrap with run_in_executor |
| CPU-bound work | No | Yes (or multiprocessing) |
| Simple concurrent I/O, few connections | Either | Fine |
| Mixed: async request layer + sync libraries | Combine | run_in_executor for sync calls |

- Modern production services combine both: async for request handling, thread pool for sync calls
- `loop.run_in_executor(None, sync_function, args)` offloads synchronous work to a thread pool

---

## Slide 11: Key Takeaways — Lecture 21

- `async def` defines a coroutine; calling it returns a coroutine object, not a result
- `await expr` is the suspension point — the event loop runs other coroutines while waiting
- `asyncio.gather()` runs a fixed set of coroutines concurrently; total time = max, not sum
- `asyncio.create_task()` schedules a background task and returns immediately
- `async with` and `async for` bring familiar Python patterns into the async model
- Always re-raise `CancelledError` after cleanup — never swallow it
- One blocking call can freeze the entire event loop; keep the async stack consistent
- Next lecture: how the event loop works internally
