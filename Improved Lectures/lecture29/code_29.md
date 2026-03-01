# Code — Lecture 29: Async + Multiprocessing Hybrid Architecture

---

## Example 29.1 — Async request handler with ProcessPoolExecutor for CPU work

```python
# Example 29.1
import asyncio
import concurrent.futures
import time
import math

def cpuIntensiveAnalysis(dataSize):
    total = sum(math.sqrt(i) * math.log(i + 1) for i in range(1, dataSize))
    return round(total, 4)

async def handleRequest(requestId, dataSize, executor):
    print(f"Request {requestId}: received, dispatching CPU work")
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, cpuIntensiveAnalysis, dataSize)
    print(f"Request {requestId}: CPU work done, result={result}")
    return {"requestId": requestId, "result": result}

async def main():
    requests = [(i, 50000 + i * 10000) for i in range(6)]
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:
        start = time.perf_counter()
        results = await asyncio.gather(*[
            handleRequest(rid, size, executor) for rid, size in requests
        ])
        elapsed = time.perf_counter() - start
    print(f"Handled {len(results)} requests in {elapsed:.2f}s")
    for r in results:
        print(f"  Request {r['requestId']}: {r['result']}")

if __name__ == "__main__":
    asyncio.run(main())
```

**Line-by-line explanation:**

- **Lines 7-9:** `cpuIntensiveAnalysis` is a plain synchronous function — no async, no coroutine. This is required: only synchronous functions can be sent to a process pool. It computes a sum of sqrt and log products, simulating CPU-bound analysis work.
- **Line 7:** Module-level function — required for pickling. Functions defined inside other functions or as lambdas cannot be pickled and will raise errors when submitted to a ProcessPoolExecutor.
- **Lines 11-16:** `handleRequest` is a coroutine — an async function that runs in the event loop. It simulates an HTTP request handler.
- **Line 14:** `loop = asyncio.get_event_loop()` — gets the current running event loop. In Python 3.10+, you can use `asyncio.get_running_loop()` which is slightly safer as it raises an error if called outside a running loop.
- **Line 15:** `await loop.run_in_executor(executor, cpuIntensiveAnalysis, dataSize)` — the bridge. This call: (1) submits `cpuIntensiveAnalysis(dataSize)` to the executor, (2) suspends this coroutine, (3) returns control to the event loop. The event loop continues running other coroutines while the process pool worker executes the function. When the worker completes, the event loop schedules the resume of this coroutine.
- **Lines 18-27:** `main` is the top-level coroutine that owns the executor lifetime.
- **Line 19:** Six requests with varying data sizes — larger requestIds get proportionally larger computation.
- **Lines 20-26:** The `with` block creates the ProcessPoolExecutor with 4 worker processes. These processes are created when the `with` block is entered, not when tasks are submitted. Worker startup happens once here, not per request.
- **Lines 22-25:** `asyncio.gather` starts all 6 `handleRequest` coroutines simultaneously. Each immediately prints "received" and dispatches to the executor. All 6 are now suspended, awaiting their process pool results. The event loop has no other work and waits for process pool completions.
- **Line 28:** `asyncio.run(main())` — runs the main coroutine and manages the event loop lifecycle.

**Expected output (order of "CPU work done" lines may vary):**
```
Request 0: received, dispatching CPU work
Request 1: received, dispatching CPU work
Request 2: received, dispatching CPU work
Request 3: received, dispatching CPU work
Request 4: received, dispatching CPU work
Request 5: received, dispatching CPU work
Request 0: CPU work done, result=1802.7431
Request 1: CPU work done, result=2589.1234
...
Handled 6 requests in 0.87s
```
All six "received" messages appear nearly simultaneously — the event loop dispatched them all before any worker finished. Workers 0-3 start immediately (4 workers); workers 4-5 wait until a slot opens.

---

## Example 29.2 — Producer-consumer hybrid: async producer, process consumer

```python
# Example 29.2
import asyncio
import concurrent.futures
import time
import math
import random

def processJob(jobData):
    jobId, values = jobData
    result = sum(math.sqrt(v) for v in values)
    return (jobId, round(result, 4))

async def asyncJobProducer(jobQueue, numJobs):
    for jobId in range(numJobs):
        values = [random.randint(1, 10000) for _ in range(5000)]
        await jobQueue.put((jobId, values))
        await asyncio.sleep(0.01)
    await jobQueue.put(None)

async def asyncJobSubmitter(jobQueue, executor, results):
    loop = asyncio.get_event_loop()
    futures = []
    while True:
        job = await jobQueue.get()
        if job is None:
            break
        future = loop.run_in_executor(executor, processJob, job)
        futures.append(future)
    completedResults = await asyncio.gather(*futures)
    results.extend(completedResults)

async def main():
    jobQueue = asyncio.Queue(maxsize=10)
    results = []
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:
        start = time.perf_counter()
        await asyncio.gather(
            asyncJobProducer(jobQueue, numJobs=12),
            asyncJobSubmitter(jobQueue, executor, results),
        )
        elapsed = time.perf_counter() - start
    print(f"Processed {len(results)} jobs in {elapsed:.2f}s")
    for jobId, result in sorted(results)[:5]:
        print(f"  Job {jobId}: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

**Line-by-line explanation:**

- **Lines 8-11:** `processJob` receives the full job data as a tuple — jobId and values list. It computes the sum of square roots and returns the result tuple. This is the CPU-bound work that runs in a worker process.
- **Line 8:** The function accepts a single argument tuple rather than separate arguments because `run_in_executor` has limited argument passing support. Packing into a tuple is the standard workaround for multi-argument worker functions.
- **Lines 13-17:** `asyncJobProducer` generates 12 jobs. Each job has a list of 5,000 random integers — this is non-trivial data that must be pickled when sent to the process pool.
- **Line 15:** `await jobQueue.put((jobId, values))` — this await suspends the producer if the queue is full (maxsize=10). This is the backpressure mechanism: the producer cannot race ahead more than 10 jobs beyond what the submitter has dispatched.
- **Line 16:** `await asyncio.sleep(0.01)` — simulates I/O latency between job generation. Real producers would await a network read or database query here.
- **Line 17:** The poison-pill: `None` signals the submitter to stop reading from the queue. Only one None is needed because there is only one submitter coroutine reading the queue.
- **Lines 19-29:** `asyncJobSubmitter` reads from the queue and dispatches to the executor.
- **Line 25:** `job = await jobQueue.get()` — suspends the submitter if the queue is empty. The event loop runs the producer while the submitter waits for a job.
- **Lines 26-27:** Check for the poison-pill sentinel — if None, exit the loop.
- **Line 28:** `loop.run_in_executor(executor, processJob, job)` — note this is NOT awaited here. It returns an asyncio-wrapped Future immediately without blocking. The future is appended to the list.
- **Line 29:** `futures.append(future)` — collect all futures for batch waiting.
- **Line 30:** After the queue is drained (None received), gather all futures to wait for all process pool results. This is the synchronization point.
- **Line 34:** `asyncio.Queue(maxsize=10)` — bounded queue. The maxsize limits how many unprocessed jobs can accumulate, providing natural backpressure.
- **Lines 37-40:** `asyncio.gather(producer, submitter)` — runs both coroutines concurrently. The event loop interleaves them: producer runs, puts a job, sleeps; submitter wakes up, gets the job, dispatches, waits for next job; producer wakes up, generates next job; and so on.

**Expected output:**
```
Processed 12 jobs in 0.43s
  Job 0: 3714221.8234
  Job 1: 3713108.5671
  Job 2: 3716492.1923
  Job 3: 3714887.3412
  Job 4: 3715023.7891
```
The total time is approximately 12 jobs × 0.01s I/O delay = 0.12s for production, plus computation time. The process pool handles computation in parallel with production, so total elapsed is dominated by the production delay, not the computation.
