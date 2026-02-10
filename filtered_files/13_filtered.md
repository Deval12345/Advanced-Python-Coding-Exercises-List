
# Concurrency in Python

## Overview

Python programs spend most time waiting, not computing.

Concurrency exists to hide latency — not just to add parallelism.

Different waiting patterns require different models:

- I/O-bound → threads or async
- CPU-bound → processes
- high-latency I/O → event-driven async
- mixed workloads → futures + executors

Concurrency is an architectural decision.

---

## Section 1 — I/O-bound vs CPU-bound Work

I/O-bound work waits on external systems:

- network calls
- disk reads
- APIs
- databases

CPU-bound work consumes processor time:

- heavy math
- simulations
- encoding
- parsing large datasets

Threads help I/O.
Processes help CPU.
Async scales high-latency I/O.

Choosing wrong model wastes performance.

---

## Exercise 1 — Threads (I/O-bound Shared State)

Download files concurrently and track shared progress.

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

Threads overlap waiting.
Shared memory requires synchronization.

---

## Section 2 — Async Event Loop

Async avoids blocking threads.

One event loop coordinates many tasks.

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
        fetch("notifications"),
    )
    print(results)

asyncio.run(dashboard())
```

Total time ≈ slowest task.
Waiting overlaps automatically.

---

## Exercise 2 — Async High-Latency Fetch

Fetch hundreds of URLs without spawning hundreds of threads.

Explain why async scales better.

---

## Section 3 — Futures Abstraction

Futures represent running tasks.

They decouple execution from result handling.

```python
from concurrent.futures import ThreadPoolExecutor

def send(msg):
    return f"{msg} sent"

with ThreadPoolExecutor() as pool:
    futures = [
        pool.submit(send, "email"),
        pool.submit(send, "sms"),
    ]

    for f in futures:
        print(f.result())
```

Futures centralize result handling.

---

## Section 4 — CPU-bound Work Needs Processes

Threads do not bypass the GIL.

CPU-heavy work requires process isolation.

```python
from concurrent.futures import ProcessPoolExecutor
import math

def heavy(n):
    return sum(math.sqrt(i) for i in range(n))

with ProcessPoolExecutor() as pool:
    results = pool.map(heavy, [1_000_000] * 4)
    print(list(results))
```

True multi-core parallelism.

---

## Exercise 3 — Process Parallelism

Measure speedup for CPU-heavy computation.

Explain why threads do not help here.

---

## Final Takeaways

Concurrency is about:

- latency hiding
- responsiveness
- architecture choices
- workload classification

Threads → simple I/O overlap
Async → scalable network systems
Processes → CPU parallelism
Futures → structured task management

Correct model selection matters more than syntax.
