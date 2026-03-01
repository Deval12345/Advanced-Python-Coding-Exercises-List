# Slides — Lecture 25: Multiprocessing — True Parallelism Beyond the GIL

---

## Slide 1 — Lecture Overview

**The Door Through the GIL**

- The GIL is per-interpreter — one Python interpreter, one GIL, one core at a time
- Multiprocessing creates separate OS processes: each has its own interpreter, GIL, and memory
- True CPU parallelism — multiple cores running Python simultaneously, no contention
- Today: process creation, IPC primitives, pickling costs, and the Pool API
- When to use multiprocessing: CPU-bound work; when NOT to: tiny tasks, large data transfer

---

## Slide 2 — Why Threads Fail on CPU-Bound Work

**The GIL wall**

- A thread doing CPU computation holds the GIL — all other threads wait for it
- 8 threads, 8 cores, 1 GIL: only 1 core runs Python bytecode at any instant
- Threading helps I/O-bound work because threads yield the GIL while waiting
- For pure computation — prime counting, image processing, data transformation — no speedup
- Solution: create processes instead of threads — each gets its own GIL

---

## Slide 3 — Process Start Methods: Fork vs Spawn

**Platform differences matter**

- **Linux default: `fork`** — child gets complete copy of parent memory (copy-on-write); fast startup; duplicates file descriptors and locks — can cause subtle bugs
- **macOS (≥3.8) and Windows default: `spawn`** — fresh interpreter, imports module from scratch; safer; 50–200ms startup cost per process
- **The mandatory `if __name__ == "__main__"` guard:** spawn re-imports your script; without the guard, each child spawns more children → infinite fork bomb → machine crash
- **Always** wrap your main code in this guard when using multiprocessing

---

## Slide 4 — Process Overhead: Be Honest About Costs

**Processes are heavyweight**

- Process startup: 50–200ms per process (depends on imported libraries)
- Memory per process: 30–100 MB (Python interpreter + loaded modules)
- Never create one process per small task — overhead dominates computation
- **Solution: Process pools** — create N workers once, reuse them for thousands of tasks
- Rule of thumb: use multiprocessing when each task does ≥100ms of CPU work

---

## Slide 5 — Inter-Process Communication: The Price of Isolation

**No shared memory means everything must be serialized**

- Processes cannot share variables — they live in completely separate memory spaces
- All data crossing process boundaries is **pickled** (serialized) and **unpickled** (deserialized)
- IPC channel types: Queue (general), Pipe (point-to-point), Value/Array (shared numeric memory)
- Pickling overhead is real: a 100 MB array crossing a process boundary is copied 3 times
- The golden rule: **send small task parameters in; receive small results out; heavy work stays inside the worker**

---

## Slide 6 — multiprocessing.Queue: Producer-Consumer Across Processes

**The general-purpose IPC channel**

- `multiprocessing.Queue` is process-safe FIFO backed by a pipe and internal locks
- `queue.put(item)` serializes and sends; `queue.get()` blocks until an item is ready
- Use sentinel pattern: producer puts `None` for each worker to signal shutdown
- Each worker: receive `None` → break the loop → process exits cleanly
- Design pattern: task queue + result queue — workers pull tasks, push results

---

## Slide 7 — multiprocessing.Pool: The High-Level Process Pool

**Amortize startup cost across many tasks**

- `Pool(processes=N)` creates N workers at startup; keeps them alive for reuse
- `pool.map(fn, iterable)` — distributes items, collects results in submission order (blocking)
- `pool.imap(fn, iterable)` — lazy iterator; yields results as they complete; memory-efficient
- `pool.apply_async(fn, args)` — submit one task; returns `AsyncResult` for later `.get()`
- Use as context manager: `with Pool(4) as pool:` ensures clean shutdown

---

## Slide 8 — The Chunksize Parameter

**Reducing IPC overhead for small tasks**

- `pool.map(fn, items)` by default sends one item per IPC message
- With 10,000 small items: 10,000 IPC round-trips — queue overhead dominates
- `pool.map(fn, items, chunksize=100)` batches 100 items per IPC message → 100 round-trips
- Chunksize trades latency for throughput: higher chunksize = fewer IPC calls = faster for small uniform tasks
- Rule: increase chunksize when tasks are small and the iterable is large

---

## Slide 9 — The Minimal IPC Principle

**Why video pipelines send file paths, not frames**

- Naive design: serialize 4K video frames, send through Queue, deserialize in worker → slower than sequential
- Correct design: send file paths (a few bytes) → each worker reads its own file from disk
- Same principle for ML preprocessing: send row index ranges, not full datasets
- Shared memory (`multiprocessing.Value`, `Array`, `mmap`) for numeric data avoids serialization entirely
- Next lecture: zero-copy NumPy sharing — sending data without pickling at all

---

## Slide 10 — When Multiprocessing Hurts

**Knowing when NOT to use it**

- Task duration < serialization time → slower than sequential
- Large data must cross process boundary → IPC overhead dominates
- Many tiny short-lived processes → startup cost dominates
- Process failure isolation helps OR hurts: a crashed worker cannot corrupt shared state, but a crashed pool worker loses its in-flight task
- Profile first: measure sequential vs. parallel before assuming a speedup

---

## Slide 11 — multiprocessing vs. Threading vs. async: The Decision Tree

**Choosing the right model**

| Workload | Best Tool |
|---|---|
| High-concurrency network I/O | async/await |
| I/O-bound + legacy sync libs | threading + run_in_executor |
| CPU-bound, parallelizable | multiprocessing.Pool |
| CPU-bound, large data | multiprocessing + mmap/shared memory |

- Mixing is valid: async event loop handles I/O + process pool handles computation
- Next lectures explore exactly this hybrid architecture

---

## Slide 12 — Lecture 25 Key Principles

**What to carry forward**

- Multiprocessing bypasses the GIL by giving each process its own Python interpreter
- Isolation is a guarantee: processes cannot accidentally corrupt each other's state
- `if __name__ == "__main__"` is mandatory — without it, spawn creates infinite child loops
- Keep IPC minimal — pickling is the performance bottleneck; send parameters, not data payloads
- Use Pool to amortize process startup cost; raw Process for fine-grained control
- Next lecture: IPC cost analysis, shared memory, and the boundary where multiprocessing stops helping

---
