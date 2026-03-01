# Code — Lecture 31: Concurrency Architecture Patterns

---

## Example 31.1 — Three-Tier Architecture: Async Gateway + Thread Pool + Process Pool

```python
# Example 31.1
import asyncio
import concurrent.futures
import threading
import time
import math
import random

def legacySyncLibraryCall(dataId):
    time.sleep(random.uniform(0.05, 0.15))
    return f"legacy_result_{dataId}"

def cpuHeavyWork(inputValue):
    return sum(math.sqrt(i) for i in range(inputValue))

async def handleIncomingRequest(requestId, threadPool, processPool):
    loop = asyncio.get_event_loop()
    legacyResult = await loop.run_in_executor(threadPool, legacySyncLibraryCall, requestId)
    computeInput = 30000 + requestId * 5000
    cpuResult = await loop.run_in_executor(processPool, cpuHeavyWork, computeInput)
    return {
        "requestId": requestId,
        "legacyData": legacyResult,
        "cpuResult": round(cpuResult, 2)
    }

async def main():
    requestIds = list(range(8))
    with concurrent.futures.ThreadPoolExecutor(max_workers=8) as threadPool:
        with concurrent.futures.ProcessPoolExecutor(max_workers=4) as processPool:
            start = time.perf_counter()
            results = await asyncio.gather(*[
                handleIncomingRequest(rid, threadPool, processPool)
                for rid in requestIds
            ])
            elapsed = time.perf_counter() - start
    print(f"Handled {len(results)} requests in {elapsed:.2f}s")
    for r in results:
        print(f"  Request {r['requestId']}: legacy={r['legacyData']}, cpu={r['cpuResult']}")

if __name__ == "__main__":
    asyncio.run(main())
```

### Line-by-Line Explanation

**`def legacySyncLibraryCall(dataId):`**
A synchronous blocking function simulating a legacy library call — perhaps an old database driver or an HTTP client that does not support async. The `time.sleep(random.uniform(0.05, 0.15))` simulates I/O latency of 50–150 milliseconds. This function cannot be `await`ed directly; it must run in a thread pool.

**`def cpuHeavyWork(inputValue):`**
A CPU-bound function — pure computation with no I/O. `sum(math.sqrt(i) for i in range(inputValue))` computes a sum of square roots, burning CPU cycles without waiting. This function must run in a process pool, not a thread pool, because the GIL prevents true CPU parallelism across threads.

**`async def handleIncomingRequest(requestId, threadPool, processPool):`**
The async gateway handler. It receives two executor references — one for sync I/O work, one for CPU work — and orchestrates both. The function itself is a coroutine: it can be awaited, and while it is suspended at each `await`, the event loop can run other handlers.

**`loop = asyncio.get_event_loop()`**
Retrieves the running event loop. Required to call `run_in_executor`, which is a method on the loop, not on the coroutine.

**`legacyResult = await loop.run_in_executor(threadPool, legacySyncLibraryCall, requestId)`**
Submits `legacySyncLibraryCall(requestId)` to the thread pool. The event loop creates a Future wrapping the thread's result and suspends this coroutine at `await`. Other coroutines (other incoming requests) run while the thread executes the blocking call. When the thread finishes, the Future is resolved and this coroutine resumes with the result. The event loop is never blocked.

**`cpuResult = await loop.run_in_executor(processPool, cpuHeavyWork, computeInput)`**
Submits `cpuHeavyWork(computeInput)` to the process pool. The function and its argument are pickled, sent to a worker process, executed in parallel with other workers, and the pickled result is sent back. The event loop stays unblocked while the worker runs. Note: `computeInput` must be picklable (an integer is).

**`with concurrent.futures.ThreadPoolExecutor(max_workers=8) as threadPool:`**
Creates a thread pool with 8 workers. `max_workers=8` allows 8 simultaneous blocking calls — enough to handle all 8 requests simultaneously. The `with` block ensures all threads are shut down cleanly when the block exits, even on exception.

**`with concurrent.futures.ProcessPoolExecutor(max_workers=4) as processPool:`**
Creates a process pool with 4 worker processes — one per CPU core, the standard configuration for CPU-bound work. The nested `with` blocks ensure both pools are shut down in the correct order.

**`results = await asyncio.gather(*[handleIncomingRequest(rid, ...) for rid in requestIds])`**
Creates 8 coroutines (one per request) and runs them concurrently. `asyncio.gather` returns when all coroutines complete. All 8 requests are in flight simultaneously — the async gateway handles all 8 concurrently.

---

## Example 31.2 — Fan-Out / Fan-In with Result Aggregation and Timeout

```python
# Example 31.2
import asyncio
import time
import random

async def fetchFromShard(shardId, query):
    latency = random.uniform(0.05, 0.4)
    await asyncio.sleep(latency)
    numResults = random.randint(0, 10)
    return {
        "shardId": shardId,
        "results": [f"record_{shardId}_{i}" for i in range(numResults)],
        "latency": latency
    }

async def fanOutQuery(query, numShards):
    tasks = [asyncio.create_task(fetchFromShard(i, query)) for i in range(numShards)]
    done, pending = await asyncio.wait(tasks, timeout=0.3)
    results = []
    for task in done:
        results.append(task.result())
    for task in pending:
        task.cancel()
    await asyncio.gather(*pending, return_exceptions=True)
    allRecords = [rec for r in results for rec in r["results"]]
    allRecords.sort()
    return {
        "query": query,
        "shardsResponded": len(results),
        "shardsTimedOut": len(pending),
        "totalRecords": len(allRecords),
        "records": allRecords
    }

async def main():
    start = time.perf_counter()
    result = await fanOutQuery("SELECT * FROM events", numShards=8)
    elapsed = time.perf_counter() - start
    print(f"Fan-out query completed in {elapsed:.3f}s")
    print(f"  Shards responded: {result['shardsResponded']}/8")
    print(f"  Shards timed out: {result['shardsTimedOut']}/8")
    print(f"  Total records collected: {result['totalRecords']}")

asyncio.run(main())
```

### Line-by-Line Explanation

**`async def fetchFromShard(shardId, query):`**
Simulates querying one shard of a distributed database. `latency = random.uniform(0.05, 0.4)` gives each shard a random response time between 50ms and 400ms. Shards with latency below 300ms will succeed; those above 300ms will be cancelled.

**`await asyncio.sleep(latency)`**
Simulates the actual network round-trip to the shard. Using `asyncio.sleep` rather than `time.sleep` is essential: it yields control to the event loop, allowing other shard queries to run simultaneously. `time.sleep` would block the event loop and make the queries sequential.

**`tasks = [asyncio.create_task(fetchFromShard(i, query)) for i in range(numShards)]`**
Creates 8 tasks — one per shard — and immediately schedules them for execution. `create_task` differs from a bare coroutine: the task starts running on the event loop immediately, not only when explicitly awaited. All 8 shards begin their latency simulation simultaneously.

**`done, pending = await asyncio.wait(tasks, timeout=0.3)`**
Dispatches all 8 tasks simultaneously and waits up to 300ms. Returns two sets: `done` contains tasks that completed within 300ms; `pending` contains tasks that are still running when the timeout expires. Unlike `asyncio.gather`, `asyncio.wait` does not cancel pending tasks automatically — you must do that explicitly.

**`for task in done: results.append(task.result())`**
Extracts results from completed tasks. Calling `.result()` on a done task is safe — it returns the return value or re-raises any exception that occurred. Only do this for tasks in `done`; calling `.result()` on a pending task would block.

**`for task in pending: task.cancel()`**
Sends a cancellation signal to each pending task. The task receives a `CancelledError` at its next `await` point. Cancellation is cooperative — the task must be awaiting something to receive the cancellation.

**`await asyncio.gather(*pending, return_exceptions=True)`**
Waits for all cancelled tasks to finish cancelling. Without this, the tasks may continue running in the background after `fanOutQuery` returns. `return_exceptions=True` prevents `asyncio.gather` from raising `CancelledError` — the exceptions are returned as values instead, and we ignore them. This is the correct cleanup pattern for cancelled tasks.

**`allRecords = [rec for r in results for rec in r["results"]]`**
Flattens the nested list of results from all shards into a single list. This is the fan-in operation: N lists from N shards merged into one. The nested list comprehension first iterates over shard results, then over each record within a shard's results.

**`allRecords.sort()`**
Sorts the merged results. In a real search engine, sorting would be by relevance score; here it is alphabetical. The fan-in aggregation always includes a merge step — results from different shards need to be combined according to the query semantics.

---
