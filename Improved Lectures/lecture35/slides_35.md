# Slides — Lecture 35: Big Project Stage 3 — Concurrency Layer Part 1 (Async Ingestion)

---

## Slide 1: The Sequential Bottleneck

**10 sensors, 100ms read time each:**

Sequential reading:
```
Sensor 1: [=====wait=====][read] 100ms
Sensor 2:                        [=====wait=====][read] 100ms
...                                                          1000ms total
```

- Each sensor: **0.1 Hz** (sampled once per second)
- You wanted: **1 Hz** per sensor
- Add 10 more sensors: **0.05 Hz** per sensor

**The more sensors you add, the slower each one gets.**

---

## Slide 2: Async Reading — All Simultaneously

**Same 10 sensors, same 100ms, with asyncio:**

```
Sensor 1:  [=====wait=====][read]
Sensor 2:  [=====wait=====][read]
Sensor 3:  [=====wait=====][read]
...
All 10:    [=====wait=====][read]  ← all finish at ~100ms
```

- Each sensor: **1 Hz** — what you wanted
- Total time: **~100ms** instead of 1000ms
- **10x improvement. Zero new hardware. Single thread.**

---

## Slide 3: Why Waiting Is Not Work

**During a network request, the CPU is idle.**

CPU state during sequential sensor read:
```
[ SEND REQUEST ] [ idle idle idle idle idle ] [ PROCESS RESPONSE ]
                  ↑ 100ms of nothing
```

Asyncio: while waiting for sensor 1, **process other events**
```
Sensor 1: [ SEND ] [ waiting... ] [ RECV ]
Sensor 2:          [ SEND ][ RECV ]
Sensor 3:          [ SEND ][ RECV ]
         ↑ event loop switches at every await point
```

---

## Slide 4: The AsyncSource Protocol

**Synchronous (Lecture 33):**
```python
def stream(self):
    while True:
        time.sleep(self.interval)  # BLOCKS — nothing else runs
        yield self.read()
```

**Asynchronous (Today):**
```python
async def stream(self):
    while True:
        await asyncio.sleep(self.interval)  # suspends — others run
        yield self.read()
```

**Two characters added (`async`). Execution model transformed.**

---

## Slide 5: Async Generator Mechanics

```python
class AsyncSensorSource:
    async def stream(self):          # async generator function
        count = 0
        while count < self.numReadings:
            await asyncio.sleep(self.readIntervalSec)  # ← suspension point
            yield {                                      # ← yield value
                "sensorId": self.sensorId,
                "value": random.gauss(50.0, 10.0),
            }
            count += 1
```

- `async def` + `yield` = async generator
- Iterate with `async for record in sensor.stream():`
- `await` inside gives event loop control while waiting

---

## Slide 6: The Concurrency Trap

**This looks async but is NOT concurrent:**
```python
# BAD: sequential despite async syntax
await ingestOne(sensor1)   # sensor2 waits for sensor1 to finish
await ingestOne(sensor2)
await ingestOne(sensor3)
```

**This IS concurrent:**
```python
# GOOD: all run simultaneously
await asyncio.gather(
    ingestOne(sensor1),
    ingestOne(sensor2),
    ingestOne(sensor3),
)
```

**`gather` is what creates concurrency. `async` alone does not.**

---

## Slide 7: asyncio.gather — Fan-Out Pattern

```python
async def ingestAllSensors(sensors, outputQueue):
    async def ingestOne(sensor):
        async for record in sensor.stream():
            await outputQueue.put(record)
    
    await asyncio.gather(*[ingestOne(s) for s in sensors])
    await outputQueue.put(None)  # sentinel: ingestion complete
```

- `*[ingestOne(s) for s in sensors]` — fan out to all sensors
- `gather` schedules all coroutines and awaits all completions
- Sentinel `None` signals the consumer to stop

---

## Slide 8: Bridging Async to Sync — asyncio.Queue

**Problem:** async generator produces records; sync pipeline consumes them.

**Solution:** asyncio.Queue as the boundary

```python
queue = asyncio.Queue(maxsize=20)

# Async side (producer)
await queue.put(record)

# Sync side (consumer) — after event loop completes
while not queue.empty():
    record = queue.get_nowait()
    if record is None: break
    pipeline.process(record)
```

`maxsize=20` provides backpressure — producer pauses if consumer is slow.

---

## Slide 9: Why No Locks Are Needed

**Threading:** multiple OS threads run truly in parallel — shared state needs locks.

**Asyncio:** single thread, cooperative multitasking — only one coroutine runs at a time.

```python
# Safe in asyncio — only one coroutine touches the queue at a time
await queue.put(record)   # coroutine 1
await queue.put(record)   # coroutine 2 (runs when coroutine 1 awaits)
```

**Asyncio concurrency is safe by design — explicit `await` points are the only handoff.**

---

## Slide 10: Running the Event Loop

```python
async def main():
    sensors = [
        AsyncSensorSource(f"TEMP_{i:02d}", readIntervalSec=0.05, numReadings=5)
        for i in range(4)
    ]
    queue = asyncio.Queue(maxsize=20)
    
    await asyncio.gather(ingestAllSensors(sensors, queue))
    
    records = []
    while not queue.empty():
        item = queue.get_nowait()
        if item is None: break
        records.append(item)
    
    print(f"Ingested {len(records)} records")

asyncio.run(main())  # creates event loop, runs main, closes loop
```

---

## Slide 11: I/O-Bound vs CPU-Bound

| Type | Example | Right Tool |
|---|---|---|
| I/O-bound | Sensor reads, HTTP, DB queries | asyncio (today) |
| CPU-bound | Normalization, statistics, ML | ProcessPoolExecutor (Lecture 36) |

**Asyncio is wrong for CPU-bound work — it blocks the event loop.**

The next lecture: bridging async ingestion to process-pool computation.

---

## Slide 12: Lecture 35 Summary

**The problem:** sequential reading gives 0.1Hz per sensor when you need 1Hz.

**The mechanism:** async generators + `asyncio.gather` run all sensors simultaneously.

**Key syntax:**
- `async def stream(self): yield` — async generator
- `async for record in sensor.stream():` — async iteration
- `await asyncio.sleep(n)` — suspension point (not blocking)
- `asyncio.gather(*coroutines)` — concurrent scheduling

**The bridge:** `asyncio.Queue` decouples async producers from sync consumers.

**Next:** CPU-bound processing with `ProcessPoolExecutor` — the other half of the concurrency story.
