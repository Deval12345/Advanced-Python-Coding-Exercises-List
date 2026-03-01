# Slides — Lecture 24: High-Level Concurrency APIs — Controlling Async at Scale

---

## Slide 1 — Lecture Overview

**From Writing Async Code to Making It Production-Safe**

- Lecture 23 covered Futures and Executors — the mechanical bridge between sync and async
- Writing code that works is one skill; writing code that is safe under scale is another
- Today: five coordination tools that turn a concurrent script into a production async system
- Tools: `asyncio.Semaphore`, `asyncio.wait`, `asyncio.wait_for`, `asyncio.Queue`, `asyncio.Event`
- These tools do not replace `gather` and `create_task` — they build on top of them

---

## Slide 2 — The Unbounded Concurrency Problem

**Why unlimited async parallelism is dangerous**

- `asyncio.gather(*(coroutine() for _ in range(100_000)))` spawns 100,000 concurrent connections
- External APIs respond with HTTP 429 — Too Many Requests; your IP gets blocked
- Your own process exhausts its file descriptor limit — "Too many open files"
- Well-intentioned async code becomes an accidental denial-of-service attack on your vendor
- Solution: encode real-world resource limits directly into your async model

---

## Slide 3 — asyncio.Semaphore: The Parking Lot

**Rate-limiting concurrent coroutines**

- A Semaphore is a parking lot with N spaces — N coroutines allowed inside simultaneously
- `asyncio.Semaphore(N)` creates the gate; `async with semaphore:` is how you enter
- If all N spaces are taken, new coroutines pause at the `async with` until one space opens
- API rate limits (100 requests/second), database connection pools (20 connections) → encode as Semaphores
- Semaphore-wrapped HTTP clients are the standard production pattern for fan-out operations

---

## Slide 4 — asyncio.wait: Fine-Grained Task Coordination

**Three coordination strategies**

- `asyncio.gather` waits for everyone — too blunt for many real patterns
- `asyncio.wait(tasks, return_when=...)` returns control when a meaningful event occurs
- `FIRST_COMPLETED` — race pattern: return as soon as any one task finishes
- `FIRST_EXCEPTION` — fail-fast: return immediately when any task raises
- `ALL_COMPLETED` — like gather but yields individual results and exceptions separately
- Returns two sets: `done` (ready tasks) and `pending` (still running)

---

## Slide 5 — The Race Pattern with FIRST_COMPLETED

**CDN and database replica patterns**

- Submit the same request to multiple regional endpoints simultaneously
- `asyncio.wait` with `FIRST_COMPLETED` returns the moment the fastest responds
- Cancel remaining pending tasks immediately after collecting the winner
- Always `await asyncio.gather(*pending, return_exceptions=True)` to drain cancelled tasks
- Without that drain: dangling tasks produce event loop warnings on shutdown

---

## Slide 6 — asyncio.wait_for: Every External Call Needs a Deadline

**The cascading failure problem**

- A coroutine waiting on a slow or hung external service waits forever without a timeout
- Many hung coroutines → task queue grows → memory climbs → cascading service failure
- `asyncio.wait_for(coro, timeout=seconds)` raises `asyncio.TimeoutError` after the deadline
- On timeout, the underlying coroutine is CANCELLED — its cleanup code still runs
- Production rule: **every external service call must have a timeout**

---

## Slide 7 — Timeout + Retry: The Resilient Service Client Pattern

**Combining wait_for with exponential backoff**

- Catch `asyncio.TimeoutError`, sleep briefly, retry — up to a maximum number of attempts
- Each caller has independent retry state — a timeout in one does not affect others
- asyncio.timeout (Python 3.11+) adds composable context manager timeout blocks
- asyncio.timeout_at accepts an absolute deadline — useful for shared wall-clock deadlines
- This pattern is the foundation of all production resilient API clients

---

## Slide 8 — asyncio.Queue: Async Producer-Consumer with Backpressure

**Dynamic workloads where tasks aren't known upfront**

- `asyncio.gather` requires knowing all tasks upfront — wrong tool for streaming workloads
- `asyncio.Queue` separates the producer (generates work) from the consumer (processes work)
- `maxsize` is the backpressure valve: when the queue is full, `await queue.put()` blocks the producer
- This automatically throttles source rate to match processing capacity
- Used in Kafka/Kinesis async consumers, WebSocket processors, and async job pipelines

---

## Slide 9 — The Sentinel Pattern for Queue Shutdown

**Cleanly stopping consumer coroutines**

- Producer puts `None` after all work items are produced — the shutdown sentinel
- Consumer receives `None`, re-puts it for the next consumer, then breaks its loop
- Re-putting the sentinel ensures all consumers receive the shutdown signal regardless of count
- Never use a shared flag or shared counter for shutdown — race-prone in async code
- The sentinel pattern is the canonical clean-shutdown mechanism for async queues

---

## Slide 10 — asyncio.Event: One-Shot Coordination Signal

**Timing between coroutines**

- `asyncio.Event` starts unset; any number of coroutines can `await event.wait()`
- When one coroutine calls `event.set()`, ALL waiting coroutines are immediately scheduled to resume
- Event stays set — future waiters continue immediately without pausing
- Edge-triggered: wakeup is instant when the flag is set; no polling, no delay
- Use cases: startup coordination (all workers wait for config to load), availability signals (pool has a free connection), initialization gates

---

## Slide 11 — asyncio.Event vs Polling

**Why event signalling beats shared-flag polling**

- Polling loop: check flag, sleep briefly, check again — wastes CPU, adds latency
- Event: wakeup happens the instant `event.set()` fires — zero polling delay, zero wasted work
- asyncio.Condition extends Event with a predicate: wait until a specific condition is true
- asyncio.Event is NOT thread-safe — use only within a single event loop
- For cross-thread signalling: use `loop.call_soon_threadsafe` or `asyncio.Queue` instead

---

## Slide 12 — Lecture 24 Key Principles

**The five coordination tools**

- `asyncio.Semaphore(N)`: limits simultaneous access — encodes rate limits and pool ceilings
- `asyncio.wait(tasks, FIRST_COMPLETED)`: race pattern — use fastest response, cancel others
- `asyncio.wait_for(coro, timeout)`: enforce deadlines — every external call needs one
- `asyncio.Queue(maxsize)`: async producer-consumer with built-in backpressure
- `asyncio.Event`: one-shot signal that wakes all waiting coroutines simultaneously
- Next lecture: Multiprocessing — true CPU parallelism when async is not enough

---
