# Code — Lecture 35: Big Project Stage 3 — Concurrency Layer Part 1 (Async Ingestion)

---

## Example 35.1 — AsyncSensorSource and Concurrent Ingestion

```python
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

**Line-by-line explanation:**

`class AsyncSensorSource:` — Represents a single physical sensor with an async streaming interface. The class structure is nearly identical to the synchronous version from Lecture 33; the difference is in the `stream()` method signature.

`def __init__(self, sensorId, readIntervalSec, numReadings=None):` — `sensorId` is a string identifier for logging and routing. `readIntervalSec` controls how long to wait between readings. `numReadings=None` means "run forever" — useful for long-running pipelines. A finite value is useful for testing.

`async def stream(self):` — `async def` makes this a coroutine function. The presence of `yield` inside it makes it an async generator function. Calling `sensor.stream()` returns an async generator object, not a regular generator. The distinction matters: you iterate it with `async for`, not `for`.

`while self.numReadings is None or count < self.numReadings:` — The loop runs indefinitely if `numReadings` is `None`, or for exactly `numReadings` iterations otherwise. `is None` is the idiomatic Python check for the sentinel value.

`await asyncio.sleep(self.readIntervalSec)` — This is the I/O suspension point. `asyncio.sleep` returns a coroutine that completes after the specified duration. `await` suspends this coroutine and returns control to the event loop. During the sleep, the event loop can run other coroutines — crucially, the `ingestOne` coroutines for other sensors. This is what creates concurrency.

`yield { "sensorId": self.sensorId, "timestamp": time.time(), "value": random.gauss(50.0, 10.0), "unit": "celsius" }` — Produces one sensor record. `random.gauss(50.0, 10.0)` generates values from a normal distribution with mean 50 and standard deviation 10, simulating realistic sensor noise. `time.time()` gives the current Unix timestamp as a float.

`count += 1` — Incremented after the yield, so it counts completed yields.

`async def ingestAllSensors(sensors, outputQueue):` — An async function (not a generator) that orchestrates concurrent ingestion from multiple sensors. It does not produce records itself; it manages the fan-out to individual sensor coroutines.

`async def ingestOne(sensor):` — A nested async function defined inside `ingestAllSensors`. Closures work the same way in async functions as in synchronous ones — `sensor` from the outer scope is captured. This function consumes one sensor's stream and forwards records to the shared queue.

`async for record in sensor.stream():` — Async iteration syntax. This is the consumer side of the async generator. On each iteration, Python resumes the generator, which runs until the next `yield`, and the yielded value is bound to `record`. The `async for` itself suspends at each `await asyncio.sleep` inside the generator, allowing other coroutines to run.

`await outputQueue.put(record)` — `asyncio.Queue.put` is an async method — if the queue is at `maxsize`, it suspends the coroutine until space is available. This is backpressure: if the downstream consumer is slower than ingestion, the queue fills up and the producers slow down automatically.

`await asyncio.gather(*[ingestOne(s) for s in sensors])` — List comprehension creates one `ingestOne(s)` coroutine per sensor. `*` unpacks the list into positional arguments. `asyncio.gather` schedules all coroutines concurrently and awaits all of them. The `await` here suspends `ingestAllSensors` until every sensor has finished all its readings.

`await outputQueue.put(None)` — Sentinel value. After all sensors complete, we put `None` to signal that no more records will arrive. The consumer side checks for `None` to know when to stop draining.

`sensors = [AsyncSensorSource(f"TEMP_{i:02d}", readIntervalSec=0.05, numReadings=5) for i in range(4)]` — Creates 4 sensor sources. `f"TEMP_{i:02d}"` formats i as a zero-padded two-digit integer: `TEMP_00`, `TEMP_01`, `TEMP_02`, `TEMP_03`. `readIntervalSec=0.05` is 50ms — fast enough to demonstrate concurrency without taking long to run. `numReadings=5` means each sensor produces exactly 5 records, for a total of 20.

`queue = asyncio.Queue(maxsize=20)` — Creates a bounded async-safe queue. `maxsize=20` means the queue holds at most 20 items. With 4 sensors producing 5 records each (20 total), this just fits. In production, size it larger than your expected burst.

`await asyncio.gather(ingestAllSensors(sensors, queue),)` — The trailing comma makes this a single-element tuple, which is fine for `gather`. This pattern allows easy addition of other concurrent tasks alongside ingestion.

`while not queue.empty():` — Drains the queue after the event loop portion completes. `queue.empty()` is synchronous and safe to call from sync code when the event loop has finished.

`item = queue.get_nowait()` — Synchronous non-blocking get. Raises `asyncio.QueueEmpty` if empty. We are safe here because we checked `queue.empty()` one line earlier, but in production code, wrap this in try/except.

`if item is None: break` — Check for the sentinel and stop draining. Note that `is None` is used instead of `== None` — best practice for sentinel checks.

`asyncio.run(main())` — The standard entry point for asyncio programs in Python 3.7+. Creates a new event loop, runs `main()` until it completes or raises, then closes the loop and performs cleanup. Should only appear once at the top level of a script.

---

## Example 35.2 — Demonstrating Sequential vs Concurrent Timing

```python
import asyncio
import time

async def simulatedSensorRead(sensorId, delayMs):
    """Simulate a network read that takes delayMs milliseconds."""
    await asyncio.sleep(delayMs / 1000)
    return {"sensorId": sensorId, "value": 42.0}

async def sequentialReads(sensorIds, delayMs):
    start = time.perf_counter()
    results = []
    for sid in sensorIds:
        result = await simulatedSensorRead(sid, delayMs)
        results.append(result)
    elapsed = (time.perf_counter() - start) * 1000
    print(f"Sequential: {len(results)} reads in {elapsed:.0f}ms "
          f"({elapsed/len(results):.0f}ms per sensor)")
    return results

async def concurrentReads(sensorIds, delayMs):
    start = time.perf_counter()
    results = await asyncio.gather(*[
        simulatedSensorRead(sid, delayMs) for sid in sensorIds
    ])
    elapsed = (time.perf_counter() - start) * 1000
    print(f"Concurrent: {len(results)} reads in {elapsed:.0f}ms "
          f"({elapsed/len(results):.0f}ms per sensor)")
    return results

async def main():
    sensorIds = [f"S{i}" for i in range(10)]
    delayMs = 50  # 50ms per sensor read
    
    print(f"\n{len(sensorIds)} sensors, {delayMs}ms read time each:")
    await sequentialReads(sensorIds, delayMs)
    await concurrentReads(sensorIds, delayMs)

asyncio.run(main())
```

**Line-by-line explanation:**

`async def simulatedSensorRead(sensorId, delayMs):` — A simple coroutine that simulates one sensor network read. The docstring clarifies the simulation.

`await asyncio.sleep(delayMs / 1000)` — The simulated network latency. 50ms becomes 0.05 seconds.

`async def sequentialReads(sensorIds, delayMs):` — Reads each sensor one at a time using a `for` loop with `await`. Despite using `async/await`, this is not concurrent — each `await simulatedSensorRead` completes before the loop moves to the next sensor.

`for sid in sensorIds: result = await simulatedSensorRead(sid, delayMs)` — Sequential await: the loop waits for each sensor to complete before starting the next. With 10 sensors at 50ms each, this takes ~500ms.

`async def concurrentReads(sensorIds, delayMs):` — Reads all sensors simultaneously using `gather`.

`results = await asyncio.gather(*[simulatedSensorRead(sid, delayMs) for sid in sensorIds])` — Creates 10 coroutines and runs them concurrently. All 10 `asyncio.sleep` calls overlap — they all start at roughly the same time and all finish at roughly the same time. Total time: ~50ms instead of ~500ms.

`print(f"Sequential: ... ({elapsed/len(results):.0f}ms per sensor)")` — Shows effective per-sensor rate. Sequential: 50ms per sensor (10 * 50 / 10 = 50ms, but total is 500ms). Concurrent: 5ms per sensor (50ms total / 10 sensors).
