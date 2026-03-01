# Code — Lecture 24: High-Level Concurrency APIs — Controlling Async at Scale

---

## Example 24.1 — asyncio.Semaphore for rate limiting

```python
# Example 24.1
import asyncio                                         # line 1
import time                                            # line 2

MAX_CONCURRENT = 3                                     # line 3
semaphore = asyncio.Semaphore(MAX_CONCURRENT)          # line 4

async def fetchWithLimit(sessionId, delay):            # line 5
    async with semaphore:                              # line 6
        print(f"[{time.perf_counter():.2f}] Session {sessionId}: started")  # line 7
        await asyncio.sleep(delay)                     # line 8
        print(f"[{time.perf_counter():.2f}] Session {sessionId}: done")  # line 9
        return f"result_{sessionId}"                   # line 10

async def main():                                      # line 11
    tasks = [                                          # line 12
        fetchWithLimit(sessionId=i, delay=0.3)         # line 13
        for i in range(9)                              # line 14
    ]                                                  # line 15
    start = time.perf_counter()                        # line 16
    results = await asyncio.gather(*tasks)             # line 17
    elapsed = time.perf_counter() - start              # line 18
    print(f"All done in {elapsed:.2f}s")               # line 19
    print(f"Results: {results}")                       # line 20

asyncio.run(main())                                    # line 21
```

**Line-by-line explanation:**

- **Line 3:** `MAX_CONCURRENT = 3` — the resource ceiling; encode your API's stated rate limit here.
- **Line 4:** `asyncio.Semaphore(MAX_CONCURRENT)` — creates the semaphore with N=3 slots. Must be created at module level or inside an async context; creating it at module level before the event loop starts is fine in Python 3.10+.
- **Line 5:** `async def fetchWithLimit(sessionId, delay)` — the coroutine that represents one API session.
- **Line 6:** `async with semaphore:` — the gate. If fewer than 3 coroutines are inside, entry is immediate. If all 3 slots are occupied, this line suspends the coroutine and yields to the event loop until a slot opens.
- **Lines 7-9:** The work inside the critical section: print start, simulate the network call with `asyncio.sleep`, print completion.
- **Line 10:** Return the result. Exiting the `async with` block releases the semaphore slot automatically, waking the next waiting coroutine.
- **Lines 12-15:** Create nine coroutines. All nine are passed to `gather` at once — without the semaphore, all nine would start simultaneously.
- **Line 17:** `asyncio.gather(*tasks)` — starts all nine. The semaphore ensures only three run concurrently at any moment.
- **Lines 18-20:** Measure and print total elapsed time and all results.

**Expected output pattern:** Three sessions start at t≈0s, three more at t≈0.3s, three more at t≈0.6s. Total ≈ 0.9s with nine tasks, demonstrating three sequential batches of three.

---

## Example 24.2 — asyncio.wait with FIRST_COMPLETED race pattern

```python
# Example 24.2
import asyncio                                         # line 1
import time                                            # line 2
import random                                          # line 3

async def queryRegion(regionName, baseDelay):          # line 4
    jitter = random.uniform(0, 0.3)                    # line 5
    delay = baseDelay + jitter                         # line 6
    await asyncio.sleep(delay)                         # line 7
    return f"Response from {regionName} in {delay:.2f}s"  # line 8

async def main():                                      # line 9
    regions = [                                        # line 10
        asyncio.create_task(queryRegion("us-east", 0.2)),   # line 11
        asyncio.create_task(queryRegion("eu-west", 0.3)),   # line 12
        asyncio.create_task(queryRegion("ap-south", 0.4)),  # line 13
    ]                                                  # line 14

    done, pending = await asyncio.wait(                # line 15
        regions,                                       # line 16
        return_when=asyncio.FIRST_COMPLETED            # line 17
    )                                                  # line 18

    fastestTask = done.pop()                           # line 19
    print(f"Winner: {fastestTask.result()}")            # line 20

    for task in pending:                               # line 21
        task.cancel()                                  # line 22
    await asyncio.gather(*pending, return_exceptions=True)  # line 23
    print(f"Cancelled {len(pending)} slower tasks")    # line 24

asyncio.run(main())                                    # line 25
```

**Line-by-line explanation:**

- **Lines 4-8:** `queryRegion` simulates a geographic endpoint query. `random.uniform(0, 0.3)` adds jitter so the fastest region varies between runs — realistic simulation of network variability.
- **Lines 11-13:** `asyncio.create_task` is critical here, not bare coroutines. `asyncio.wait` requires Task objects; it will not accept raw coroutines. Each task starts running immediately in the background.
- **Lines 15-18:** `asyncio.wait` with `FIRST_COMPLETED`. The event loop runs all three tasks concurrently. The instant any one of them finishes, `wait` returns with that task in the `done` set and the other two in `pending`.
- **Line 19:** `done.pop()` retrieves the winning task. `done` is a set (unordered), but since we used `FIRST_COMPLETED` there is exactly one task in it.
- **Line 20:** `fastestTask.result()` returns the winner's result string.
- **Lines 21-23:** Cancel the remaining tasks. `task.cancel()` schedules `CancelledError` for delivery at the task's next `await`. The subsequent `asyncio.gather(*pending, return_exceptions=True)` allows the event loop to deliver those cancellations and run the tasks' cleanup code before exiting. Without this, the event loop logs "Task was destroyed but it is pending!"
- **Line 24:** Confirm how many slower tasks were cancelled.

---

## Example 24.3 — asyncio.wait_for with timeout and retry

```python
# Example 24.3
import asyncio                                         # line 1
import random                                          # line 2

async def unreliableApi(apiId):                        # line 3
    delay = random.uniform(0.1, 2.0)                   # line 4
    print(f"API {apiId}: will respond in {delay:.2f}s")  # line 5
    await asyncio.sleep(delay)                         # line 6
    return f"API {apiId} response"                     # line 7

async def callWithTimeout(apiId, timeoutSeconds, maxRetries):  # line 8
    for attempt in range(1, maxRetries + 1):           # line 9
        try:                                           # line 10
            result = await asyncio.wait_for(           # line 11
                unreliableApi(apiId),                  # line 12
                timeout=timeoutSeconds                 # line 13
            )                                          # line 14
            print(f"API {apiId} succeeded on attempt {attempt}")  # line 15
            return result                              # line 16
        except asyncio.TimeoutError:                   # line 17
            print(f"API {apiId} attempt {attempt} timed out")  # line 18
    return f"API {apiId} failed after {maxRetries} attempts"   # line 19

async def main():                                      # line 20
    results = await asyncio.gather(                    # line 21
        callWithTimeout("A", timeoutSeconds=0.5, maxRetries=3),  # line 22
        callWithTimeout("B", timeoutSeconds=0.5, maxRetries=3),  # line 23
        callWithTimeout("C", timeoutSeconds=0.5, maxRetries=3),  # line 24
    )                                                  # line 25
    for result in results:                             # line 26
        print(result)                                  # line 27

asyncio.run(main())                                    # line 28
```

**Line-by-line explanation:**

- **Lines 3-7:** `unreliableApi` simulates a real-world service with highly variable response times (0.1s to 2.0s). Four out of five calls will exceed the 0.5s timeout, making retries necessary.
- **Lines 8-19:** `callWithTimeout` is the resilient client pattern. It loops up to `maxRetries` times.
- **Line 9:** `for attempt in range(1, maxRetries + 1)` — one-indexed for human-readable logging.
- **Lines 10-14:** `asyncio.wait_for(unreliableApi(apiId), timeout=0.5)` — every call gets a fresh coroutine. Do not reuse the same coroutine across retries; once a coroutine raises, it cannot be resumed.
- **Line 17:** Catching `asyncio.TimeoutError` specifically — not a bare `Exception`. When `TimeoutError` fires, `wait_for` has already cancelled the underlying coroutine; you do not need to do anything else to clean it up.
- **Line 19:** If all retries fail, return a failure string rather than propagating an exception — this keeps `gather` from cancelling the other two callers.
- **Lines 20-27:** Three independent callers run concurrently. Each manages its own retry loop independently.

---

## Example 24.4 — Async producer-consumer with asyncio.Queue

```python
# Example 24.4
import asyncio                                         # line 1
import time                                            # line 2
import random                                          # line 3

async def dataProducer(queue, numItems):               # line 4
    for itemId in range(numItems):                     # line 5
        item = {"id": itemId, "value": random.randint(1, 100)}  # line 6
        await queue.put(item)                          # line 7
        print(f"Produced: {item}")                     # line 8
        await asyncio.sleep(0.05)                      # line 9
    await queue.put(None)                              # line 10  sentinel

async def dataConsumer(consumerId, queue, results):    # line 11
    while True:                                        # line 12
        item = await queue.get()                       # line 13
        if item is None:                               # line 14
            await queue.put(None)                      # line 15  re-put for other consumers
            break                                      # line 16
        processed = item["value"] * 3                  # line 17
        results.append({"id": item["id"], "result": processed})  # line 18
        print(f"Consumer {consumerId}: processed item {item['id']} → {processed}")  # line 19
        await asyncio.sleep(0.08)                      # line 20

async def main():                                      # line 21
    queue = asyncio.Queue(maxsize=5)                   # line 22
    results = []                                       # line 23
    await asyncio.gather(                              # line 24
        dataProducer(queue, numItems=10),              # line 25
        dataConsumer(1, queue, results),               # line 26
        dataConsumer(2, queue, results),               # line 27
    )                                                  # line 28
    print(f"Total processed: {len(results)}")          # line 29

asyncio.run(main())                                    # line 30
```

**Line-by-line explanation:**

- **Line 22:** `asyncio.Queue(maxsize=5)` — the backpressure valve. Without `maxsize`, the queue is unbounded and a fast producer can fill memory without limit.
- **Lines 4-9:** The producer generates one item every 0.05 seconds. `await queue.put(item)` blocks if the queue is full (at capacity 5), naturally throttling the producer to match consumer speed.
- **Line 10:** `await queue.put(None)` — the sentinel. One `None` for two consumers; the sentinel re-put pattern handles distribution to all consumers.
- **Lines 11-20:** Each consumer loops on `await queue.get()` — this suspends the coroutine until an item is available, yielding control to the event loop and other coroutines in the meantime.
- **Lines 14-16:** On receiving `None`, re-put it before breaking. This ensures the other consumer also receives the shutdown signal. If you have N consumers, the sentinel propagates automatically through the re-put chain.
- **Line 17:** Processing: multiply value by 3. Because asyncio is single-threaded, modifying the shared `results` list on line 18 is safe — no lock needed between two `await` points.
- **Line 20:** `await asyncio.sleep(0.08)` — consumer is slightly slower than producer (0.08s vs 0.05s), demonstrating backpressure: the queue will fill up and the producer will occasionally block at line 7.
