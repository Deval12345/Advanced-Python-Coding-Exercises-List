# Slides — Lecture 31: Concurrency Architecture Patterns

---

## Slide 1 — Lecture Overview

**From Tools to Patterns: Structuring Concurrent Systems**

- The three concurrency layers: async, thread pool, process pool
- Fan-out / fan-in: parallel dispatch and aggregation
- Tail latency management: asyncio.wait with timeout
- Circuit breaker pattern: resilience against failing dependencies
- The architecture decision framework: four questions for any workload

---

## Slide 2 — The Three Concurrency Layers

**Each kind of work has one correct home**

| Work Type | Library Type | Correct Tool |
|-----------|-------------|-------------|
| I/O-bound | Async-compatible | `asyncio` directly |
| I/O-bound | Synchronous/blocking | `ThreadPoolExecutor` + `run_in_executor` |
| CPU-bound | Any | `ProcessPoolExecutor` + `run_in_executor` |

- Wrong choice → performance regression or correctness bugs
- Async for CPU-bound work: freezes the event loop
- Threads for CPU-bound work: GIL prevents true parallelism
- Processes for I/O work: IPC overhead on every request — wasteful

---

## Slide 3 — Three-Tier Architecture in Practice

**Real production frameworks implement all three layers explicitly**

```
[Incoming Request]
       │
       ▼
[Async Event Loop]  ← asyncio: concurrent I/O routing
       │              hundreds of concurrent requests, one thread
    ┌──┴──┐
    │     │
    ▼     ▼
[Thread  [Process
 Pool]    Pool]
  ↑         ↑
Sync I/O  CPU work
blocking  parallel
library   computation
```

- FastAPI: asyncio for routing, ThreadPool for sync DB drivers, ProcessPool for ML inference
- Each layer is independently scaled based on its bottleneck

---

## Slide 4 — Example 31.1 — Three-Tier Request Handler

**Async gateway + thread pool + process pool — 8 concurrent requests**

```python
async def handleIncomingRequest(requestId, threadPool, processPool):
    loop = asyncio.get_event_loop()
    # Sync legacy library → thread pool (unblocks event loop)
    legacyResult = await loop.run_in_executor(
        threadPool, legacySyncLibraryCall, requestId)
    # CPU computation → process pool (true parallelism)
    cpuResult = await loop.run_in_executor(
        processPool, cpuHeavyWork, 30000 + requestId * 5000)
    return {"requestId": requestId, "legacy": legacyResult, "cpu": cpuResult}
```

- `run_in_executor(threadPool, ...)`: blocking call in thread, event loop continues
- `run_in_executor(processPool, ...)`: CPU work in separate process, event loop continues
- Both awaited sequentially within one coroutine — async orchestration

---

## Slide 5 — Fan-Out / Fan-In Pattern

**Parallel dispatch to N backends → aggregate N results**

```
           ┌─ Shard 0 ─┐
           ├─ Shard 1 ─┤
Dispatcher ├─ Shard 2 ─┤ → Aggregator → Unified Result
           ├─ Shard 3 ─┤
           └─ Shard N ─┘
```

- **Sequential query**: N × per-shard latency (e.g. 8 × 200ms = 1600ms)
- **Fan-out query**: max(per-shard latency) (e.g. 200ms)
- N-fold latency reduction — bounded by the slowest shard
- Used in: Elasticsearch (multi-shard queries), Spark (scatter-gather), BigQuery

---

## Slide 6 — Tail Latency: The Slow Backend Problem

**asyncio.wait with timeout: accept partial results at deadline**

```python
done, pending = await asyncio.wait(tasks, timeout=0.3)
# Collect results from shards that responded within 300ms
results = [task.result() for task in done]
# Cancel shards still running after deadline
for task in pending:
    task.cancel()
await asyncio.gather(*pending, return_exceptions=True)
```

- Timeout = SLA parameter: maximum acceptable response latency
- 1 slow shard → rest wait indefinitely (without timeout) vs. fast partial response (with)
- "Good enough": 7/8 shards in 300ms > 8/8 shards in 2000ms
- From Google's "Tail at Scale" paper (Dean & Barroso, 2013)

---

## Slide 7 — Example 31.2 — Fan-Out with Timeout

**8 simulated shards, 300ms deadline, partial result accepted**

- Each shard: random latency 50–400ms
- `asyncio.wait(tasks, timeout=0.3)` returns after 300ms
- `done` set: tasks that completed → extract results
- `pending` set: tasks still running → cancel them cleanly
- Output: shards responded / timed out, total records from partial result
- Key insight: response time is always 300ms, not bounded by the slowest shard

---

## Slide 8 — Circuit Breaker Pattern

**Prevent cascading failures from a degraded downstream service**

```
    ┌─────────────────────────────────────┐
    │           CLOSED (normal)           │
    │    failures < threshold → pass      │
    └──────────────┬──────────────────────┘
                   │ failures >= threshold
                   ▼
    ┌─────────────────────────────────────┐
    │     OPEN (fail-fast, protecting)    │
    │  all requests fail immediately      │◄─── probe failed
    └──────────────┬──────────────────────┘
                   │ reset interval elapsed
                   ▼
    ┌─────────────────────────────────────┐
    │  HALF-OPEN (probing for recovery)   │
    │  one test request allowed through   │─── probe succeeded ──► CLOSED
    └─────────────────────────────────────┘
```

- Origin: Michael Nygard, "Release It!" (2007) — electrical breaker analogy
- Prevents resource exhaustion from accumulating blocked requests
- Gives the downstream service time to recover without load

---

## Slide 9 — Why Circuit Breakers Are Required

**Without them, one slow dependency takes down the whole service**

Scenario: service calls downstream for every request; downstream degrades (5s timeout)

- 100 simultaneous requests → all blocked for 5s each
- Each holds: connection slot + thread/coroutine + memory
- New requests arrive → also block → resources exhausted
- Result: **cascading failure** — your service cannot handle ANY requests

With circuit breaker:
- After 5 failures → circuit opens
- Subsequent calls fail fast (< 1ms) — no downstream contact
- Resources freed immediately
- Downstream gets recovery time
- After reset: one probe → if OK, resume

---

## Slide 10 — Architecture Decision Framework

**Four questions for any new concurrent workload**

1. **I/O-bound or CPU-bound?**
   - I/O-bound → need concurrency (waiting)
   - CPU-bound → need parallelism (true parallel execution)

2. **For I/O: async-compatible libraries?**
   - Yes → use asyncio directly
   - No (blocking) → ThreadPoolExecutor + run_in_executor

3. **Scale?**
   - Thousands of connections → async (threads cannot scale to thousands)
   - Dozens of parallel CPU tasks → ProcessPoolExecutor (1 worker per CPU core)

4. **Failure modes?**
   - Async services → circuit breaker + timeout
   - Process pools → picklable args, worker crash recovery
   - Thread pools → lock contention under high load

---

## Slide 11 — Common Architectural Mistakes

**Each wrong choice has a predictable failure signature**

| Mistake | Symptom |
|---------|---------|
| Async for CPU-bound | Event loop freezes under load; response latency spikes |
| Threads for CPU-bound | CPU usage plateaus at 1 core; no scaling despite more threads |
| Processes for I/O | IPC overhead per request dominates; latency worse than single-threaded |
| Fan-out without timeout | One slow backend dominates response time for all clients |
| No circuit breaker | One degraded service causes cascading failure of caller |
| Random lock ordering | Intermittent deadlock under load; threads freeze without error |

---

## Slide 12 — Lecture 31 Key Principles

**What to carry forward into the project**

- Three layers: async for async I/O; thread pool for sync I/O; process pool for CPU
- Fan-out: `asyncio.gather` or `asyncio.wait` for parallel dispatch
- Fan-in with timeout: accept partial results at deadline, cancel slow backends
- Circuit breaker: three states (closed → open → half-open); fail fast when open
- Decision framework: I/O vs CPU → async vs thread pool → scale → failure modes
- Next: the big project begins — all these tools combined in a real architecture

---
