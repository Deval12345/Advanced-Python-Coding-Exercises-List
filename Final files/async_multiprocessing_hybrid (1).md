
# Async + Multiprocessing Hybrid Architecture — Scaling I/O and CPU Together

Modern high-performance systems rarely rely on a single concurrency model.

Real production pipelines must handle:

- massive I/O concurrency
- heavy CPU computation
- safe coordination
- scalable throughput

Async alone cannot scale CPU.
Multiprocessing alone cannot scale network concurrency.

The winning architecture combines both.

This document explains hybrid design using a realistic system scenario.

---

## Real-World Scenario: Large-Scale Web Scraping + ML Processing

Imagine a data company that:

- scrapes 100,000 web pages per hour
- extracts product data
- runs ML classification
- stores analytics results

The workload splits naturally:

I/O-bound stage → network scraping  
CPU-bound stage → ML processing

If we use only async:

CPU tasks block the event loop.

If we use only multiprocessing:

network efficiency collapses.

We must combine architectures.

---

## Architecture Diagram

```
Async Scraper → Task Queue → Worker Pool → Database
```

Async handles network scale.
Workers handle CPU load.

---

## Hybrid Implementation Example

```python
import asyncio
import multiprocessing as mp
import aiohttp
import random

def cpu_worker(task_q):
    while True:
        item = task_q.get()
        if item is None:
            break
        # simulate CPU-heavy ML
        result = sum(i*i for i in range(50_000))
        print("Processed:", item, result)

async def fetch(url, task_q):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as r:
            text = await r.text()
            task_q.put(len(text))

async def main(task_q):
    urls = ["https://example.com"] * 10
    tasks = [fetch(u, task_q) for u in urls]
    await asyncio.gather(*tasks)

if __name__ == "__main__":
    task_q = mp.Queue()

    workers = [mp.Process(target=cpu_worker, args=(task_q,)) for _ in range(4)]
    for w in workers:
        w.start()

    asyncio.run(main(task_q))

    for _ in workers:
        task_q.put(None)

    for w in workers:
        w.join()
```

Async stage feeds CPU workers efficiently.

This architecture scales both network and compute.

---

## Why Hybrid Wins

Async stage:

- thousands of connections
- minimal memory
- no thread explosion

Worker pool:

- true CPU parallelism
- multi-core utilization
- heavy computation isolated

They complement each other.

---

## Engineering Insight

Real systems separate:

I/O orchestration  
CPU execution

This prevents resource contention.

Event loops remain responsive.
Workers stay compute-focused.

---

## Systems Using Hybrid Architecture

This pattern powers:

- large web crawlers
- streaming analytics engines
- AI preprocessing farms
- ETL pipelines
- recommendation systems
- log ingestion platforms
- real-time fraud detection

Almost every modern data platform.

---

## Final Takeaway

Async scales waiting.
Multiprocessing scales computation.

Hybrid architecture scales systems.
