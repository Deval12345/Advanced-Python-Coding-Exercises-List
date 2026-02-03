
# Concurrency in Python — Futures, asyncio, and Event-Driven Design

## Overview

Python programs often spend more time waiting than computing.

Concurrency exists to hide latency — not just to add parallelism.

Different workloads require different models:

- I/O-bound → async / event loop
- CPU-bound → processes
- Mixed workloads → futures + executors

This module explains concurrency as architecture.

You will learn:

✔ I/O-bound vs CPU-bound work  
✔ Threads vs async vs processes  
✔ Futures abstraction  
✔ Event loop fundamentals  
✔ Cooperative multitasking  
✔ Structured concurrency patterns  

---

## Section 1 — Why Concurrency Exists

Most time is spent waiting for:

- network I/O
- disk I/O
- database responses
- external APIs

Sequential waiting wastes time.

Concurrency overlaps waiting.

Throughput increases without faster CPUs.

---

## Exercise 1 — Threads for I/O-bound Work

### Scenario

Download files concurrently and update shared progress.

```python
import threading
import time
import random

progress = 0
lock = threading.Lock()

def download(file_id):
    global progress
    time.sleep(random.uniform(0.1, 0.5))
    with lock:
        progress += 1
        print("Downloaded", file_id, "progress:", progress)

threads = [threading.Thread(target=download, args=(i,)) for i in range(10)]

for t in threads:
    t.start()

for t in threads:
    t.join()
```

### Lesson

✔ I/O overlaps  
✔ Shared memory is easy  
✔ Synchronization is required  

---

## Section 2 — Async Event Loop (I/O-heavy systems)

Async avoids blocking threads.

One event loop manages many tasks.

### Exercise 2 — Async Dashboard Aggregator

```python
import asyncio
import random

async def fetch(name):
    delay = random.uniform(0.5, 1.5)
    await asyncio.sleep(delay)
    return f"{name} ready"

async def dashboard():
    results = await asyncio.gather(
        fetch("profile"),
        fetch("orders"),
        fetch("recommendations"),
        fetch("notifications"),
    )
    print(results)

asyncio.run(dashboard())
```

Outcome:

✔ Total time ≈ slowest call  
✔ Waiting overlapped  
✔ Event loop coordinates tasks  

---

## Section 3 — Futures Replace Callback Chaos

Futures represent running tasks.

They decouple execution from result handling.

### Exercise 3 — Callback → Futures Refactor

```python
from concurrent.futures import ThreadPoolExecutor

def send_notification(name):
    return f"{name} sent"

with ThreadPoolExecutor() as executor:
    futures = [
        executor.submit(send_notification, "email"),
        executor.submit(send_notification, "sms"),
        executor.submit(send_notification, "push"),
    ]

    for f in futures:
        print(f.result())
```

Futures provide:

✔ centralized error handling  
✔ result management  
✔ cancellation ability  

---

## Section 4 — CPU-bound Work Requires Processes

Threads do not bypass the GIL.

CPU-heavy work needs process isolation.

### Exercise 4 — Parallel CPU Work

```python
from concurrent.futures import ProcessPoolExecutor
import math

def heavy(n):
    return sum(math.sqrt(i) for i in range(n))

with ProcessPoolExecutor() as executor:
    results = executor.map(heavy, [10_000_00] * 4)
    print(list(results))
```

Outcome:

✔ True parallelism  
✔ Memory isolation  
✔ Multi-core usage  

---

## Section 5 — Mixing Async + Executors

Async event loop stays responsive while CPU work runs elsewhere.

### Exercise 5 — Async + Process Executor

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_task(n):
    return sum(i*i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_task, 10_000_000)
        print("Result:", result)

asyncio.run(main())
```

Event loop remains free to serve requests.

---

## Section 6 — Background Job Orchestrator

High-level concurrency patterns.

```python
import asyncio

async def job(n):
    await asyncio.sleep(1)
    return f"job {n} done"

async def orchestrator():
    tasks = [job(i) for i in range(5)]
    results = await asyncio.gather(*tasks)
    print(results)

asyncio.run(orchestrator())
```

Fan-out → fan-in pattern made simple.

---

## Final Takeaways

Concurrency is about:

✔ Latency hiding  
✔ Structured task management  
✔ Correct model selection  
✔ Responsiveness under load  
✔ Scalable architecture  

Threads → simple I/O overlap  
Async → high-latency network systems  
Processes → CPU parallelism  
Futures → task abstraction

---

## Suggested Extensions

1. Build async web crawler
2. Implement job scheduler
3. Add timeout handling
4. Combine threads + async
5. Benchmark concurrency models

---

End of module.
