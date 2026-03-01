# Code — Lecture 36: Big Project Stage 3 — Concurrency Layer Part 2 (Process Pool for CPU Work)

---

## Example 36.1 — Batch Processing with ProcessPoolExecutor

```python
import asyncio
import concurrent.futures
import time
import math
import random

def processRecordBatch(records):
    results = []
    for record in records:
        value = record["value"]
        normalized = (value - 20.0) / 60.0
        rollingFeature = math.sqrt(abs(normalized)) * math.log(abs(value) + 1)
        results.append({**record, "normalized": round(normalized, 4), "feature": round(rollingFeature, 4)})
    return results

async def asyncPipelineWithProcessPool(numSensors, numReadingsEach, batchSize=10):
    loop = asyncio.get_event_loop()
    records = [{"sensorId": f"S{i}", "timestamp": time.time() + i*0.001, "value": random.gauss(50, 15), "unit": "C"}
               for i in range(numSensors * numReadingsEach)]
    
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
        batches = [records[i:i+batchSize] for i in range(0, len(records), batchSize)]
        start = time.perf_counter()
        processedBatches = await asyncio.gather(*[
            loop.run_in_executor(pool, processRecordBatch, batch)
            for batch in batches
        ])
        elapsed = time.perf_counter() - start
    
    allResults = [r for batch in processedBatches for r in batch]
    print(f"Processed {len(allResults)} records in {elapsed:.3f}s ({len(allResults)/elapsed:.0f} rec/s)")
    return allResults

if __name__ == "__main__":
    results = asyncio.run(asyncPipelineWithProcessPool(numSensors=8, numReadingsEach=50, batchSize=20))
    print(f"Sample: {results[0]}")
```

**Line-by-line explanation:**

`def processRecordBatch(records):` — Defined at module top level. This is a requirement, not a style choice. Python's `multiprocessing` pickles function references by module path and function name. A function defined inside another function or as a lambda cannot be reliably pickled and will raise `AttributeError` or `PicklingError` when submitted to a process pool.

`results = []` — A local list that accumulates output records. Each worker process has its own memory space — this list is not shared with the main process or other workers.

`normalized = (value - 20.0) / 60.0` — Maps values in the expected range [20, 80] to approximately [0.0, 1.0]. Values outside this range produce values outside [0, 1]. In a real pipeline, you would clamp the result or handle out-of-range values explicitly.

`rollingFeature = math.sqrt(abs(normalized)) * math.log(abs(value) + 1)` — A non-linear feature transformation. `abs(normalized)` guards against negative values from the `sqrt`. `abs(value) + 1` guards against `log(0)`. This is the kind of computation that is genuinely CPU-bound — multiple floating-point operations per record, none of which involve waiting.

`results.append({**record, "normalized": round(normalized, 4), "feature": round(rollingFeature, 4)})` — Dict unpacking merges the original record fields with the new computed fields into a single dict. `**record` copies all key-value pairs from the input dict. The new keys `"normalized"` and `"feature"` are added. `round(..., 4)` limits precision to 4 decimal places for cleaner output.

`return results` — Returns the entire processed batch. This list is pickled by the worker and sent back to the main process via the IPC pipe.

`async def asyncPipelineWithProcessPool(numSensors, numReadingsEach, batchSize=10):` — An async function (not a generator) that orchestrates the process pool computation. It is async because it `await`s the `asyncio.gather` call.

`loop = asyncio.get_event_loop()` — Gets the currently running event loop. `run_in_executor` is a method on the loop object. In Python 3.10+, `asyncio.get_running_loop()` is preferred — it raises `RuntimeError` if called outside a running loop, which is safer than silently creating a new loop.

`records = [{"sensorId": f"S{i}", ..., "value": random.gauss(50, 15), "unit": "C"} for i in range(numSensors * numReadingsEach)]` — Generates `numSensors * numReadingsEach` synthetic records. `random.gauss(50, 15)` gives values centered at 50 with standard deviation 15 — a wider spread than the filter bounds [20, 80], so some values will be outside the expected range (testing the `abs()` guards in feature computation).

`with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:` — Creates 4 worker processes at the `with` statement, before any work is submitted. The `__exit__` method calls `pool.shutdown(wait=True)`, which signals all workers to exit and waits for them to terminate. This is the correct way to manage process pool lifetime — always use the context manager form.

`batches = [records[i:i+batchSize] for i in range(0, len(records), batchSize)]` — Creates slices of the records list. `range(0, len(records), batchSize)` produces 0, batchSize, 2*batchSize, ... up to but not exceeding `len(records)`. The last batch may have fewer than `batchSize` records — list slicing handles this gracefully (the slice stops at the end of the list).

`start = time.perf_counter()` — Start timing after the pool is created (pool creation is one-time overhead). We measure only the computation time.

`processedBatches = await asyncio.gather(*[loop.run_in_executor(pool, processRecordBatch, batch) for batch in batches])` — The core of the bridge pattern. For each batch:
1. `loop.run_in_executor(pool, processRecordBatch, batch)` — pickles `batch`, submits to pool, returns a Future immediately (does not block).
2. The list comprehension creates one Future per batch — all batches submitted in the same synchronous pass.
3. `asyncio.gather(*futures)` — schedules all futures concurrently and awaits all completions.
4. `await` suspends this coroutine until all batches are done. During this suspension, the event loop can run ingestion coroutines.
5. When all done, `processedBatches` is a list of lists — one list per batch.

`elapsed = time.perf_counter() - start` — Measured inside the `with` block, before the pool shuts down. This captures computation time plus pool shutdown overhead.

`allResults = [r for batch in processedBatches for r in batch]` — Two-level list comprehension. Outer loop: `for batch in processedBatches` iterates through the list of batches. Inner loop: `for r in batch` iterates through records in each batch. Result: a flat list of all processed records in the original order. (asyncio.gather preserves order — results arrive in the order futures were submitted, even if workers finish in a different order.)

---

## Example 36.2 — Benchmarking Sequential vs Batched Process Pool

```python
import asyncio
import concurrent.futures
import time
import random
import math

def heavyTransform(record):
    """Simulates expensive per-record computation."""
    v = record["value"]
    for _ in range(100):  # extra work to make CPU-bound
        v = math.sin(v) * math.cos(v) + math.log(abs(v) + 1)
    return {**record, "result": round(v, 6)}

def heavyTransformBatch(records):
    return [heavyTransform(r) for r in records]

async def benchmarkApproaches(records):
    loop = asyncio.get_running_loop()
    
    # Approach 1: sequential (synchronous)
    start = time.perf_counter()
    seqResults = [heavyTransform(r) for r in records]
    seqTime = time.perf_counter() - start
    print(f"Sequential: {len(seqResults)} records in {seqTime:.3f}s ({len(seqResults)/seqTime:.0f} rec/s)")
    
    # Approach 2: process pool, one record per call (bad batching)
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
        start = time.perf_counter()
        unbatchedResults = await asyncio.gather(*[
            loop.run_in_executor(pool, heavyTransform, r) for r in records[:200]  # fewer records to avoid timeout
        ])
        unbatchedTime = time.perf_counter() - start
        print(f"Pool (1/call, 200 records): {len(unbatchedResults)} records in {unbatchedTime:.3f}s "
              f"({len(unbatchedResults)/unbatchedTime:.0f} rec/s)")
    
    # Approach 3: process pool, batched (good batching)
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
        batchSize = 50
        batches = [records[i:i+batchSize] for i in range(0, len(records), batchSize)]
        start = time.perf_counter()
        batchedBatches = await asyncio.gather(*[
            loop.run_in_executor(pool, heavyTransformBatch, batch) for batch in batches
        ])
        batchedResults = [r for b in batchedBatches for r in b]
        batchedTime = time.perf_counter() - start
        print(f"Pool (batch={batchSize}): {len(batchedResults)} records in {batchedTime:.3f}s "
              f"({len(batchedResults)/batchedTime:.0f} rec/s)")

if __name__ == "__main__":
    records = [{"sensorId": f"S{i%5}", "value": random.gauss(50, 15), "unit": "C"}
               for i in range(1000)]
    asyncio.run(benchmarkApproaches(records))
```

**Line-by-line explanation:**

`for _ in range(100): v = math.sin(v) * math.cos(v) + math.log(abs(v) + 1)` — Artificially increases per-record CPU cost. 100 iterations of trigonometric and logarithmic operations make each record take significantly longer to process, amplifying the difference between sequential and parallel approaches and making the IPC overhead ratio more visible.

`seqResults = [heavyTransform(r) for r in records]` — Pure synchronous list comprehension. No IPC, no pooling. This is the baseline.

`unbatchedResults = await asyncio.gather(*[loop.run_in_executor(pool, heavyTransform, r) for r in records[:200]])` — One `run_in_executor` call per record. We limit to 200 records (instead of 1000) because with 200 separate IPC calls, this approach is already very slow — 1000 would take an unreasonably long time in a demo setting.

`batchSize = 50` — Each batch contains 50 records. 1000 records / 50 = 20 batches. With 4 workers, each worker handles 5 batches. This is a reasonable batch size for demonstrating the improvement.

`batchedBatches = await asyncio.gather(*[loop.run_in_executor(pool, heavyTransformBatch, batch) for batch in batches])` — 20 futures submitted, one per batch, all run concurrently across 4 workers.

`batchedResults = [r for b in batchedBatches for r in b]` — Flattens results. The nested comprehension reads as: "for each batch b in the list of batches, for each record r in that batch, yield r."
