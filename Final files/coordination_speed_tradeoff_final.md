
# Coordination and Speed Trade-off in Parallel Systems

## Overview

Parallel systems do not fail because of CPU limits.

They fail because of coordination cost.

Every time workers talk to each other,
the system slows down.

Multiprocessing performance is limited by:

> coordination overhead, not computation

This module explains why shared state reduces speed,
and how real systems minimize communication.

---

## Section 1 — Why Coordination Exists

Workers must coordinate to ensure:

- correctness
- consistency
- safe shutdown
- early exit
- shared results

Coordination mechanisms include:

- queues
- locks
- shared flags
- synchronization barriers

Each adds latency.

Safety competes with speed.

---

## Section 2 — Work Queue Pattern

Queues distribute tasks safely.

They enable dynamic load balancing.

```python
import multiprocessing as mp

def worker(q):
    while not q.empty():
        print("Working on", q.get())

if __name__ == "__main__":
    q = mp.Queue()
    for i in range(5):
        q.put(i)

    procs = [mp.Process(target=worker, args=(q,)) for _ in range(2)]

    for p in procs: p.start()
    for p in procs: p.join()
```

But queues require:

- pickling
- copying
- locks

If tasks are tiny → queue overhead dominates.

Parallel becomes slower than serial.

---

## Section 3 — Shared Flags and Early Exit

Cooperative search requires a shared stop signal.

Example: fraud detection engine.

If any worker finds fraud → stop all.

### Raw shared flag

```python
flag = mp.RawValue('b', 0)
```

Fast but unsafe.

### Manager.Value

Safe but slower.

Shared state always costs performance.

---

## Section 4 — Throughput vs Communication

Workers that communicate heavily:

- stall waiting
- serialize execution
- waste cores

Fast systems:

- partition work
- minimize sharing
- batch communication
- use immutable data

Less talking → more speed.

---

## Section 5 — Architectural Rule

The fastest multiprocessing systems:

avoid shared state entirely.

They use:

- independent tasks
- message passing
- isolated workers
- immutable inputs

Shared memory is last resort.

---

## Exercise — Coordination Benchmark

Compare:

1. pure parallel compute
2. queue-heavy architecture
3. shared lock architecture

Measure slowdown caused by coordination.

Explain results.

---

## Final Takeaways

Coordination is a tax on speed.

You trade:

correctness ↔ performance

Great parallel architecture:

- reduces shared state
- minimizes synchronization
- isolates workers
- communicates sparingly

The fastest systems talk less.
They compute more.
