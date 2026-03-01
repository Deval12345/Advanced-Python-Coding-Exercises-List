# Speech — Lecture 35: Big Project Stage 3 — Concurrency Layer Part 1 (Async Ingestion)

---

I want you to imagine a very specific scenario, because it is the scenario that motivates everything we are doing today.

You have ten temperature sensors deployed across a factory floor. Each sensor has a network endpoint — you send it a request, it sends back a temperature reading. The round trip takes 100 milliseconds. Nothing dramatic. 100 milliseconds.

If you read those ten sensors sequentially — ask sensor 1, wait 100ms, ask sensor 2, wait 100ms, all the way through — your complete reading cycle takes one full second. One second for ten readings. Each individual sensor gets sampled once per second — at the absolute best.

Now add twenty more sensors. Now you need 3 seconds per cycle. Each sensor is sampled once every 3 seconds. You wanted 1Hz. You have 0.33Hz and getting worse. The more sensors you add, the slower each individual sensor gets sampled, because you are burning time waiting for one sensor while all the others sit idle.

This is the sequential ingestion bottleneck, and it is not a software problem. It is a conceptual problem. You are treating waiting as work. It is not.

When you send a request to a sensor and wait for the response, your CPU is idle. It is doing nothing. It is not computing. It is not transforming data. It is waiting. And while it waits, nine other sensors could be sending their requests and receiving their responses simultaneously. The CPU does not need to be involved in the waiting — it only needs to be involved in processing the response when it arrives.

This is the core insight behind asyncio — Python's asynchronous I/O framework. A single thread manages many pending I/O operations. Whenever one operation is waiting, the thread switches to another. When a response arrives, the thread comes back to process it. The waiting is handled by the operating system's I/O subsystem, not by the CPU. The CPU only wakes up when there is actual work to do.

Let me show you how this changes the math. Ten sensors, 100ms round-trip each. With sequential reading: 1 second per cycle. With asyncio: all ten requests are sent simultaneously, all ten responses arrive at roughly the same time, total time is approximately 100ms. You get 10Hz total throughput — one reading per sensor per 100ms — instead of 0.1Hz per sensor. That is a ten-fold improvement without any hardware change, without any algorithmic change, without any additional servers.

The mechanism that makes this possible is the async generator.

In Lecture 33, we defined the Source protocol with a synchronous `stream()` generator — `def stream(self): yield record`. That works perfectly for data already in memory. But for a network sensor, that `yield` means the thread blocks for 100ms doing nothing. The event loop is frozen. No other sensors can run.

The fix is syntactically minimal. We change `def stream(self)` to `async def stream(self)`. We add `await asyncio.sleep(readInterval)` before each reading to simulate the network wait. The shape of the code is identical. The execution model is fundamentally different.

An `async def` function that contains `yield` is an async generator. It can be iterated with `async for`, and at every `await` inside it, control returns to the event loop. The event loop can run other coroutines while this one waits. When the wait completes, the event loop comes back and the coroutine continues from exactly where it left off.

`asyncio.sleep(0.05)` — this is our simulated sensor read delay. Fifty milliseconds. In a real system, this would be an `await aiohttp.get(sensorEndpoint)` or an `await asyncpg.fetch(query)`. The mechanics are identical. The `await` says: "I am going to be waiting for something. Give the event loop a chance to run other things while I wait."

Now — and this is the trap that catches almost every developer new to asyncio — having async generators is not enough by itself. You can write perfectly correct async generators and still get zero concurrency, if you do not schedule them to run simultaneously.

Consider this: if you do `await ingestOne(sensor1); await ingestOne(sensor2)`, you have not gained anything. The second `await` does not start until the first one is completely finished. Async syntax without concurrent scheduling is just overhead.

The tool for concurrent scheduling is `asyncio.gather`. It takes any number of coroutines and schedules them all to run at the same time. While sensor 1's coroutine is waiting at its `asyncio.sleep`, sensor 2's coroutine is running. When sensor 1's sleep resolves, the event loop comes back to it. They interleave at every suspension point.

`asyncio.gather(*[ingestOne(s) for s in sensors])` — this is the line that creates the concurrency. The list comprehension creates a coroutine for each sensor; the `*` unpacking passes them all to `gather`; `gather` schedules them all and awaits all of them simultaneously. All ten sensors are now ingesting in parallel, on a single thread, with no additional memory overhead beyond the coroutines themselves.

Now there is a challenge: our pipeline stages from Lecture 33 are synchronous. They expect a regular Python iterator, not an async generator. We need a bridge — somewhere that async records can be collected and made available to the synchronous pipeline.

That bridge is `asyncio.Queue`. The async ingestion layer — running under the event loop — puts each record into the queue with `await queue.put(record)`. The synchronous pipeline layer reads from the queue. The queue decouples the two worlds. The async side can be running faster or slower than the sync side; the queue absorbs the difference.

We send a sentinel `None` value when all sensors are done, so the consumer knows to stop reading. This is the producer-consumer pattern — classic, reliable, and exactly right for crossing the async/sync boundary.

Let me address something you might be wondering. We are using a single `asyncio.Queue` that all sensors write into. Does this create a race condition? No — because asyncio uses cooperative multitasking on a single thread. There is no true parallelism. Only one coroutine runs at a time; they take turns. The `await queue.put(record)` call is a suspension point — if the queue is full, the coroutine waits, but only one coroutine is ever modifying the queue at any given moment. No locks needed.

This is one of asyncio's great advantages over threading. With threads, shared state requires locks, and locks are a source of bugs — deadlocks, priority inversion, forgotten acquisition. With asyncio, shared state is safe because the concurrency is cooperative: you only give up control at explicit `await` points.

We are building something important here. The async ingestion layer is the front of the pipeline — the interface with the physical world. Sensors are I/O-bound — they are slow because of network latency, not because of computation. Asyncio handles I/O-bound concurrency with minimal overhead.

Next lecture, we will encounter the opposite problem: the computation inside the pipeline — normalization, statistics, feature extraction — is CPU-bound. The event loop is the wrong tool for CPU-bound work. We will bring in process pools for that. Two different problems, two different solutions, bridged by a queue.

But today, we have solved the first problem: reading from many sensors at the sampling rate they require, using the minimum possible resources. That is async ingestion.
