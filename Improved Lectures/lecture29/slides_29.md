# Slides — Lecture 29: Async + Multiprocessing Hybrid Architecture

---

## Slide 1 — Lecture Overview

**The Hybrid Architecture — I/O Concurrency + CPU Parallelism**

- Why neither async alone nor multiprocessing alone is sufficient
- loop.run_in_executor as the bridge between coroutines and process pools
- Architecture pattern: async front-end + ProcessPoolExecutor back-end
- Producer-consumer hybrid: async producer with asyncio.Queue + process consumer
- Common mistakes: blocking the event loop, sharing event loops across processes

---

## Slide 2 — The Dual Workload Problem

**I/O work and CPU work need different solutions**

- **I/O work** (read request, parse JSON, query cache, write response)
  - Most time is waiting — async handles thousands simultaneously in one thread
  - Event loop multiplexes waiting clients; no CPU needed during waits

- **CPU work** (model inference, feature computation, data transformation)
  - Never yields — freezes event loop for entire duration
  - 200ms inference = 200ms pause for ALL other clients waiting on event loop
  - Needs separate processes on separate CPU cores

- One thread, one event loop, one process pool — all three at once

---

## Slide 3 — loop.run_in_executor: The Bridge

**Submitting CPU work from a coroutine without blocking the event loop**

```
coroutine (event loop):
  result = await loop.run_in_executor(executor, cpuFunction, arg)
```

1. `run_in_executor` submits `cpuFunction(arg)` to the ProcessPoolExecutor
2. Returns immediately — coroutine is suspended, event loop continues
3. Event loop handles other coroutines while process worker runs
4. Worker completes → event loop resumes the awaiting coroutine with result
5. Coroutine receives result as if it were a local function call return

- No blocking. No freezing. No GIL held in main process during CPU work.

---

## Slide 4 — The Executor Lifecycle Rule

**Create once — share across all handlers — close at shutdown**

- Process pool startup: 50–200ms per worker process
- Creating a new `ProcessPoolExecutor` per request = startup cost per request
- Correct pattern: create executor ONCE in application startup
- Pass executor reference to all request handlers as argument
- Close executor in application shutdown (context manager handles this)

```python
# Correct: shared executor
async def main():
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = await asyncio.gather(*[
            handleRequest(rid, size, executor) for rid, size in requests
        ])
```

---

## Slide 5 — Example 29.1: Async Handler + ProcessPoolExecutor

**All requests dispatched concurrently — process pool schedules workers**

- 6 requests arrive simultaneously via `asyncio.gather`
- Each coroutine dispatches to process pool and suspends
- Event loop handles all 6 concurrently (all suspended, waiting for workers)
- Process pool: 4 workers run first 4 tasks in parallel; last 2 queue
- Total elapsed ≈ time of longest task batch, not sum of all tasks
- Key: `await loop.run_in_executor(executor, cpuIntensiveAnalysis, dataSize)`

---

## Slide 6 — Producer-Consumer Pattern

**Async I/O producer + process pool CPU consumer**

```
asyncJobProducer → asyncio.Queue(maxsize=10) → asyncJobSubmitter → ProcessPool
```

- Producer: generates jobs at I/O speed (async, with delays)
- Queue: buffer with backpressure — `maxsize` limits how far producer leads
- Submitter: reads from queue, dispatches each to executor, gathers results
- Process pool: computes jobs in parallel on multiple CPU cores
- Both producer and submitter run concurrently in same event loop

---

## Slide 7 — Backpressure with asyncio.Queue

**Why bounded queue size matters for memory safety**

- `asyncio.Queue(maxsize=10)` — producer blocks when queue has 10 items
- Without maxsize: producer can generate 10,000 jobs before workers process 1
- Each job carries data → unbounded queue → memory exhaustion
- With maxsize=10: producer waits when queue full → bounded memory usage
- `await jobQueue.put(item)` suspends producer if queue is full
- `await jobQueue.get()` suspends submitter if queue is empty
- Backpressure automatically balances production rate with consumption rate

---

## Slide 8 — Example 29.2: Producer-Consumer Hybrid

**Streaming pipeline with natural backpressure**

- Producer generates 12 jobs with `await asyncio.sleep(0.01)` between each
- Submitter reads from queue, dispatches to executor — no sleep, max throughput
- `asyncio.gather(producer, submitter)` — both run concurrently in event loop
- Poison-pill: producer puts `None` when done; submitter reads None → breaks loop
- After loop: `await asyncio.gather(*futures)` collects all process pool results
- Results collected via `results.extend(completedResults)` — thread-safe append

---

## Slide 9 — Common Mistake 1: Direct executor.submit in Coroutine

**Wrong tool — concurrent.futures.Future is not awaitable**

```python
# WRONG
future = executor.submit(cpuFunction, arg)
result = await future  # TypeError: object Future can't be used in await expression

# CORRECT
result = await loop.run_in_executor(executor, cpuFunction, arg)
```

- `executor.submit` returns `concurrent.futures.Future` — not asyncio-compatible
- `run_in_executor` wraps the Future in an asyncio Task — awaitable
- This is not a minor distinction — incorrect code freezes the event loop

---

## Slide 10 — Common Mistake 2: Sharing Event Loops Across Processes

**Event loops are process-local — coroutines cannot cross process boundaries**

- Each process running `asyncio.run()` has its own isolated event loop
- The main process's event loop is NOT accessible in worker processes
- Coroutines cannot be pickled and sent to a process pool
- Only regular, picklable functions can be submitted to ProcessPoolExecutor

**The boundary rule:**
```
Main process:     coroutines, event loop, I/O, await
─────────────── run_in_executor boundary ───────────────
Worker process:   synchronous functions, CPU computation
```
No coroutines cross this boundary. Ever.

---

## Slide 11 — Architecture Summary: The Three-Layer Model

**Async front-end + thread pool (optional) + process pool**

- **Layer 1:** Event loop + coroutines — I/O concurrency, request multiplexing
- **Layer 2 (optional):** ThreadPoolExecutor — blocking sync library calls
- **Layer 3:** ProcessPoolExecutor — CPU-intensive computation, GIL bypass

- `run_in_executor` bridges layers 1→2 and 1→3
- Executor created once at application startup; closed at shutdown
- Workers in Layer 3 must be regular functions — no coroutines, no event loops
- Next lecture (31) expands this into full concurrency architecture patterns

---

## Slide 12 — Lecture 29 Key Principles

**What to carry forward**

- Async for I/O concurrency; processes for CPU parallelism — both at once via hybrid
- `loop.run_in_executor(executor, fn, *args)` — the only correct bridge from coroutine to process pool
- Create `ProcessPoolExecutor` once per application — never per request
- `asyncio.Queue(maxsize=N)` provides backpressure in producer-consumer pipelines
- Only picklable synchronous functions can cross to worker processes — no coroutines
- Event loops are process-local — each process's loop is completely isolated
- Next: race conditions and deadlock — the failure modes of shared-state concurrency

---
