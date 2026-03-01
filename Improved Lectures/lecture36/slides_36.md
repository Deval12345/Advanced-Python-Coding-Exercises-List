# Slides — Lecture 36: Big Project Stage 3 — Concurrency Layer Part 2 (Process Pool for CPU Work)

---

## Slide 1: Two Kinds of Slowness

| Type | Cause | Example | Tool |
|---|---|---|---|
| I/O-bound | Waiting for external response | Sensor reads, HTTP, DB | asyncio (Lecture 35) |
| CPU-bound | Python bytecode executing | Normalization, statistics | ProcessPoolExecutor (Today) |

**You diagnosed the problem. Now you match the tool.**

---

## Slide 2: Why Asyncio Fails for CPU Work

**The event loop is cooperative — coroutines yield at `await` points.**

```python
async def badNormalize(inputStream):
    async for record in inputStream:
        # No await here — runs for 10ms
        # Event loop is FROZEN for 10ms
        result = heavyComputation(record)
        yield result
```

- 50,000 records × 10ms = **500s of event loop freeze**
- During freeze: no sensor reads, no queue drains, no heartbeats
- **The event loop cannot interrupt CPU work.**

---

## Slide 3: Why Thread Pools Don't Fully Solve It

**Python's Global Interpreter Lock (GIL):**

```
Thread 1: [=Python=][waiting for GIL][=Python=]
Thread 2: [waiting for GIL][=Python=][waiting for GIL]
```

- Only one thread runs Python bytecode at a time
- Threading gives **concurrency** (taking turns) — not **parallelism** (simultaneous)
- For CPU-bound work: 4 threads ≠ 4x speedup — you get ≈ 1x

**Processes have separate GILs. Processes give true parallelism.**

---

## Slide 4: ProcessPoolExecutor — True Parallelism

```python
with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
    future = pool.submit(processRecordBatch, batch)
    result = future.result()  # blocks until done
```

- Each worker: **separate Python interpreter, separate GIL**
- 4 workers on 4 cores: **4x speedup for CPU-bound work**
- Workers communicate via IPC (inter-process communication)
- **Rule: max_workers = number of physical CPU cores**

---

## Slide 5: Bridging Asyncio and ProcessPoolExecutor

```python
async def asyncPipelineWithProcessPool(records, batchSize):
    loop = asyncio.get_event_loop()
    
    with ProcessPoolExecutor(max_workers=4) as pool:
        batches = [records[i:i+batchSize]
                   for i in range(0, len(records), batchSize)]
        
        # Submit all batches — event loop continues running
        processedBatches = await asyncio.gather(*[
            loop.run_in_executor(pool, processRecordBatch, batch)
            for batch in batches
        ])
```

`run_in_executor` = submit to pool + return awaitable Future.

---

## Slide 6: The IPC Overhead Problem

**Each run_in_executor call:**
1. Pickle function arguments (serialize to bytes)
2. Send bytes to worker process via pipe
3. Worker unpickles, executes function
4. Worker pickles result, sends back
5. Main process unpickles result

**Fixed overhead per call: ~0.5-2ms regardless of data size.**

```
1 record/call × 50,000 calls × 1ms = 50 seconds of IPC overhead
```

---

## Slide 7: Batching Amortizes IPC Cost

**Batch 20 records per call:**

```python
batches = [records[i:i+batchSize] for i in range(0, len(records), batchSize)]
# 50,000 records / 20 per batch = 2,500 batches
# 2,500 calls × 1ms = 2.5 seconds IPC — 20x less
```

**The IPC overhead is paid once per batch, not per record.**

Batch size trade-offs:
- Too small → IPC overhead dominates
- Too large → fewer batches → workers idle
- Sweet spot: each batch takes 5-50ms to process

---

## Slide 8: Process Pool Constraints

**Functions submitted to pool must be:**
- Defined at module top level (not lambdas or nested functions)
- Picklable (standard Python types, dicts, lists — yes; sockets, file handles — no)

**Workers are stateless:**
- No open connections
- No shared memory
- Data in, results out — that is it

**This is a feature, not a limitation — stateless workers are easy to test and scale.**

---

## Slide 9: The processRecordBatch Function

```python
def processRecordBatch(records):
    results = []
    for record in records:
        value = record["value"]
        normalized = (value - 20.0) / 60.0
        rollingFeature = math.sqrt(abs(normalized)) * math.log(abs(value) + 1)
        results.append({
            **record,                              # copy existing fields
            "normalized": round(normalized, 4),
            "feature": round(rollingFeature, 4)
        })
    return results
```

Plain synchronous function. No async. No state. Input list → output list.

---

## Slide 10: Full Architecture — Both Layers

```
[Async Ingestion Layer]              [CPU Computation Layer]
                                     
AsyncSensorSource ──┐                ┌── Worker 0: batch[0..19]
AsyncSensorSource ──┤   asyncio      ├── Worker 1: batch[20..39]
AsyncSensorSource ──┤   .Queue  ──→  ├── Worker 2: batch[40..59]
AsyncSensorSource ──┘   (bridge)     └── Worker 3: batch[60..79]
                                     
Event loop (single thread)           ProcessPoolExecutor (4 processes)
I/O-bound — cooperative              CPU-bound — parallel
```

---

## Slide 11: When to Use Each Tool

**asyncio:** any `await`-able I/O — HTTP, database queries, sensor reads, file I/O
**ThreadPoolExecutor:** blocking I/O that cannot be made async (legacy libraries)
**ProcessPoolExecutor:** CPU-bound computation — normalization, statistics, ML inference

**Decision rule:**
1. Does it spend time waiting? → asyncio
2. Does it spend time computing Python code? → ProcessPoolExecutor
3. Does it call a blocking library that is not async-aware? → ThreadPoolExecutor

---

## Slide 12: Lecture 36 Summary

**The GIL problem:** threading cannot parallelize CPU-bound Python code.

**The solution:** `ProcessPoolExecutor` — separate processes, separate GILs, true parallelism.

**The bridge:** `loop.run_in_executor(pool, fn, args)` — submits to pool, returns awaitable.

**The batching requirement:** IPC overhead is fixed per call; batch N records to amortize it.

**Constraints:** pool functions must be module-level and picklable; workers are stateless.

**The architecture:** async event loop handles I/O, process pool handles CPU, asyncio.Queue bridges them.

**Next: Measuring what we built — profiling and per-stage metrics.**
