# Slides — Lecture 28: Sharing NumPy Data with Multiprocessing — Zero-Copy Design

---

## Slide 1 — Lecture Overview

**Zero-Copy Design — Making Large Arrays Work Across Processes**

- Why Queue.put(array) is catastrophic for large NumPy arrays
- numpy.frombuffer: creating a NumPy view of shared memory without copying
- multiprocessing.Array as backing storage for zero-copy NumPy views
- multiprocessing.shared_memory (Python 3.8+): cleaner, name-based approach
- When locks are needed and when they are not: read-only vs. partitioned writes

---

## Slide 2 — The Three-Copy Problem for NumPy Arrays

**What Queue.put(array) actually does**

- Copy 1: `pickle.dumps(array)` — dtype, shape, strides, all bytes → bytes buffer in sending process
- Copy 2: OS writes bytes through kernel pipe into pipe buffer
- Copy 3: `pickle.loads(bytes)` — reconstruct full NumPy array in worker process
- 100 MB array → 300 MB peak memory in transit + ~2 seconds of serialization
- At 30 arrays/second: 1.8 GB/sec pickling requirement — Python cannot sustain this
- The Queue is designed for small messages — not multi-megabyte arrays

---

## Slide 3 — The Zero-Copy Insight

**A NumPy array is bytes + metadata**

- NumPy array = raw bytes (data buffer) + metadata (dtype, shape, strides)
- If multiple processes see the SAME bytes (shared memory page), each can wrap them in its own NumPy object
- The array object (metadata) is tiny — kilobytes
- The data buffer is shared — no copy, no serialization, no IPC
- `numpy.frombuffer(buffer, dtype)` — wraps any buffer-protocol object as a NumPy array
- `multiprocessing.Array` implements the buffer protocol → direct frombuffer wrapping

---

## Slide 4 — multiprocessing.Array as NumPy Backing Storage

**ctypes shared memory with NumPy view**

```
Main process:
  sharedArray = multiprocessing.Array(ctypes.c_double, size)
  arr = np.frombuffer(sharedArray, dtype=np.float64)  # zero-copy view
  arr[:] = data                                         # write initial data

Worker process (receives sharedArray reference):
  arr = np.frombuffer(sharedArray, dtype=np.float64)  # own view, same bytes
  arr[start:end] = arr[start:end] ** 2                # writes to shared memory
```

- Workers receive the Array handle (bytes, not data)
- Each worker creates its own lightweight NumPy view
- Writes from workers are immediately visible to main process — no result transfer

---

## Slide 5 — Example 28.1: In-Place Squaring via Shared Memory

**Four workers, one array, zero copies**

- 1,000,000 float64 elements → 8 MB of shared memory
- Each worker squares its assigned chunk in-place: `arr[start:end] **= 2`
- Non-overlapping chunks → no locks needed
- After join: main process's `frombuffer` view reflects all changes immediately
- Total IPC: process startup arguments only (handles, not data)
- Compare to Queue approach: 24 MB of serialization overhead for same operation

---

## Slide 6 — multiprocessing.shared_memory (Python 3.8+)

**Named raw bytes — cleaner than ctypes Array**

- `shm.SharedMemory(create=True, size=data.nbytes)` — allocate named block
- `np.ndarray(shape, dtype=dtype, buffer=sharedMem.buf)` — wrap as NumPy array
- Workers receive: name string (str), shape, dtype — all tiny IPC payloads
- Worker attaches: `shm.SharedMemory(name=sharedName)` → same physical bytes
- No ctypes type codes — full NumPy dtype support including structured dtypes
- Name-based access: works across unrelated processes (not just Pool workers)

---

## Slide 7 — shared_memory Lifecycle

**Create, use, close, unlink — in that order**

- `create=True` → allocates new block, assigns OS-managed name
- `.name` → the string identifier workers use to attach
- `existing.close()` → releases this process's mapping (does NOT free memory)
- `sharedMem.unlink()` → destroys the OS-level resource (call once, in creator)
- Forgetting `unlink()` → ghost shared memory block in OS registry until reboot
- Pattern: `try: ... finally: sharedMem.close(); sharedMem.unlink()` in main process

---

## Slide 8 — Example 28.2: Zero-Copy Analysis with shared_memory

**Workers read, compute, return only statistics**

- 2,000,000 float64 elements → 16 MB of shared memory, allocated once
- Workers attach by name — no data transfer to workers
- Each worker computes: mean, std, max of its chunk
- Workers send back: small statistics dict (< 200 bytes each) through Queue
- Total IPC payload: 4 × 200 bytes = 800 bytes vs. 16 MB without shared memory
- After join: `sharedMem.close()` + `sharedMem.unlink()` — clean OS resource release

---

## Slide 9 — Lock Rules for Shared Memory

**When to lock and when not to**

| Access Pattern | Lock Needed? | Reason |
|---|---|---|
| All workers read-only | No | Concurrent reads are always safe |
| Workers write non-overlapping chunks | No | Different memory addresses — no conflict |
| Workers write overlapping regions | Yes | Data race — both writes to same address |
| One writer, many readers | Yes (reader-writer lock) | Writer must be atomic |

- Partitioned writes: assign each worker a non-overlapping index range → no locks
- Reductions: each worker writes to separate output slot → no locks in hot path

---

## Slide 10 — Production Design: The Partitioned Pattern

**Divide, process, aggregate — no locks in the hot path**

```
Main process:
  data = SharedMemory(size)       # one allocation
  results = Array(float, numWorkers)  # one result per worker

Workers:
  view = frombuffer(data)         # zero-copy
  results[workerId] = compute(view[start:end])  # partitioned write

Main process after join:
  finalResult = aggregate(results[:])  # combine
```

- Data: shared, read (or partitioned write) — no locks
- Results: one slot per worker — no locks
- Aggregation: sequential in main process — no concurrency issue

---

## Slide 11 — multiprocessing.Array vs. shared_memory: Which to Use

**Practical decision guide**

| Criterion | `multiprocessing.Array` | `multiprocessing.shared_memory` |
|---|---|---|
| Python version | Any | 3.8+ |
| Type system | ctypes types | Raw bytes + NumPy dtype |
| Structured dtypes | Awkward | Fully supported |
| Cross-process name lookup | No | Yes (`name` attribute) |
| Built-in lock | Yes | No — manual |
| API complexity | Low | Slightly higher (lifecycle management) |

- Default to `shared_memory` for Python 3.8+; use `Array` for compatibility or simple scalar types

---

## Slide 12 — Lecture 28 Key Principles

**What to carry forward**

- Never send large NumPy arrays through a Queue — three copies, seconds of overhead
- `numpy.frombuffer` wraps any buffer-protocol object as a zero-copy NumPy view
- `multiprocessing.Array` + `frombuffer`: pre-3.8 zero-copy sharing with ctypes backing
- `multiprocessing.shared_memory`: modern, dtype-agnostic, name-based large array sharing
- Lock rules: reads always safe; partitioned writes always safe; overlapping writes require locks
- Design: data in shared memory, IPC carries only names, indices, and statistics
- Next: async + multiprocessing hybrid — event loop front-end, process pool back-end

---
