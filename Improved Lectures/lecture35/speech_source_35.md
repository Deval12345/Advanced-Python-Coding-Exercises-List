# Speech Source — Lecture 35: Big Project Stage 3 — Concurrency Layer Part 1 (Async Ingestion)

---

## CONCEPT 1.1 — Why Concurrency: The Sequential Ingestion Bottleneck

**PROBLEM:**
Your pipeline reads from 10 temperature sensors. Each sensor requires a network request that takes 100 milliseconds to complete. If you read them sequentially — sensor 1, wait 100ms, sensor 2, wait 100ms, sensor 3, wait 100ms — the total time per full reading cycle is 1,000 milliseconds: one full second. This means each sensor is effectively sampled at 1Hz divided by 10 sensors — 0.1Hz. You wanted 1Hz per sensor; you got 0.1Hz. Your sampling rate is ten times slower than required, and the gap grows linearly with every sensor you add. 20 sensors: 0.05Hz. 100 sensors: 0.01Hz.

**WHY IT WAS INVENTED:**
The insight behind asyncio — and asynchronous programming in general — is that waiting for I/O does not require the CPU. When you send a network request to a sensor and wait for the response, your Python thread is idle. It is doing nothing. It could be sending nine other requests simultaneously and waiting for all ten responses at once. The event loop is the mechanism that makes this possible: a single thread that manages many pending I/O operations by switching between them whenever one is waiting. `asyncio` brings this model to Python through `async def` functions, `await` for suspension points, and `async for` for asynchronous iteration.

**WHAT HAPPENS WITHOUT IT:**
Without concurrency, scaling a data pipeline to many sensors requires either running one OS thread per sensor — which is expensive in memory (each thread has its own stack, typically 8MB) and in scheduling overhead — or running one process per sensor. With 100 sensors, 100 threads means 800MB of stack memory and constant context switching. Asyncio achieves the same concurrency with a single thread and a single event loop, with no per-sensor memory overhead beyond the coroutine frame itself.

**INDUSTRY IMPACT:**
Asyncio is the foundation of Python's modern I/O-intensive infrastructure. FastAPI — the most widely adopted Python web framework — is built on asyncio. AIOHTTP, the asynchronous HTTP client, handles hundreds of concurrent requests on a single thread. Kafka consumer clients, Redis clients (aioredis), PostgreSQL clients (asyncpg) — the entire modern Python async ecosystem was built to solve exactly the problem of waiting for many I/O operations simultaneously. For sensor networks, IoT platforms, and real-time analytics, async ingestion is not an optimization — it is the minimum viable approach.

---

## CONCEPT 2.1 — The AsyncSource Protocol: Async Generators

**PROBLEM:**
The synchronous `Source` protocol uses `stream()` as a regular generator — `def stream(self): yield ...`. This works when the data is already in memory or available on disk. But when data comes from a network sensor that must be polled with an HTTP request, `yield`ing inside a regular generator means the entire thread blocks while waiting for the network response. During that wait, no other code runs. The event loop cannot service other sensors. The async/await machinery is completely bypassed.

**WHY IT WAS INVENTED:**
Python 3.6 introduced `async def` combined with `yield` — the asynchronous generator. An `async def` function that contains `yield` is an async generator. It can be iterated with `async for`, and within it, you can `await` coroutines — which hands control back to the event loop while waiting, allowing other async operations to proceed simultaneously. The `AsyncSource` protocol replaces `def stream(self): yield ...` with `async def stream(self): yield ...`. The shape is identical; the execution model is fundamentally different.

**WHAT HAPPENS WITHOUT IT:**
Without async generators, you would need to separate the "produce data" step from the "send data somewhere" step using explicit callbacks or queues — the callback-based style that asyncio was designed to replace. Async generators allow the same lazy, composable pipeline structure as synchronous generators, but with the cooperative multitasking of the event loop embedded at every `await` point.

**INDUSTRY IMPACT:**
Async generators are used throughout modern Python streaming systems. Kafka's `aiokafka` library uses them to represent topic partitions as streams. WebSocket server libraries like `websockets` use async generators for message streams. gRPC's Python async API uses them for server streaming and bidirectional streaming RPCs. Any system that needs lazy, backpressure-aware streaming over I/O naturally gravitates toward async generators.

---

**EXAMPLE 2.1 — AsyncSensorSource with Async Generator**

```python
# Example 35.1
import asyncio
import random
import time

class AsyncSensorSource:
    def __init__(self, sensorId, readIntervalSec, numReadings=None):
        self.sensorId = sensorId
        self.readIntervalSec = readIntervalSec
        self.numReadings = numReadings
    
    async def stream(self):
        count = 0
        while self.numReadings is None or count < self.numReadings:
            await asyncio.sleep(self.readIntervalSec)
            yield {
                "sensorId": self.sensorId,
                "timestamp": time.time(),
                "value": random.gauss(50.0, 10.0),
                "unit": "celsius"
            }
            count += 1

async def ingestAllSensors(sensors, outputQueue):
    async def ingestOne(sensor):
        async for record in sensor.stream():
            await outputQueue.put(record)
    
    await asyncio.gather(*[ingestOne(s) for s in sensors])
    await outputQueue.put(None)

async def main():
    sensors = [AsyncSensorSource(f"TEMP_{i:02d}", readIntervalSec=0.05, numReadings=5) for i in range(4)]
    queue = asyncio.Queue(maxsize=20)
    start = time.perf_counter()
    await asyncio.gather(
        ingestAllSensors(sensors, queue),
    )
    records = []
    while not queue.empty():
        item = queue.get_nowait()
        if item is None: break
        records.append(item)
    elapsed = time.perf_counter() - start
    print(f"Ingested {len(records)} records in {elapsed:.2f}s from {len(sensors)} sensors")
    for r in records[:5]:
        print(f"  {r['sensorId']}: {r['value']:.2f} at {r['timestamp']:.3f}")

asyncio.run(main())
```

**NARRATION:**
`class AsyncSensorSource:` — The async counterpart to the synchronous `SensorSource` from Lecture 33. Same fields, same purpose — represents a single physical sensor. The difference is entirely in `stream()`.

`async def stream(self):` — The `async` keyword makes this a coroutine function. Combined with `yield`, it becomes an async generator function. Calling `sensor.stream()` returns an async generator object that can be iterated with `async for`.

`await asyncio.sleep(self.readIntervalSec)` — The critical line. This suspends the current coroutine for the specified duration and yields control back to the event loop. During this suspension, the event loop can run other coroutines — other sensors can send their requests, receive their responses, and produce records. This is cooperative multitasking.

`yield { "sensorId": ..., "value": random.gauss(50.0, 10.0) }` — After the sleep resolves, we simulate a sensor reading with Gaussian-distributed noise. `random.gauss(50.0, 10.0)` produces values centered at 50 with standard deviation 10 — realistic for a temperature sensor.

`async def ingestOne(sensor):` — A nested coroutine that consumes one sensor's async generator and puts each record into the shared queue. `async for record in sensor.stream()` is the async iteration syntax.

`await asyncio.gather(*[ingestOne(s) for s in sensors])` — `asyncio.gather` runs all coroutines concurrently. While `ingestOne(sensors[0])` is waiting at its `asyncio.sleep`, `ingestOne(sensors[1])` is running, and so on. All four sensors are ingested simultaneously.

`await outputQueue.put(record)` — Puts the record into an `asyncio.Queue`, which is thread-safe for use within the event loop. The `maxsize=20` bound provides backpressure — if the queue is full, `put` suspends until space is available.

---

## CONCEPT 3.1 — asyncio.gather for Multi-Sensor Ingestion

**PROBLEM:**
Even with async generators, if you run each sensor's `ingestOne` coroutine with `await` in a sequential loop — `await ingestOne(sensor1); await ingestOne(sensor2)` — you have not gained concurrency. Each `await` suspends the outer coroutine until the inner one completes. Sensor 2 does not start until sensor 1 has finished all its readings. The problem of sequential waiting is still present, now dressed in async syntax.

**WHY IT WAS INVENTED:**
`asyncio.gather(*coroutines)` schedules all provided coroutines as concurrent tasks and awaits all of them simultaneously. The event loop runs all of them, switching between them at `await` points. When all are done, `gather` returns a list of their results. This is the primary tool for running N independent I/O operations concurrently in asyncio — the async equivalent of spawning N threads and joining them all.

**WHAT HAPPENS WITHOUT IT:**
Without `gather` (or equivalent tools like `asyncio.create_task`), async code runs no faster than sequential code. The `async`/`await` syntax adds overhead without adding concurrency. Many developers fall into this trap when first learning asyncio — they convert functions to `async def` but still call them sequentially with individual `await` statements, then wonder why their async code is slower than the synchronous version.

**INDUSTRY IMPACT:**
`asyncio.gather` is the workhorse of Python async concurrency. FastAPI uses it internally to run background tasks and middleware hooks concurrently. Data pipeline frameworks like Prefect and Airflow use it to run parallel task branches. Web scrapers, API aggregators, and any system that fans out to multiple external services rely on `gather` or the equivalent `asyncio.create_task` pattern. Understanding `gather` is the minimum required to write useful async Python.
