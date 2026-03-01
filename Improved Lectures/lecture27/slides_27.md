# Slides — Lecture 27: Coordination Speed Trade-off in Multiprocessing

---

## Slide 1 — Lecture Overview

**Coordination — The Hidden Cost of Parallel Decisions**

- How multiple workers signal each other to stop or surrender a task
- The poison-pill pattern: using the data channel for graceful shutdown
- Cooperative early-exit: shared flag with multiprocessing.Value
- Check frequency as a performance design variable
- Manager objects vs. Value vs. Array: when to use each

---

## Slide 2 — The Poison-Pill Shutdown Pattern

**Using the data channel to carry the control signal**

- Problem: how do you tell queue workers to stop gracefully?
- Poison-pill: put `None` (sentinel) into the Queue — one per worker
- Worker reads `None` → exits cleanly, no work lost, no shared state corrupted
- Ordering guarantee: tasks arrive before sentinel — Queue is FIFO
- No Manager, no extra lock, no separate channel needed
- Used in: Celery, Kafka consumers, Redis queue processors

---

## Slide 3 — Cooperative Early-Exit: The Problem

**"First found wins" — what happens to the rest?**

- Search, optimization, fraud detection: any worker may find the answer
- Without early-exit: all workers run to completion even after result is found
- Four workers, each searching 100,000 items: result found at position 1,000
- Three workers waste 99,000 unnecessary iterations each → 297,000 wasted iterations
- Goal: workers voluntarily stop the moment another worker succeeds

---

## Slide 4 — multiprocessing.Value as a Shared Flag

**Shared memory scalar — no serialization, nanosecond reads**

- `multiprocessing.Value('b', False)` — boolean in shared memory (type code 'b')
- All processes sharing the Value see the same physical memory page
- Read: `foundFlag.value` — memory access + optional memory barrier
- Write safely: `with foundFlag.get_lock(): foundFlag.value = True`
- Critical pattern: acquire lock → check flag again → act → release
- Double-check prevents two workers both writing simultaneously

---

## Slide 5 — Example 27.1: Cooperative Prime Search

**Lock-check-set-return sequence**

```
Worker receives: workerId, searchRange, foundFlag, resultQueue
For each n in range:
    if foundFlag.value: return immediately   ← cheap flag read
    if isPrime(n):
        with foundFlag.get_lock():           ← acquire lock
            if not foundFlag.value:          ← double-check inside lock
                foundFlag.value = True       ← set flag
                resultQueue.put(result)      ← record result
        return
```

- The double-check inside the lock is mandatory — prevents duplicate results
- Only the first worker through the lock writes the result

---

## Slide 6 — Check Frequency as a Design Variable

**The responsiveness vs. throughput trade-off**

- Every flag check costs: memory access + memory barrier (~10–100 ns)
- Check every iteration: pays cost once per loop cycle
- Check every 1,000 iterations: amortizes cost over 1,000 units of computation
- **Frequent check:** high responsiveness, potential throughput loss in tight loops
- **Rare check:** high throughput, delayed response to stop signal
- Rule: check frequency ∝ 1 / (iteration cost)

---

## Slide 7 — Example 27.2: Measuring Coordination Cost

**Same work, different check intervals — measured difference**

- Frequent worker: checks `sharedFlag.value` every 10 iterations
- Rare worker: checks `sharedFlag.value` every 1,000 iterations
- Same prime-counting computation, same 4 workers, same workload
- Measured time difference = coordination overhead of the more frequent checker
- In tight loops (cheap iterations): frequent checking is measurably slower
- In heavy loops (expensive iterations): difference is negligible

---

## Slide 8 — The Shared-State Cost Spectrum

**Not all shared state costs the same**

| Mechanism | Access Latency | Flexibility | Lock Required |
|---|---|---|---|
| `Manager.dict` / `.list` | 1–5 ms (IPC round-trip) | Arbitrary Python objects | Manager handles it |
| `multiprocessing.Value` | ~100 ns (memory access) | Single scalar | Optional (built-in lock) |
| `multiprocessing.Array` | ~100 ns per element | Fixed-size typed array | Optional (built-in lock) |
| `shared_memory` | ~100 ns | Raw bytes / NumPy view | Manual |

- Manager: 1,000–50,000× slower per access than Value
- Use Manager for flexibility; use Value/Array for performance

---

## Slide 9 — Manager vs. Value: Decision Framework

**Match the mechanism to the access pattern**

- **Use Manager.dict / Manager.list when:**
  - Data is complex, variable-shape, or holds arbitrary Python objects
  - Accessed infrequently (once per task, not once per iteration)
  - Correctness is easier to reason about than performance

- **Use multiprocessing.Value when:**
  - Single scalar: flag, counter, threshold, timestamp
  - Accessed frequently (every iteration or every few iterations)
  - Lock discipline is manageable with a small number of writers

- **Use multiprocessing.Array when:**
  - Fixed-size numeric data: result buffer, histogram, accumulator
  - Multiple workers each write to a distinct slice (no lock needed)

---

## Slide 10 — Fraud Detection Analogy

**Real-time early termination in production systems**

- Four workers analyze parallel transaction streams
- Worker 1 identifies a fraudulent pattern → sets shared flag
- Workers 2, 3, 4: at next flag check → abandon current analysis
- All four workers immediately begin processing the next transaction
- Check frequency determines: how quickly system responds to the fraud signal
- Too infrequent: workers waste analysis time on invalid transactions
- Too frequent: coordination cost reduces per-worker analysis throughput

---

## Slide 11 — Worker Pool Design for Early Termination

**Structural requirements for cooperative exit**

1. Shared flag: `multiprocessing.Value('b', False)` — one per pool
2. Check interval: defined as a parameter — tuned per workload
3. Lock protocol: acquire → check again → act → release (always double-check)
4. Result channel: `multiprocessing.Queue()` — one result per worker maximum
5. Cleanup: all workers joined before main reads results — no zombie processes
6. Verification: check `not resultQueue.empty()` before reading — handle no-result case

---

## Slide 12 — Lecture 27 Key Principles

**What to carry forward**

- Poison-pill: put sentinels into the Queue — one per worker — for ordered, clean shutdown
- Cooperative early-exit: `multiprocessing.Value` shared flag, lock-check-set, double-check inside lock
- Check frequency is a performance parameter: calibrate to iteration cost vs. lock cost
- Manager objects = IPC round-trip per access = milliseconds — use for infrequent, flexible state
- Value / Array = shared memory access = nanoseconds — use for frequent, fixed-layout state
- Next: zero-copy NumPy sharing — multiprocessing.Array as NumPy backing storage

---
