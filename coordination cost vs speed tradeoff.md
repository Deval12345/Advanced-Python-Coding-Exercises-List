# IPC Coordination, Work Queues, and Shared Flags in Multiprocessing

Multiprocessing is not only about splitting work.

It is about **coordinating workers efficiently**.

This section explains:

- queues of work
- poison-pill shutdown patterns
- cooperative search
- shared flags
- Manager vs RawValue vs mmap vs Redis
- why synchronization costs dominate performance

This is where multiprocessing becomes systems engineering.

---

## Real-World Scenario: Fraud Detection Search

Imagine a fraud detection engine:

- millions of transactions
- workers scanning suspicious patterns
- if ANY worker finds fraud → stop immediately

We want:

> early exit when answer is found

But workers run in parallel.

They must coordinate.

This is a classic cooperative search problem.

---

## Queues of Work — Basic Pattern

Workers pull jobs from a queue.

Parent pushes tasks.

Queue = shared inbox.

This is dynamic load balancing.

---

## Prime Search Example (Work Queue Model)

This mirrors the book’s cooperative search design.

```python
import multiprocessing as mp
import math

FLAG_DONE = "STOP"
FLAG_WORKER_FINISHED = "WORKER_DONE"

def is_prime(n):
    if n < 2: return False
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

def worker(task_q, result_q):
    while True:
        n = task_q.get()
        if n == FLAG_DONE:
            result_q.put(FLAG_WORKER_FINISHED)
            break
        if is_prime(n):
            result_q.put(n)

if __name__ == "__main__":
    tasks = mp.Queue()
    results = mp.Queue()

    for n in range(100_000, 100_200):
        tasks.put(n)

    workers = [mp.Process(target=worker, args=(tasks, results)) for _ in range(4)]

    for w in workers: w.start()

    for _ in workers:
        tasks.put(FLAG_DONE)

    for w in workers: w.join()

    while not results.empty():
        print(results.get())
```

Key idea:

Queue ensures safe task distribution.

But:

Queues require pickling + locks.

They are not free.

---

## Poison Pill Pattern

We used:

```
STOP flag
```

This is called a **sentinel** or **poison pill**.

It signals workers to exit cleanly.

This is production-grade shutdown logic.

Used in:

- job schedulers
- message brokers
- streaming systems
- ETL pipelines

---

## Queue Cost Insight

Queues serialize objects.

This means:

- pickling cost
- copying cost
- locking cost

If tasks are tiny → queue overhead dominates.

Parallel becomes slower than serial.

Rule:

> queue overhead must be small relative to task cost

---

## Cooperative Early Exit with Shared Flag

Now we extend scenario:

If one worker finds fraud → everyone stops.

We need a shared flag.

---

## Manager.Value — Safe but Slow

Manager wraps Python objects.

Safe and synchronized.

But expensive.

### Example

```python
import multiprocessing as mp

FLAG_CLEAR = b'0'
FLAG_SET = b'1'

def worker(flag):
    if expensive_search():
        flag.value = FLAG_SET

if __name__ == "__main__":
    manager = mp.Manager()
    flag = manager.Value('c', FLAG_CLEAR)
```

Manager uses proxies.

Every access crosses process boundary.

Heavy overhead.

Good for flexibility.

Bad for speed.

---

## RawValue — Fast but Dangerous

RawValue skips synchronization.

Almost raw shared memory.

```python
flag = mp.RawValue('c', FLAG_CLEAR)
```

No locks.

Fast.

But:

race conditions possible.

Only safe when:

- flag only flips one direction
- correctness doesn’t depend on timing

Example: “stop search” signal.

---

## mmap — Near Zero Overhead Flag

Fastest shared byte mechanism.

Memory-mapped anonymous region.

```python
import mmap

shm = mmap.mmap(-1, 1)
shm.write(b'0')

# worker
shm.seek(0)
if shm.read(1) == b'1':
    exit_early()
```

This behaves like a 1-byte shared file.

No Python-level synchronization.

Almost zero overhead.

Used in:

- trading systems
- game engines
- high-frequency pipelines

---

## Redis — External Shared State

Redis is a networked memory store.

Processes talk via TCP.

```python
import redis

r = redis.StrictRedis()
r["flag"] = b"0"

if r["flag"] == b"1":
    exit()
```

Advantages:

- language independent
- cross-machine coordination
- observable system state

Disadvantages:

- network overhead
- slower than local memory

Used when:

- clustering
- monitoring required
- multi-language systems

---

## Performance Tradeoff Summary

| Method | Speed | Safety | Scope |
|-------|------|-------|------|
| Manager.Value | slow | safe | local |
| RawValue | fast | unsafe | local |
| mmap | fastest | unsafe | local |
| Redis | slowest | safe | distributed |

---

## Engineering Insight

Shared state adds:

- synchronization cost
- debugging complexity
- race risk
- architectural fragility

The fastest multiprocessing systems:

> avoid shared state entirely

Instead:

- independent tasks
- immutable inputs
- isolated outputs

Shared flags are last resort.

---

## Real System Design Rule

Prefer:

```
independent workers
stateless jobs
message passing
```

Avoid:

```
shared mutable state
fine-grained synchronization
tight coupling
```

This is why distributed systems adopt:

- actor models
- message queues
- event-driven design

---

## Final Principle

Multiprocessing performance is limited by:

> coordination overhead

Not CPU.

The more workers must talk:

the slower the system becomes.

The fastest systems:

talk less.

compute more.
