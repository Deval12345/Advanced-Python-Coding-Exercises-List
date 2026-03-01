# Lecture 29: Async + Multiprocessing Hybrid Architecture
## speech_29.md — Instructor Spoken Narrative

---

Today we combine the two major concurrency paradigms we have built up over these lectures — asyncio and multiprocessing — into a single architecture. And I want to be clear at the outset: this is not a compromise or a workaround. This is the correct architecture for a specific, extremely common problem. Let me describe that problem precisely.

You are building a web service. Clients send requests. Each request requires two things: first, I/O work — reading the request body, parsing JSON, validating inputs, writing a response. Second, CPU work — running a machine learning model, computing features, transforming data. These are not the same kind of work. They do not benefit from the same concurrency mechanism.

The I/O work benefits from async. A single thread in an event loop can handle thousands of simultaneous clients because most of the time, those clients are waiting — for network bytes to arrive, for a database query to return, for the response to be written. The event loop multiplexes all of this waiting onto one thread, switching context every time a client has to wait for something. No threads needed; no GIL contention.

The CPU work does not benefit from async. Running a model means executing Python — numpy operations, math, loops — that never yields to the event loop. If you run model inference synchronously inside a coroutine, the event loop is frozen for the entire duration of that inference. During that time — say, 200 milliseconds — every other client's request is paused, waiting for the frozen event loop to wake up. Your 200-millisecond inference becomes the bottleneck for every client simultaneously.

This is the motivation for the hybrid. Async handles the I/O concurrency. Processes — on separate CPU cores, bypassing the GIL — handle the CPU computation. loop.run_in_executor is the bridge.

Here is exactly how the bridge works. Inside a coroutine, you call loop.run_in_executor and pass it the ProcessPoolExecutor and the function you want to run. run_in_executor submits the function to the process pool — which already has worker processes warm and ready — and immediately returns to the event loop. The event loop continues handling other coroutines: other clients' I/O, other handlers finishing their work. When the process pool worker finishes, the event loop is notified, and it resumes the coroutine that was awaiting, with the result.

From the coroutine's perspective, it dispatched work and awaited the result. From the event loop's perspective, the coroutine was suspended and other work ran in the meantime. From the process pool's perspective, it received a regular function call and ran it to completion. Each layer sees exactly what it needs to see. No layer is blocked waiting for another.

Example 29.1 makes this concrete. Six simulated requests arrive — different data sizes, different computation times. All six coroutines are dispatched via asyncio.gather, which starts them all simultaneously. Each coroutine dispatches to the process pool and awaits. The event loop runs all six coroutines concurrently — they are all suspended, waiting for their process pool workers. The process pool has four workers. The first four tasks start immediately; the remaining two wait in the queue. As workers finish, the next queued tasks begin. Total elapsed time is approximately the time for the longest-running batch of tasks, not the sum of all tasks.

Now let me point out the key design decisions in that example. The ProcessPoolExecutor is created once — outside the request handler coroutines, in the main function — and passed to each handler as an argument. This is essential. Process pool startup costs 50 to 200 milliseconds per worker. If you created a new executor inside each handler, every request would pay that startup cost. Create the executor once; share it across all handlers for the lifetime of the application. In a real web framework, you would create it in the application's startup event and close it in the shutdown event.

Now let us look at the producer-consumer variant in Example 29.2. This is the streaming pipeline pattern. An async producer generates jobs one at a time, with small delays between them — simulating data arriving from a network stream or a message queue. It puts jobs into an asyncio.Queue with a bounded size. An async submitter reads from the queue and dispatches each job to the process pool. The two coroutines run concurrently via asyncio.gather.

The bounded queue — maxsize equals 10 — is the backpressure mechanism. If the submitter is falling behind the producer, the queue fills up. When it fills, the producer's await jobQueue.put blocks until the submitter takes something out. This prevents the producer from generating jobs faster than they can be processed, bounding memory usage. For large jobs — each carrying a sizable array — an unbounded queue could exhaust available RAM before the process pool makes any progress. The maxsize parameter is the rate-limiting valve.

I want to be explicit about two common mistakes I see in hybrid code.

Mistake one: calling executor.submit directly from a coroutine. executor.submit returns a concurrent.futures.Future — not an asyncio awaitable. If you try to await it, you get a TypeError. The correct call is always loop.run_in_executor, which wraps the concurrent.futures.Future in an asyncio-compatible awaitable. This is not a minor syntax difference; it determines whether your event loop remains unblocked.

Mistake two: trying to share the event loop across processes. The event loop is process-local. Each OS process running asyncio.run creates its own event loop, completely isolated from all others. You cannot send a coroutine to a worker process and have it run in the main event loop. You cannot pickle an event loop and pass it as an argument. Only regular, picklable functions can cross the process boundary. Coroutines cannot. The division is absolute: async code runs in the main process; synchronous CPU-bound functions run in worker processes.

The architecture this implies is clean: the event loop and all coroutines live in the main process; the process pool contains only synchronous functions; run_in_executor is the only point of contact between the two worlds. Everything above run_in_executor is async; everything below it is synchronous. No coroutines cross this boundary.

This is the discipline that makes the hybrid architecture both correct and maintainable. Async for I/O; processes for CPU; run_in_executor as the bridge; one executor per application, not one per request.
