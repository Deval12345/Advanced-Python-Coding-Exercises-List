# Slides — Lecture 22: The Async Event Loop — How Cooperative Concurrency Works Under the Hood

---

## Slide 1 — Lecture Overview

**Opening the Engine Compartment**

- Lecture 21 taught HOW to write async code; this lecture explains HOW it works
- The event loop is a single-threaded scheduler managing many concurrent I/O operations
- Understanding the execution cycle explains every async performance bug and gotcha
- One blocking call inside any coroutine freezes the ENTIRE event loop
- Today: the loop cycle, blocking hazards, run_in_executor, and cooperative yield points

---

## Slide 2 — The Scalability Problem: Why the Event Loop Exists

**Thread-per-connection does not scale**

- Each OS thread requires approximately 1 MB of stack memory
- 10,000 connections = 10,000 threads = 10 GB of RAM just for thread stacks
- OS context switching between thousands of threads adds measurable latency
- Most of a web server's time is spent waiting on I/O — not computing
- The event loop solution: one thread that manages all waiting, never standing idle

---

## Slide 3 — The Restaurant Analogy

**One waiter, many tables**

- The event loop is a single waiter managing an entire restaurant
- A customer places an order (coroutine starts I/O) → waiter takes it to the kitchen (OS receives request) → waiter moves immediately to the next table
- The waiter does NOT stand at the kitchen window waiting
- When the kitchen rings (I/O completes), the waiter returns to deliver the result
- One thread; thousands of concurrent operations; no idle standing

---

## Slide 4 — Inside the Event Loop: Two Data Structures

**What the loop tracks**

- Ready queue: coroutines that have data and can run right now
- I/O registry: coroutines that are suspended, each paired with the file descriptor or timer they are waiting on
- When the ready queue is empty, the loop calls the OS I/O multiplexer: `select` / `epoll` / `kqueue`
- This is a single blocking OS call that wakes when at least one I/O completes
- The OS reports which descriptors are ready; the loop moves those coroutines into the ready queue

---

## Slide 5 — The Event Loop's Three-Step Execution Cycle

**Run → Poll → Resume → Repeat**

- Step 1: Run all coroutines in the ready queue until each one voluntarily yields at an `await`
- Step 2: Call the OS I/O multiplexer with all pending file descriptors — wait until at least one is ready
- Step 3: Move coroutines whose I/O completed from the I/O registry back into the ready queue
- "Cooperative" means the event loop NEVER forces a coroutine to pause — it must yield willingly
- A coroutine that runs without an `await` monopolizes the entire event loop for that duration

---

## Slide 6 — asyncio.run() and the Loop Lifecycle

**The correct entry point for async programs**

- `asyncio.run(coro)` creates a new event loop, runs `coro` to completion, then closes the loop
- It is the only recommended way to start an async program at the top level
- `asyncio.get_event_loop()` returns the running loop from inside an async context
- Never create event loops manually or leave them unclosed — `asyncio.run()` handles cleanup
- `asyncio.sleep()` vs `time.sleep()`: one suspends the coroutine; the other blocks the OS thread

---

## Slide 7 — Blocking Calls Are Catastrophic

**The most dangerous mistake in async code**

- One blocking call freezes the ENTIRE event loop — not just the one coroutine
- The bug does not crash: the server becomes mysteriously slow under load
- Root cause is distant from symptom — a rarely-hit code path that blocks serialises all traffic
- Four common culprits: `time.sleep()`, `requests.get()`, synchronous file reads, heavy CPU loops
- Rule: inside `async def`, NEVER call anything that blocks the OS thread

---

## Slide 8 — Solutions for Blocking Code

**Two strategies to stay cooperative**

- Strategy 1 — Use async-native equivalents: `asyncio.sleep` instead of `time.sleep`; `aiohttp` / `httpx` instead of `requests`; `aiofiles` instead of `open`
- Strategy 2 — Use `loop.run_in_executor(executor, fn, *args)`: runs the blocking function on a thread pool, coroutine awaits the result without blocking the loop
- `run_in_executor` with a `ThreadPoolExecutor` is correct for I/O-bound legacy functions
- `run_in_executor` with a `ProcessPoolExecutor` is correct for CPU-bound functions
- Result: the event loop is free to run other coroutines while the executor thread blocks

---

## Slide 9 — asyncio.sleep(0): Cooperative Yield Without Waiting

**The manual yield point for CPU-heavy loops**

- `asyncio.sleep(0)` suspends the coroutine for zero seconds and returns control to the event loop
- The event loop runs all other ready coroutines for one cycle, then immediately resumes this one
- Use inside loops that do CPU work without natural I/O await points
- Without it: a loop processing 1 million items freezes all other coroutines for its entire duration
- Pattern: `await asyncio.sleep(0)` every N iterations, chosen so each batch takes only a few milliseconds

---

## Slide 10 — The Single-Threaded Guarantee

**What single-threaded means for correctness**

- Between any two consecutive `await` points in the same coroutine, that coroutine has the thread exclusively
- No other coroutine can interleave with its code during that window
- No locks needed for local variable access; no race conditions for code between yield points
- Contrast with threads: any instruction can be interrupted by the OS at any time
- The tradeoff: any blocking call anywhere freezes everything — cooperative discipline is mandatory

---

## Slide 11 — Event Loop vs. Threading: Side-by-Side

**Choosing the right concurrency model**

| Property | Event Loop (async) | Threading |
|---|---|---|
| Memory per unit | Tiny (coroutine frame) | ~1 MB (thread stack) |
| Race conditions | Only across `await` points | Every instruction |
| Blocking call impact | Freezes ALL coroutines | Freezes only that thread |
| Scale (connections) | Hundreds of thousands | Thousands |
| Legacy library support | Requires run_in_executor | Native |

- Use async for high-concurrency I/O when async libraries are available
- Use threads for legacy synchronous libraries or simpler concurrency requirements

---

## Slide 12 — Lecture 22 Key Principles

**What to remember**

- The event loop runs a three-step cycle: run ready coroutines, poll OS for I/O, resume completed ones
- Coroutines yield at `await` — the event loop NEVER forcibly interrupts them
- A blocking call inside any coroutine freezes the entire event loop — use async libraries or `run_in_executor`
- `asyncio.sleep(0)` yields for one event loop cycle — essential for cooperative CPU-heavy loops
- `asyncio.run()` is the only correct entry point: creates loop, runs coroutine, closes loop
- Next lecture: Futures and Executors — the bridge between async and synchronous-library code

---
