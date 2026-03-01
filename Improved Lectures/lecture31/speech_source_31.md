# Speech Source — Lecture 31: Concurrency Architecture Patterns

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 30 we studied the failure modes of concurrent code: race conditions from uncoordinated shared access, and deadlock from circular lock dependencies. We established the four safety rules. Today we lift the view to architecture — the structural patterns for building concurrent systems that are both correct and scalable. We examine the three concurrency layers and when to use each, the fan-out/fan-in pattern for distributed queries, the circuit breaker pattern for async services, and the decision framework for choosing the right architecture for a given workload.

---

## CONCEPT 1.1 — The Three Concurrency Layers

**Problem it solves:**
Different workloads benefit from different concurrency mechanisms, and choosing the wrong mechanism produces either correctness bugs or performance regressions. The three layers correspond to three distinct problem types: (1) I/O-bound work where waiting dominates — async is the correct choice; (2) I/O-bound work that uses synchronous libraries that cannot be made async — a thread pool is the correct choice; (3) CPU-bound Python work that needs true parallelism — a process pool is the correct choice.

**Why invented:**
The three-layer model emerged from the practical observation that no single concurrency mechanism handles all three scenarios well. asyncio handles hundreds of thousands of I/O-bound operations on one thread but cannot run CPU-bound work without blocking the event loop. ThreadPoolExecutor handles synchronous library I/O concurrently but is GIL-limited for CPU work. ProcessPoolExecutor provides true parallelism for CPU work but has high IPC overhead for small tasks. The three-layer model uses each mechanism for its natural domain.

**What happens without it:**
A system that uses threads for CPU work hits the GIL ceiling — all threads share the interpreter and CPU-bound code is effectively single-threaded. A system that uses processes for I/O work pays the IPC cost on every request, even though a single-threaded event loop could handle the I/O at much higher throughput with lower overhead. A system that uses async for synchronous blocking library calls blocks the event loop, freezing all other concurrent work.

**Industry impact:**
The three-layer model is implemented explicitly in modern Python web frameworks. FastAPI uses asyncio for request routing, ThreadPoolExecutor for synchronous database driver calls, and optionally ProcessPoolExecutor or Ray for ML inference. This explicit layering prevents the accidental mixing of incompatible concurrency mechanisms.

---

## EXAMPLE 1.1 — Three-tier architecture: async gateway + thread pool + process pool

Narration: Eight simulated requests each require two steps: a legacy synchronous library call (handled in the thread pool) and CPU-intensive computation (handled in the process pool). The async handler orchestrates both steps using run_in_executor. First, it awaits the thread pool result (synchronous blocking I/O unblocked from the event loop). Then it awaits the process pool result (CPU work run in parallel). All eight requests run concurrently in the event loop, with thread pool and process pool executing in parallel. Total elapsed time is dominated by the longest request, not the sum.

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

---

## CONCEPT 2.1 — Fan-Out / Fan-In Pattern

**Problem it solves:**
Many distributed systems need to query multiple independent backends simultaneously — shards of a database, multiple microservices, replicas in different regions — and aggregate the results. Querying them sequentially produces latency that is the sum of all individual latencies. Querying them in parallel produces latency that is the maximum of the individual latencies (assuming enough concurrency). The fan-out/fan-in pattern: one dispatcher issues N concurrent queries (fan-out), then one aggregator collects and merges the N results (fan-in).

**Why invented:**
Fan-out/fan-in is the core pattern of distributed search engines (query N shards, merge results), distributed databases (read from N replicas, return first or quorum), microservice orchestration (call N services, wait for all or a subset), and scatter-gather messaging patterns. It is named for the circuit topology analogy: one signal fans out to multiple gates, then fans back into one.

**What happens without it:**
Without fan-out, each shard is queried in sequence. For 8 shards each with 200ms latency, sequential querying takes 1600ms. With fan-out, all 8 are queried simultaneously — total latency is 200ms plus coordination overhead. For N shards, fan-out provides an N× latency reduction, bounded by the slowest shard.

**Industry impact:**
Every major search engine — Elasticsearch, Solr, Algolia — uses fan-out/fan-in internally for multi-shard queries. Apache Spark's scatter-gather execution model is fan-out/fan-in at scale. Google's BigTable and Bigtable-based systems use it for cross-region reads. In Python, asyncio.gather or asyncio.wait with timeout enables fan-out with partial failure tolerance.

---

## CONCEPT 2.2 — Handling Slow Backends: asyncio.wait with Timeout

**Problem it solves:**
In fan-out queries, one slow backend should not hold up the entire operation. If 7 of 8 shards respond within 300ms and one takes 2 seconds, waiting for all 8 means the operation takes 2 seconds — dominated by the outlier. asyncio.wait with a timeout allows collecting results from all backends that responded within the deadline and cancelling the rest. The result is slightly incomplete (the slow shard's results are missing) but dramatically faster.

**Why invented:**
This pattern implements the "good-enough" principle: returning a partial result quickly is better than waiting indefinitely for a complete result. Web search engines use this explicitly: if a shard is slow, its results are omitted from the first page and may be added when the shard recovers. The timeout is the SLA parameter — the maximum acceptable latency at which you accept partial results.

**What happens without it:**
Without a timeout, a single slow or failed backend causes the entire fan-out to wait indefinitely or fail. In practice, a backend that has crashed or is paging due to memory pressure may take tens of seconds to respond or never respond. Without timeout-based cancellation, the fan-out hangs until the caller's connection times out externally.

**Industry impact:**
Tail latency management through partial result acceptance is a core technique described in Google's "Tail at Scale" paper (Dean and Barroso, 2013). The paper introduced the "hedged request" and "tied request" patterns that extend fan-out with speculative retries. In Python async code, asyncio.wait with timeout is the standard implementation.

---

## EXAMPLE 2.1 — Fan-out/fan-in with result aggregation and timeout

Narration: We query eight simulated shards simultaneously. Each shard has a random latency between 50ms and 400ms. We set a 300ms deadline using asyncio.wait with timeout=0.3. Shards that respond within 300ms contribute their results; the others are cancelled. The aggregated results are sorted and returned. The output shows how many shards responded versus timed out, and the total records collected from the responding shards.

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

---

## CONCEPT 3.1 — Circuit Breaker Pattern for Async Services

**Problem it solves:**
When a downstream service is failing — returning errors, timing out, or responding slowly — a naive client continues sending requests to it. Each failed request consumes resources (connection slots, timeout budget, retry logic). If the downstream service is severely degraded, the flood of retrying requests from all clients can make recovery harder. The circuit breaker pattern tracks failure rates and "opens the circuit" when failures exceed a threshold — subsequent requests fail fast without contacting the downstream service, giving it time to recover.

**Why invented:**
The circuit breaker pattern was popularized by Michael Nygard in "Release It!" (2007) and named for the electrical circuit breaker analogy: when current exceeds a safe threshold, the breaker opens and stops current flow, protecting the circuit from damage. In software, the "circuit" is the communication channel to a downstream service. The three states are: closed (normal operation), open (failing fast, not contacting downstream), and half-open (probing with a test request to check if recovery has occurred).

**What happens without it:**
A client without circuit breaking that calls a slow downstream service sends requests that time out after 5 seconds each. With 100 simultaneous clients, each call consumes a thread or a coroutine for 5 seconds. The 500 seconds of aggregate waiting capacity is consumed maintaining connections to a service that cannot respond. The client system exhausts its own connection pool and becomes unable to handle other requests — a cascading failure where one degraded downstream service takes down the entire client system.

**Industry impact:**
Circuit breakers are a foundational resilience pattern in microservice architectures. Netflix's Hystrix library (now Resilience4j) popularized circuit breakers in JVM systems. In Python, the `circuitbreaker` package and similar libraries provide decorator-based circuit breakers. The pattern is standard in any system where service calls can fail, timeout, or degrade — which is every production distributed system.

---

## CONCEPT 4.1 — Choosing Architecture: The Decision Framework

**Problem it solves:**
Given a new workload, which concurrency architecture is correct? The decision framework reduces to four questions. First: is the work I/O-bound or CPU-bound? I/O-bound work (network, disk, database) benefits from concurrency without parallelism. CPU-bound work (numerical computation, compression, parsing large files) requires true parallelism. Second: does the I/O work use async-compatible libraries? If yes, use asyncio. If no (legacy JDBC drivers, blocking HTTP clients), use a thread pool. Third: what is the scale? Thousands of simultaneous connections need async. Dozens of parallel CPU operations need a process pool. Fourth: what are the failure modes? Async services need circuit breakers and timeout handling. Process pools need result serialization and worker isolation.

**Why invented:**
The decision framework is a synthesis of accumulated practitioner experience — the trial-and-error of real systems discovering that the wrong concurrency model produces performance regressions that are invisible in development but catastrophic under production load.

**What happens without it:**
Without a systematic framework, developers default to the concurrency model they are most familiar with. A developer who knows threads well uses threads for everything — I/O work, CPU work, mixed workloads. GIL contention limits CPU parallelism. A developer who knows async well uses async for everything — but tries to run CPU work in coroutines, blocking the event loop. The framework prevents these category errors.

**Industry impact:**
The decision framework has been codified in Python's official documentation, in the async Python community's best practices guides, and in the internal architecture guides of major Python shops. It is the first topic covered in any senior Python engineer's concurrency review.

---

## CONCEPT 5.1 — Final Takeaway Lecture 31

The three concurrency layers: async for I/O with async-compatible libraries, thread pool for I/O with synchronous libraries, process pool for CPU-bound work. The fan-out/fan-in pattern: issue N concurrent queries with asyncio.gather or asyncio.wait, collect results, cancel timed-out tasks, aggregate. The circuit breaker: track failure rates, open the circuit when the threshold is exceeded, fast-fail during open state, probe during half-open state. The architecture decision framework: I/O vs. CPU; async vs. thread pool for I/O; process pool for CPU; scale determines resource sizing; failure modes determine resilience patterns.

In the next and final lecture of this module we turn to measurement: cProfile for function-level profiling, line_profiler for line-by-line analysis, timeit for microbenchmarking, and the profiling-first methodology that prevents architectural mistakes by making performance visible before they become production regressions.
