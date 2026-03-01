# Speech Source — Lecture 29: Async + Multiprocessing Hybrid Architecture

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 28 we solved the large-array sharing problem: shared memory, zero-copy NumPy views, and the partitioned-write pattern for parallel in-place transforms. Today we combine the two major concurrency paradigms we have studied — asyncio and multiprocessing — into a hybrid architecture. The motivation is that neither paradigm alone handles the full problem: async handles thousands of concurrent I/O connections efficiently but cannot bypass the GIL for CPU computation; processes bypass the GIL but cannot efficiently handle thousands of concurrent connections. The hybrid architecture uses async as the front-end request handler and a process pool as the back-end computation engine, bridged by loop.run_in_executor.

---

## CONCEPT 1.1 — Why You Need Both: The Problem Each Paradigm Solves

**Problem it solves:**
A web API that serves machine learning inference requests has two fundamentally different workloads. First, handling incoming connections: reading request bytes, parsing headers, deserializing JSON, writing response bytes. This is I/O — thousands of simultaneous clients spend most of their time waiting, and async's cooperative scheduling handles them with a single thread. Second, running the ML model on the request's input data: floating-point matrix multiplication across millions of parameters. This is CPU-bound — it releases no I/O and cannot yield to the event loop. A single thread running it blocks all other requests for the duration of the inference.

**Why invented:**
asyncio was designed for I/O concurrency: a single thread can manage thousands of concurrent connections because most of the time is spent waiting for network bytes, not executing Python. ProcessPoolExecutor was designed for CPU parallelism: separate processes bypass the GIL and run Python compute on separate CPU cores simultaneously. The hybrid architecture connects them: the event loop handles I/O concurrency while the process pool handles CPU parallelism, and loop.run_in_executor is the bridge that submits CPU work to the process pool from within a coroutine, awaiting the result without blocking the event loop.

**What happens without it:**
Without the hybrid architecture, you have two bad options. Option one: run inference synchronously in the async handler. The event loop blocks for the entire inference duration — 50 to 500 milliseconds — rejecting or delaying all other incoming requests. For 100 simultaneous clients, the tail latency grows linearly with inference time. Option two: use only processes without asyncio. You either fork one process per connection (expensive), use a thread pool (GIL-limited for CPU work), or use a process pool with synchronous request handling (no I/O concurrency).

**Industry impact:**
The async front-end + process pool back-end pattern is the architecture of production ML inference servers (Ray Serve, TorchServe, Triton), API gateways that perform per-request computation, real-time data enrichment services, and any web server that must combine I/O scalability with CPU-intensive processing. FastAPI uses this pattern implicitly when you combine async endpoints with background task submission.

---

## CONCEPT 2.1 — loop.run_in_executor with ProcessPoolExecutor as the Bridge

**Problem it solves:**
The critical technical problem is: how do you submit work to a process pool from inside a coroutine without blocking the event loop? The answer is loop.run_in_executor. This method takes an executor (ThreadPoolExecutor or ProcessPoolExecutor) and a callable with its arguments, submits the callable to the executor, wraps the resulting Future in an asyncio-compatible awaitable, and returns immediately to the event loop. The event loop continues handling other coroutines while the process pool worker runs. When the worker completes, the event loop resumes the awaiting coroutine with the result.

**Why invented:**
loop.run_in_executor was designed to be the universal bridge between asyncio and blocking or CPU-bound work. It handles three scenarios: (1) blocking I/O calls that cannot be made async — legacy database drivers, synchronous HTTP clients — run in a ThreadPoolExecutor, unblocking the event loop. (2) CPU-bound Python code that cannot yield to the event loop runs in a ProcessPoolExecutor, bypassing the GIL. (3) Calls to native libraries that release the GIL during computation can run in either executor type.

**What happens without it:**
Without run_in_executor, the only alternative for dispatching CPU work from a coroutine is asyncio.create_subprocess_exec, which creates a full child process per task — unacceptably expensive overhead. Or using asyncio.to_thread (Python 3.9+) for thread-based dispatch, but this is still GIL-limited for pure Python CPU work. run_in_executor with a ProcessPoolExecutor is the correct solution for CPU-bound work that must integrate with an async front-end.

**Industry impact:**
Every async Python web framework uses run_in_executor or an equivalent. FastAPI's BackgroundTasks, Starlette's run_in_threadpool, and Sanic's executor integration all use this pattern. The difference between thread and process executors is determined by whether the work is I/O-bound (thread executor) or CPU-bound Python code that cannot release the GIL (process executor).

---

## EXAMPLE 2.1 — Async request handler with ProcessPoolExecutor for CPU work

Narration: We simulate a web API that receives six requests with varying data sizes. Each request dispatches CPU-intensive analysis to a ProcessPoolExecutor. The coroutines run concurrently in the event loop, all dispatching work simultaneously. The process pool runs up to four workers in parallel. asyncio.gather collects all results. The key observation is that all six requests are dispatched nearly simultaneously — the event loop does not wait for one to complete before starting the next. The process pool schedules them in order of worker availability.

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

---

## CONCEPT 3.1 — Architecture Pattern: Async Front-End + Process Pool Back-End

**Problem it solves:**
Designing the hybrid architecture correctly requires understanding the lifecycle of a request as it moves through the system. A request arrives at the async handler — a coroutine. The coroutine performs I/O-bound preprocessing (read request body, parse JSON, validate, look up metadata from cache). It then dispatches the CPU-bound work to the process pool via run_in_executor and awaits. The event loop handles other incoming requests while the process pool worker runs. When the worker completes, the event loop resumes the coroutine, which performs I/O-bound postprocessing (serialize result, write response) and returns. The process pool is shared across all handlers — it is created once and used throughout the application's lifetime.

**Why invented:**
This architecture was developed as the natural complement to the reactor pattern in async networking. The reactor (the event loop) handles the multiplexing of many connections. When a connection needs non-I/O work, it submits the work to a pool and suspends. The pool provides parallelism. The combination provides both concurrency (many connections simultaneously) and parallelism (multiple CPU cores working simultaneously).

**What happens without it:**
A poorly structured hybrid architecture creates two common bugs. Bug one: creating a new ProcessPoolExecutor per request. Process pool startup costs 50-200ms per worker. Creating a new pool for each request adds that overhead to every single request. Bug two: sharing the event loop across processes. asyncio.run() in a worker process creates a new event loop for that process — the event loops are not shared and should not be. Each process runs its own independent Python interpreter.

**Industry impact:**
The shared executor pattern — creating the executor once in the application's main coroutine and passing it as an argument to request handlers — is a fundamental pattern in production async Python applications. FastAPI's lifespan events are the recommended way to create the executor at startup and close it at shutdown.

---

## CONCEPT 4.1 — Producer-Consumer Hybrid: Async Producer, Process Consumer

**Problem it solves:**
In pipeline architectures, data production is often I/O-bound — reading from a network stream, consuming from a message queue, generating inputs with random delays — while data consumption is CPU-bound — applying a model, computing statistics, transforming features. The producer-consumer pattern separates these concerns: an async producer generates jobs at I/O speed and puts them into an asyncio.Queue; an async submitter reads from the queue and dispatches each job to the process pool via run_in_executor; the results are collected with asyncio.gather.

**Why invented:**
asyncio.Queue provides backpressure through its maxsize parameter: if the queue is full, the producer awaits until the consumer (the submitter) takes an item. This prevents the producer from getting too far ahead of the consumer, bounding memory usage. The combination of async queue for backpressure plus process pool for CPU parallelism is a principled solution to the streaming compute problem.

**What happens without it:**
Without backpressure, a fast async producer can generate thousands of jobs before the process pool has processed any. Each job's data consumes memory while it waits in the queue. For large input data — arrays, documents, images — an unbounded queue exhausts available RAM before the workers make progress. The maxsize parameter on asyncio.Queue is the rate-limiting mechanism that keeps memory consumption bounded.

**Industry impact:**
The async producer + process pool consumer pattern is used in real-time data enrichment systems, event-driven ML pipelines, streaming feature engineering, and any system where input arrives at variable rate and must be processed with CPU-intensive work. Kafka consumer groups use a similar pattern: async consumption of Kafka messages with synchronous batch processing.

---

## EXAMPLE 4.1 — Producer-consumer hybrid: async producer, process consumer

Narration: An async producer generates 12 jobs, each with a list of 5,000 random integers. It puts each job into a bounded asyncio.Queue with maxsize 10, sleeping briefly between submissions to simulate I/O latency. An async submitter reads from the queue, dispatches each job to the ProcessPoolExecutor, collects all the resulting futures, and gathers them at the end. The process pool computes the sum of square roots for each job's values. The two async tasks run concurrently in the event loop, and asyncio.gather runs them both until completion.

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

---

## CONCEPT 5.1 — Common Mistakes: Blocking the Event Loop and Shared Event Loops

**Problem it solves:**
Two mistakes appear consistently in hybrid async+multiprocessing code. First: calling executor.submit() directly from a coroutine instead of using run_in_executor. executor.submit() returns a concurrent.futures.Future, not an asyncio-compatible awaitable. Trying to await it raises a TypeError. The correct pattern is always loop.run_in_executor, which wraps the Future in an asyncio Task. Second: attempting to share an event loop across the main process and worker processes. Each process runs its own Python interpreter with its own GIL; asyncio.run() in a worker creates a completely separate event loop, isolated from the main process's loop.

**Why invented:**
These constraints exist because asyncio's event loop is not thread-safe, is process-local, and has no serialization mechanism for transferring coroutines across process boundaries. A coroutine is a Python generator-based object; it cannot be pickled and sent to a worker process. Only regular functions — picklable callables — can be submitted to a ProcessPoolExecutor.

**What happens without it:**
A developer who tries to pass a coroutine to executor.submit gets a confusing error: the worker process runs the coroutine object as a function (not awaiting it), which is almost certainly not the intended behavior. A developer who tries to share the main process's event loop with a worker gets a different error: the event loop object cannot be pickled. Understanding these constraints is essential for writing correct hybrid code.

**Industry impact:**
These constraints drive the common architectural pattern: all asyncio code runs in the main process (or in a dedicated async worker process); all CPU-bound code runs in separate synchronous worker processes. The boundary between the two is loop.run_in_executor: async code above it, synchronous functions below it. No coroutines cross this boundary.

---

## CONCEPT 6.1 — Final Takeaway Lecture 29

The hybrid architecture separates concerns precisely: async for I/O concurrency (thousands of connections, single thread), processes for CPU parallelism (multiple cores, no GIL). loop.run_in_executor is the bridge: submits CPU work to a ProcessPoolExecutor from inside a coroutine, awaits the result without blocking the event loop, returns the result when the worker completes. Create the ProcessPoolExecutor once, share it across all handlers — never create a new executor per request. The async producer + process consumer pattern with asyncio.Queue provides backpressure to bound memory consumption. Only regular functions — not coroutines — can be submitted to a ProcessPoolExecutor. The event loop is process-local; it is not shared between the main process and worker processes.

In the next lecture we examine race conditions and deadlock in concurrent Python: the mechanics of data corruption without locks, threading.Lock as the solution, deadlock from inconsistent lock ordering, and threading.RLock and threading.Condition for more complex coordination patterns.
