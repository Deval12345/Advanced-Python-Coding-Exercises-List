# Speech Source — Lecture 36: Big Project Stage 3 — Concurrency Layer Part 2 (Process Pool for CPU Work)

---

## CONCEPT 1.1 — Moving Heavy Computation to a Process Pool

**PROBLEM:**
The asyncio event loop solved sensor ingestion — many sensors, all waited on simultaneously, single thread. But now the pipeline needs to do real computation: normalize values against calibration baselines, compute rolling statistics over windows of 50 records, apply machine learning feature transformations. These operations are CPU-bound. They do not wait for I/O. They run Python bytecode on every record, burning CPU cycles. When a CPU-bound operation runs inside an asyncio coroutine, it does not `await` anything — it just runs. And while it runs, the event loop is frozen. No other coroutines run. No new sensor readings are accepted. The entire ingestion pipeline stalls for the duration of the computation. With 50,000 records and 10 microseconds of computation per record, that is 500 milliseconds of event loop freeze — 500 milliseconds where no sensors are read.

**WHY IT WAS INVENTED:**
Python's `concurrent.futures.ProcessPoolExecutor` was built for exactly this problem. It maintains a pool of worker processes — separate OS processes, each with their own Python interpreter and their own GIL (Global Interpreter Lock). CPU-bound work submitted to the pool runs in true parallelism: if you have 4 CPU cores and 4 workers, you get 4x the throughput for CPU-bound work. `loop.run_in_executor(pool, function, args)` bridges the asyncio world to the process pool: it submits the function to the pool, immediately returns a `Future` that can be `await`ed, and the event loop continues running other coroutines while the pool workers compute.

**WHAT HAPPENS WITHOUT IT:**
Without a process pool, you have two bad choices. First, run CPU computation inline in async coroutines — which freezes the event loop and destroys ingestion concurrency. Second, run CPU computation in a thread pool — which helps with the event loop freeze but cannot achieve true parallelism for CPU-bound work because of Python's GIL. A thread pool for CPU work runs your 4 threads on at most 1 CPU simultaneously. Only a process pool gives true parallel CPU execution in Python.

**INDUSTRY IMPACT:**
Process pools are standard infrastructure in Python data pipelines. Apache Spark's Python API (PySpark) submits computation to JVM workers via a bridge layer; the Python-side architecture mirrors this pattern. Dask's distributed computing framework uses process pools for CPU-bound tasks. NumPy operations release the GIL (they are written in C), but custom Python transformation logic does not. Scientific computing pipelines in genomics, astronomy, and financial modeling routinely use `ProcessPoolExecutor` or multiprocessing directly for parallel data transformation. Any real-time pipeline that does non-trivial computation per record — signal processing, anomaly detection, feature engineering — needs process-pool-based parallelism.

---

## CONCEPT 2.1 — Bridging Async Ingestion to Process Computation

**PROBLEM:**
We have two layers: an async ingestion layer (event loop) and a CPU computation layer (process pool). They need to work together without either blocking the other. Simply running the computation inline in the async layer blocks the event loop. Running the ingestion in the process pool workers is wasteful — each worker would need its own network connections, its own sensor state. The two layers are optimized for fundamentally different work patterns.

**WHY IT WAS INVENTED:**
`loop.run_in_executor(executor, func, *args)` is the bridge. It takes a synchronous function — exactly the kind that process pool workers run — and wraps it in an asyncio `Future`. The asyncio event loop submits the function to the executor, registers the Future, and continues running other coroutines. When the executor (process pool worker) finishes, it marks the Future as done, and the awaiting coroutine resumes. The critical property is that from the event loop's perspective, `await loop.run_in_executor(pool, func, args)` looks identical to `await asyncio.sleep(0.1)` — it is just another thing to wait for. The event loop stays unblocked.

**WHAT HAPPENS WITHOUT IT:**
Without `run_in_executor`, you are forced to run CPU work either synchronously (blocking), in a thread pool (GIL-limited), or in a separate process without asyncio integration (losing the ability to `await` the result). The bridge between async and sync execution is essential when your pipeline spans both I/O-bound and CPU-bound work, which is nearly every real data pipeline.

**INDUSTRY IMPACT:**
`run_in_executor` is the standard pattern in Python async services that mix I/O and CPU work. FastAPI's `run_in_executor` integration allows background CPU work without blocking request handling. Celery's async task support uses a similar bridge pattern. Machine learning inference servers (like those built on `triton` or custom `aiohttp` servers) use `run_in_executor` to submit inference work to GPU or CPU workers while the async layer handles request routing and response serialization.

---

**EXAMPLE 2.1 — Batch Processing with ProcessPoolExecutor**

```python
# Example 36.1
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

**NARRATION:**
`def processRecordBatch(records):` — This is the function that runs inside the process pool worker. It is a plain synchronous Python function — no async, no yields. Process pool workers are synchronous. The function receives a list of records (a batch) and returns a list of processed records. The batch is serialized (pickled), sent to the worker process, processed, and the result is pickled back. This serialization overhead is why we batch: sending 20 records in one IPC call is much cheaper than 20 separate IPC calls for 1 record each.

`normalized = (value - 20.0) / 60.0` — Linear normalization: maps the expected range [20, 80] to approximately [0, 1]. This is a CPU operation: arithmetic on floats.

`rollingFeature = math.sqrt(abs(normalized)) * math.log(abs(value) + 1)` — A feature transformation combining square root and logarithm. In a real ML pipeline, this might be a more complex feature extraction function. The point is that it is CPU-bound computation.

`results.append({**record, "normalized": ..., "feature": ...})` — Dict unpacking (`**record`) copies all existing fields into a new dict and adds the new computed fields. This is the standard immutable record pattern for pipelines.

`loop = asyncio.get_event_loop()` — Gets the currently running event loop. In Python 3.10+, prefer `asyncio.get_running_loop()` which raises an error if called outside a running loop (safer). We use `get_event_loop()` for compatibility.

`with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:` — Creates a pool of 4 worker processes. The `with` block ensures all workers are properly terminated when the block exits. `max_workers=4` is appropriate for a 4-core machine; for CPU-bound work, set it to the number of physical cores.

`batches = [records[i:i+batchSize] for i in range(0, len(records), batchSize)]` — Slice the records list into batches of `batchSize`. `range(0, len(records), batchSize)` generates start indices: 0, 20, 40, 60... Each `records[i:i+batchSize]` is one batch.

`processedBatches = await asyncio.gather(*[loop.run_in_executor(pool, processRecordBatch, batch) for batch in batches])` — For each batch, `run_in_executor(pool, processRecordBatch, batch)` creates a Future representing the worker's result. `asyncio.gather` runs all these futures concurrently — the event loop submits all batches to the pool simultaneously and waits for all results. This is the bridge: async event loop + process pool workers.

`allResults = [r for batch in processedBatches for r in batch]` — Flattens the list of lists (one per batch) into a single list. This is a nested list comprehension: for each batch in the results, for each record in that batch, collect the record.

---

## CONCEPT 3.1 — Designing for the Process Boundary: Batching for IPC Efficiency

**PROBLEM:**
Process pool workers communicate with the main process through inter-process communication — serialization (pickling) of arguments and return values. Each `run_in_executor` call pickles the function arguments, sends them to a worker via a pipe or socket, the worker unpickles and executes, then pickles the result and sends it back. For a simple dict record, pickling takes on the order of microseconds. If you call `run_in_executor` once per record, you pay that serialization cost per record. For 50,000 records, that is 50,000 pickle/unpickle cycles just for the IPC, completely dominating the actual computation time.

**WHY IT WAS INVENTED:**
Batching amortizes the fixed IPC overhead across many records. Instead of sending one record per call, send 20 records per call. The pickle overhead is paid once per batch. The worker processes all 20 records sequentially (or with further parallelism inside the worker, if the computation supports it). The results are pickled and returned as a batch. The ratio of IPC overhead to computation improves by a factor of the batch size. Batch size is a tunable parameter: larger batches reduce IPC overhead but reduce parallelism (fewer batches to distribute across workers) and increase memory usage.

**WHAT HAPPENS WITHOUT IT:**
Processing 50,000 records one at a time with `run_in_executor` can be slower than processing them synchronously in the main process, because the IPC overhead dominates. The process pool becomes a net negative — you pay the overhead of multiprocessing without getting the benefits. A rule of thumb: each `run_in_executor` call should do at least several milliseconds of work; otherwise, the IPC cost dominates. Batching is how you ensure each call meets that threshold.

**INDUSTRY IMPACT:**
Batching for IPC efficiency is universal in distributed computing. Apache Spark's micro-batch streaming processes records in configurable time windows. Kafka consumers process batches of messages, not individual messages. GPU inference engines (TensorRT, ONNX Runtime) require batched inputs because the GPU communication overhead is fixed per batch, not per record. Even within a single machine, batching for process pool workers is standard practice in Python data pipelines — `multiprocessing.Pool.map` applies exactly this pattern.
