# Slides — Lecture 26: Multiprocessing Part 2 — IPC Cost, Shared Memory, and When Parallelism Fails

---

## Slide 1 — Lecture Overview

**The Cost of Isolation — Making the Implicit Explicit**

- Last lecture: what multiprocessing does — today: what it COSTS
- Three data copies per IPC message: serialize, transfer, deserialize
- Three failure scenarios where multiprocessing makes code SLOWER
- ProcessPoolExecutor as the high-level, futures-compatible alternative to Pool
- The universal rule: profile before committing to multiprocessing

---

## Slide 2 — The Three-Copy Model of IPC

**What happens when data crosses a process boundary**

- Copy 1: `pickle.dumps(obj)` — serialize object into bytes buffer in sending process
- Copy 2: OS writes bytes through kernel pipe into pipe buffer
- Copy 3: `pickle.loads(bytes)` — reconstruct object in receiving process
- A 100 MB NumPy array crossing one process boundary = 300 MB peak memory pressure
- Typical pickling speed: ~100 MB/s → 1s overhead for every 100 MB task input/output

---

## Slide 3 — Real Failure: The Video Analytics Pipeline

**1.5 GB/second of IPC overhead**

- 4K video frame ≈ 50 MB NumPy array
- Sending frames to workers: 50 MB × 30 fps = 1.5 GB/second through serialization
- System becomes memory-bound — parallelism makes it SLOWER
- Correct design: send file paths (bytes) → workers read from disk independently
- Lesson: data crossing boundaries must be parameterization info, not data payload

---

## Slide 4 — The IPC Decision Matrix

**When does parallelism pay off?**

| Computation / Data | Small Data | Large Data |
|---|---|---|
| **Heavy Compute** | ✅ Clear Win | ⚠️ Measure First |
| **Light Compute** | ❌ IPC Overhead Wins | ❌ Both IPC + Compute Lose |

- Rule of thumb: task duration must be >> IPC overhead (~5–20ms round-trip)
- Task duration < 50ms: question whether multiprocessing is worth it
- Task duration > 200ms with small output: almost always beneficial

---

## Slide 5 — concurrent.futures.ProcessPoolExecutor

**Same interface as ThreadPoolExecutor, backed by processes**

- `ProcessPoolExecutor(max_workers=4)` → same API as `ThreadPoolExecutor`
- Returns real `concurrent.futures.Future` objects (not `AsyncResult`)
- Composes with `as_completed`, `add_done_callback`, `asyncio.wrap_future`
- One-line switch: `ThreadPoolExecutor` → `ProcessPoolExecutor`
- Choose ProcessPoolExecutor when process work must compose with futures or asyncio

---

## Slide 6 — Pool vs. ProcessPoolExecutor: Which to Use

**Two interfaces, different trade-offs**

| Feature | `multiprocessing.Pool` | `ProcessPoolExecutor` |
|---|---|---|
| Return type | `AsyncResult` | `Future` |
| asyncio integration | Manual wrapping | `asyncio.wrap_future` |
| `maxtasksperchild` | ✅ Yes | ❌ No |
| Worker initializer | ✅ Yes | ✅ Yes |
| `imap` / streaming | ✅ Yes | ❌ No |
| Exception propagation | Via `.get()` | Via `.result()` |

- Use Pool for fine-grained worker control; use ProcessPoolExecutor for futures composition

---

## Slide 7 — Failure Scenario 1: Tasks Too Short

**IPC overhead dominates computation**

- Process pool IPC round-trip (warm pool): 2–20 ms per task depending on payload
- Sequential: 8 tasks × 1 ms = 8 ms
- Pool with 5ms IPC: 8 tasks × (1 ms compute + 5 ms IPC) = 48 ms (6× SLOWER)
- Threshold: task duration must be several times larger than IPC round-trip cost
- Solution: batch small tasks into larger chunks before sending to workers

---

## Slide 8 — Failure Scenario 2: Large Data Payload

**Serialization cost exceeds computation benefit**

- Task: process a 100 MB array in 200 ms — but pickling it takes 1–2 seconds
- 90% of wall-clock time is serialization; only 10% is the actual work you're parallelizing
- Four workers: each still pickles independently — 4 × 1s = 4s of serialization overhead alone
- Plus: 4 × 300 MB peak memory pressure during transit → may trigger OS swapping
- Solution: use shared memory (mmap, multiprocessing.Array, or numpy.memmap) — next lecture

---

## Slide 9 — Failure Scenario 3: Frequent Inter-Worker Coordination

**Coordination overhead compounds with worker count**

- Workers that check a shared Manager flag 10×/sec generate 40 IPC calls/sec with 4 workers
- Manager calls cross a process boundary every time — each is a serialization round-trip
- The more workers, the more total IPC load, not less
- Embarrassingly parallel tasks (no coordination) scale linearly
- Tasks requiring coordination: measure carefully — may not scale at all

---

## Slide 10 — The Crossover Point: Profile Data

**Multiprocessing wins above a threshold, loses below**

- Short tasks (1–5 ms): sequential nearly always wins
- Medium tasks (50–100 ms): measure both — outcome depends on IPC payload
- Long tasks (200 ms+): multiprocessing usually wins if payload is small
- The crossover is typically 10–50 ms on modern hardware
- Never assume — always profile sequential vs. parallel before committing

---

## Slide 11 — The Profiling Discipline

**Make data-driven architectural decisions**

- Write the sequential version first; measure it
- Create isolated benchmark: same task, same data, sequential vs. Pool with N workers
- If speedup < 1.2×: question whether the architecture is correct
- If sequential wins: try batching tasks to increase per-task compute time
- If batching doesn't help: consider whether the workload is actually CPU-bound

---

## Slide 12 — Lecture 26 Key Principles

**What to carry forward**

- IPC = three data copies: each byte in your task payload crosses the boundary three times
- Failure scenarios: task too short, data too large, too much inter-worker coordination
- `ProcessPoolExecutor` gives the same Future-based interface as `ThreadPoolExecutor` backed by processes
- Use `Pool` for fine-grained control; `ProcessPoolExecutor` for futures/asyncio composition
- Profile always — a speedup that doesn't appear in benchmarks won't appear in production
- Next: zero-copy NumPy sharing with shared memory — eliminating IPC cost for large arrays entirely

---
