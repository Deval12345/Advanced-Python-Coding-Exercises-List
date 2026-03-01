# Key Points — Lecture 35: Big Project Stage 3 — Concurrency Layer Part 1 (Async Ingestion)

---

**1. Sequential I/O creates a sampling rate bottleneck that worsens linearly with scale**
Reading N sensors sequentially with round-trip time T gives a cycle time of N * T. With 10 sensors and 100ms latency, the cycle takes 1 second, meaning each sensor is sampled at 0.1Hz. With 100 sensors, each gets 0.01Hz. Asyncio breaks this relationship: regardless of N, all sensors wait concurrently, so the cycle time remains approximately T. 10 sensors at 100ms asyncio: 100ms cycle, 1Hz per sensor. 100 sensors: still ~100ms cycle, still ~1Hz per sensor. The sampling rate is determined by network latency, not sensor count.

**2. The async generator protocol extends the Source protocol without changing its shape**
An async generator is `async def stream(self)` with at least one `yield` inside. It is iterated with `async for record in sensor.stream():` instead of `for record in sensor.stream():`. Within it, `await` expressions can appear at any suspension point. The conceptual shape — a generator that produces records on demand — is identical to the synchronous version. The difference is that `await` points allow the event loop to switch to other coroutines while waiting, whereas synchronous `time.sleep()` or blocking I/O freezes the entire thread.

**3. `asyncio.gather` is the essential tool for concurrent scheduling**
Writing `await coroutine1(); await coroutine2()` runs them sequentially — `coroutine2` does not start until `coroutine1` is complete. `asyncio.gather(coroutine1(), coroutine2())` schedules both and runs them concurrently, switching between them at every `await` point. Without `gather` (or the equivalent `asyncio.create_task`), converting code to use `async/await` syntax adds overhead without adding concurrency — a common trap that leads developers to conclude asyncio is slower than synchronous code.

**4. The event loop is a single-threaded cooperative scheduler**
Asyncio's event loop runs one coroutine at a time. Concurrency is achieved by explicitly yielding control at `await` points. This has a critical implication: any CPU-bound or blocking operation inside an async coroutine — a heavy loop, a `time.sleep()` call, a synchronous database driver — will freeze the entire event loop for its duration. No other coroutines can run. This is why asyncio is appropriate for I/O-bound work (sensor reads, HTTP requests, database queries with async drivers) but inappropriate for CPU-bound work (normalization, statistics).

**5. asyncio.Queue provides backpressure-aware bridging between async and sync layers**
An `asyncio.Queue` with a `maxsize` parameter applies backpressure: if the queue is full, `await queue.put(record)` suspends the producer coroutine until space is available. This prevents the fast async ingestion layer from overwhelming the slower synchronous processing layer. The `maxsize` parameter is a tunable buffer depth — larger buffers tolerate more burstiness but use more memory. The sentinel pattern (putting `None` at the end of ingestion) allows consumers to detect when no more records will arrive without requiring them to know how many records are expected in advance.

**6. Asyncio concurrency is safe without locks because it is single-threaded**
In threading, two threads can access shared state simultaneously, requiring explicit synchronization (locks, semaphores, queues). In asyncio, only one coroutine runs at any moment; other coroutines only run when the current one `await`s. This means asyncio code that appears to share state between coroutines — writing to the same list, reading from the same dict — is actually sequential at every non-await point. `asyncio.Queue` is itself async-aware (it uses internal synchronization that is compatible with the event loop), but ordinary Python data structures do not need locking in pure asyncio code.

**7. `asyncio.run(main())` is the standard entry point for async programs**
`asyncio.run(coro)` creates a new event loop, runs the coroutine until it completes, then closes the loop and cleans up all pending tasks. It should only be called once at the top level of a program. Inside the running loop, `asyncio.gather`, `asyncio.create_task`, and other scheduling primitives are used. `asyncio.run` was added in Python 3.7 to replace the more verbose pattern of manually creating and closing an event loop.

**8. Async generators require special cleanup when abandoned mid-stream**
If you break out of an `async for` loop before the async generator is exhausted — for example, by processing only the first 100 records — the generator is not properly closed unless you call `aclose()` on it or use it as an async context manager. Abandoned async generators hold open any resources (connections, file handles) they were using. Python will issue a `ResourceWarning` and eventually call `aclose()` during garbage collection, but in long-running servers this delay can cause resource exhaustion. Always consume async generators fully or close them explicitly.

**9. The async/sync pipeline split maps naturally to I/O-bound and CPU-bound workloads**
The async ingestion layer (Source protocol with async generators) handles I/O-bound work — waiting for sensors, databases, and network services. The synchronous pipeline stages (filter, normalize, extract features) handle CPU-bound work — computation. The `asyncio.Queue` between them is not just a convenience; it is an architectural boundary that decouples two fundamentally different execution models. This split also makes testing easier: the synchronous pipeline stages can be tested without any async infrastructure.
