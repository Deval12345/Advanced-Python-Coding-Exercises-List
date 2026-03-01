# Lecture 23: Futures and Executors — The Bridge Between Sync and Async
## slides_23.md

---

## Slide 1: The Problem With Callbacks

- Before Futures, concurrent code relied on callbacks: "when this finishes, call this function"
- Real programs have dozens of concurrent operations — callbacks nest inside callbacks
- Exception handling breaks across callback boundaries: no try/except spanning them
- Stack traces become meaningless; logic fragments across many functions
- The industry named this failure mode: "callback hell"
- Futures are the escape: a single object representing a result that doesn't exist yet

---

## Slide 2: What a Future Actually Is

- A Future is a proxy object for an in-flight computation's result
- You get a Future immediately on submission — before the task has even started
- Decouple when you START a task from when you NEED its result
- Submit 100 tasks, continue working, collect results exactly when needed

```python
with ThreadPoolExecutor(max_workers=4) as executor:
    future = executor.submit(someTask, arg)   # returns immediately
    # ... do other work here ...
    result = future.result()                  # blocks only here
```

- The Future travels through your code like any other object — clean, composable logic

---

## Slide 3: Future States and Key Methods

- States: PENDING → RUNNING → DONE (finished, cancelled, or stored exception)
- `future.result()` — blocks until done; returns value or re-raises stored exception
- `future.exception()` — returns the exception object without raising it
- `future.done()` — non-blocking check; returns True if task is finished or cancelled
- `future.cancel()` — requests cancellation; only succeeds if task is still PENDING

```python
future = executor.submit(riskyTask)
if future.done():
    try:
        value = future.result()
    except SomeError as e:
        err = future.exception()  # same error, not re-raised
```

- Exception propagation: concurrent errors look like sequential errors — try/except works normally

---

## Slide 4: ProcessPoolExecutor — True CPU Parallelism

- ThreadPoolExecutor: threads share one GIL — no real CPU parallelism for computation
- ProcessPoolExecutor: each process has its own GIL — runs on separate cores simultaneously
- API is identical — switching is ONE line of code change
- Arguments and return values must be pickled (serialized) for inter-process communication
- Rule: use when each task takes 100ms+ of CPU work; IPC overhead is then negligible

```python
# Thread pool (I/O-bound)
with ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(fetchUrl, urls))

# Process pool (CPU-bound) — same API!
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(computePrimes, workloads))
```

---

## Slide 5: The Critical ProcessPoolExecutor Guard

- On Windows and macOS, new processes are created by importing the main script
- Without the guard: spawned process imports script → tries to create its own pool → infinite spawn
- Result: program hangs or crashes the machine

```python
def computePrimeCount(limit):
    # CPU-heavy work here
    ...

if __name__ == "__main__":              # REQUIRED guard
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(computePrimeCount, workloads))
```

- Always wrap ProcessPoolExecutor usage in `if __name__ == "__main__":` — no exceptions
- Also: functions submitted to ProcessPoolExecutor must be picklable (defined at module level)

---

## Slide 6: Future Callbacks — Non-Blocking Reactions

- `future.add_done_callback(fn)` registers a function to call on completion (success or failure)
- The callback receives the future as its only argument
- If the future is already done when you register, the callback fires immediately
- Multiple callbacks can be registered on a single future — all will fire

```python
def onComplete(future):
    if future.exception():
        print(f"Failed: {future.exception()}")
    else:
        print(f"Result: {future.result()}")

future = executor.submit(riskyTask, taskId)
future.add_done_callback(onComplete)
# Execution continues immediately — callback fires later
```

- Enables reactive programming patterns: act the moment work is done, not when you poll

---

## Slide 7: as_completed() vs executor.map()

- `executor.map()` yields results in **submission order** — blocks on task 0 even if task 5 finishes first
- `concurrent.futures.as_completed()` yields futures in **completion order** — first done, first yielded
- For latency-sensitive pipelines, as_completed is the right tool

```python
# Submission order — may wait a long time for slow first task
for result in executor.map(task, items):
    process(result)

# Completion order — process each result the moment it's ready
futures = [executor.submit(task, item) for item in items]
for future in concurrent.futures.as_completed(futures):
    try:
        result = future.result()
    except Exception as e:
        handle_error(e)
```

---

## Slide 8: Error Handling Across Threads and Processes

- Worker exceptions are stored inside the Future, not lost or silently discarded
- `future.result()` re-raises them cleanly in the calling thread
- Exception handling from concurrent code uses ordinary try/except blocks
- Before Futures: exceptions from threads had to be caught manually and stored in shared queues

```python
for future in concurrent.futures.as_completed(futures):
    try:
        result = future.result()         # re-raises any worker exception
        successCount += 1
    except ValueError as e:
        failCount += 1
        print(f"Handled: {e}")           # same syntax as sequential code
```

- This normalization of error handling is one of the most important benefits of the Future abstraction

---

## Slide 9: Two Future Types — concurrent.futures vs asyncio

| Feature | concurrent.futures.Future | asyncio.Future |
|---|---|---|
| Lives in | Thread/process pool | Event loop |
| Block for result | `.result()` — blocks thread | `await future` — suspends coroutine |
| Set result | Automatically by executor | `future.set_result(value)` |
| Awaitable | No | Yes |
| Bridge | `asyncio.wrap_future()` | `loop.run_in_executor()` |

- `loop.run_in_executor(executor, fn, *args)` returns an `asyncio.Future` you can await
- The event loop stays unblocked while the thread runs the synchronous function

---

## Slide 10: The Async + Thread Pool Hybrid Pattern

- Production systems combine both worlds: event loop for I/O, thread pool for blocking calls
- Legacy synchronous libraries (PDF renderers, image processors, old ORMs) block the event loop if called directly
- Solution: run them in a thread pool via `run_in_executor` — event loop stays responsive

```python
async def asyncWrapper(executor, value):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        executor,
        legacyBlockingCompute,    # synchronous function
        value                     # its argument
    )
    return result

async def main():
    with ThreadPoolExecutor(max_workers=4) as executor:
        results = await asyncio.gather(
            asyncWrapper(executor, 10),
            asyncWrapper(executor, 20),
        )
```

- All four calls dispatch to threads simultaneously; total time is one task's time, not four

---

## Slide 11: Choosing the Right Executor

| Scenario | Use |
|---|---|
| Network requests, file I/O, database queries | `ThreadPoolExecutor` |
| CPU-heavy computation (100ms+ per task) | `ProcessPoolExecutor` |
| Blocking code called from async context | `loop.run_in_executor()` with `ThreadPoolExecutor` |
| Async-native I/O | No executor — use `asyncio` directly |

- When in doubt: profile first. IPC overhead can make ProcessPoolExecutor slower for tiny tasks.
- Always: use `if __name__ == "__main__":` guard with ProcessPoolExecutor.
- Always: define submitted functions at module level so they are picklable.

---

## Slide 12: Lecture 23 — Final Takeaways

- A Future is a proxy for a result that doesn't exist yet — decouples submission from collection
- `concurrent.futures.Future`: thread/process pool result carrier; `.result()` blocks; exceptions stored and re-raised
- `ProcessPoolExecutor`: same API as ThreadPoolExecutor; true CPU parallelism; requires pickling and the `if __name__ == "__main__"` guard
- `asyncio.Future`: awaitable; lives in event loop; `loop.run_in_executor()` bridges blocking code into async context
- `add_done_callback()`: non-blocking completion reactions registered before or after completion
- `as_completed()`: process results in arrival order, not submission order

**Next lecture:** asyncio.gather with exception policies, asyncio.wait with completion conditions, and asyncio.Semaphore for rate-limiting concurrency at scale.
