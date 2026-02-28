# Slides — Lecture 19: Threading for I/O-Bound Work — Thread Pools, Safety, and Practical Patterns

---

## Slide 1 — Lecture Overview

**Threading for I/O-Bound Work**

- Threads are the correct concurrency tool for I/O-bound tasks
- Python's `concurrent.futures.ThreadPoolExecutor` is the recommended abstraction
- Shared state between threads requires explicit synchronization
- `threading.Lock` and `queue.Queue` are your core safety tools
- Today: pool patterns, race conditions, locks, and producer-consumer design

---

## Slide 2 — Why I/O-Bound Work Benefits from Threading

**Sequential I/O wastes CPU time**

- A thread blocked on network I/O releases the GIL, allowing other threads to run
- 100 URLs at 1 second each = 100s sequential; 10 threads reduces this to ~10s
- The OS scheduler keeps the CPU busy while multiple threads wait simultaneously
- Threading works here because the bottleneck is network latency, not computation
- Rule: if your program spends more time waiting than computing, threading helps

---

## Slide 3 — OS Threads and Thread Creation Cost

**Threads are real OS-managed execution units**

- `threading.Thread` creates an OS-level thread managed by the system scheduler
- Thread creation overhead: ~50–100 microseconds per thread
- Each thread has its own stack: approximately 1 MB of memory
- Creating one thread per task is wasteful at scale (thousands of tasks)
- Thread pools solve this by reusing a fixed set of threads across many tasks

---

## Slide 4 — ThreadPoolExecutor: The Right Abstraction

**`concurrent.futures.ThreadPoolExecutor`**

- Manages a fixed pool of reusable worker threads
- `executor.submit(fn, *args)` — submit a task, returns a `Future` immediately
- `executor.map(fn, iterable)` — apply function to all inputs, yields results in order
- `concurrent.futures.as_completed(futures)` — yields futures as they finish
- Used as a context manager: all tasks complete before the `with` block exits

---

## Slide 5 — Choosing max_workers for I/O-Bound Tasks

**More workers than CPU cores is appropriate**

- Threads are mostly waiting, not computing — high worker count is fine
- Common formula: `min(32, os.cpu_count() + 4)` for I/O-bound pools
- Too many workers: OS context switching overhead, memory pressure, resource contention
- Too few workers: underutilizes available parallelism; tasks queue up waiting
- Profile your specific workload — the right number depends on the external resource

---

## Slide 6 — Shared State and Race Conditions

**Why shared mutable state is dangerous**

- Threads share memory: no copying, but no isolation either
- `count += 1` is NOT atomic — it compiles to: load, add, store (3 bytecode ops)
- The OS can interrupt a thread between any two bytecode instructions
- Two threads reading the same value and both writing back loses one update
- Race conditions are non-deterministic and hard to reproduce in testing

---

## Slide 7 — threading.Lock: The Fix for Shared State

**Mutual exclusion with locks**

- `threading.Lock()` — only one thread can hold the lock at a time
- `lock.acquire()` blocks if another thread holds the lock
- `lock.release()` releases the lock so another thread can proceed
- Context manager usage: `with lock:` guarantees release even on exception
- Keep critical sections small — only protect the shared state access, not surrounding work

---

## Slide 8 — Anatomy of a Race Condition (Diagram)

**What goes wrong without synchronization**

```
Thread A: load count (=5) → [interrupted]
Thread B: load count (=5) → add 1 → store count (=6)
Thread A: [resumes] → add 1 → store count (=6)  ← lost update!
```

- Both threads incremented, but count only increased by 1
- Final value: 6 instead of 7 — the increment from Thread A was lost
- With a Lock: Thread A holds lock → Thread B blocks → Thread A stores 6, releases lock → Thread B loads 6, stores 7
- The Lock serializes access and eliminates the lost update

---

## Slide 9 — queue.Queue: Thread-Safe Communication

**The standard channel between threads**

- `queue.Queue` has internal locks built in — `put()` and `get()` are atomic
- No manual `acquire()`/`release()` needed for basic producer-consumer use
- `maxsize` parameter enables backpressure: `put()` blocks when queue is full
- `task_done()` marks an item as processed; `join()` waits for all items to be processed
- Eliminates the risk of a race condition on a shared list

---

## Slide 10 — The Producer-Consumer Pattern

**Fundamental concurrent pipeline design**

- Producer threads generate work items and call `queue.put(item)`
- Consumer threads loop calling `queue.get()`, process the item, call `task_done()`
- Sentinel value (`None`) signals consumers to stop: re-put it so all consumers see it
- Queue acts as a buffer: decouples the rate of production from the rate of consumption
- Industry use: job queues, event pipelines, log processors, download managers

---

## Slide 11 — ThreadPoolExecutor vs Raw Threads: Comparison

**When to use each**

| Feature | `threading.Thread` | `ThreadPoolExecutor` |
|---|---|---|
| Thread reuse | No | Yes |
| Exception handling | Manual | Propagated via Future |
| Result collection | Manual | `future.result()` |
| Pool size limit | Manual | `max_workers` |
| Cleanup | Manual join | Context manager |

- Use `ThreadPoolExecutor` for task-based parallelism (submitting many callables)
- Use raw `Thread` only when you need daemon threads, thread-specific lifecycle control, or long-running background threads

---

## Slide 12 — Lecture 19 Key Principles

**What to remember**

- Use threading for I/O-bound work; threads release the GIL during I/O waits
- `ThreadPoolExecutor` is the right tool: clean API, safe lifecycle management
- Every shared mutable variable needs a `threading.Lock` — no exceptions
- `queue.Queue` is the standard way for threads to communicate; avoid shared lists
- Minimize critical section size: hold the lock only as long as necessary
- Next lecture: the GIL in depth — what it is, why it exists, and when it limits you

---
