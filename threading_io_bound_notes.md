# Threads in CPython — Why They Matter for I/O

In CPython, threads do not provide true CPU parallelism because of the Global Interpreter Lock (GIL). However, threads are still extremely powerful for real systems dominated by waiting.

The key idea: most modern programs spend more time waiting on networks, disks, and databases than doing math.

Threads allow your program to overlap waiting time. While one thread waits for an external resource, another thread runs.

This reduces total wall-clock time and makes programs feel responsive and concurrent.

---

## Detailed Real-World Scenario: Microservice API Aggregator

Imagine you are building a dashboard backend for a company.

The dashboard shows live business metrics.

To build the dashboard response, your server must query 8 different microservices:

- user analytics service
- billing service
- recommendation engine
- inventory database
- logging pipeline
- fraud detection system
- notification service
- reporting engine

Each service responds in about **1 second**.

A single-threaded program calls them one by one.

**Total time ≈ 8 seconds**

Users see a slow dashboard.

With threads, all requests run at the same time.

**Total time ≈ 1 second**

The CPU is mostly idle.

The bottleneck is network waiting.

Threads hide this latency.

---

## Sequential Version — Slow Dashboard

This simulates calling external services one at a time.

```python
import time

def call_service(i):
    print(f"Calling service {i}")
    time.sleep(1)  # simulate network delay
    print(f"Service {i} done")

start = time.time()

for i in range(8):
    call_service(i)

print("Sequential time:", round(time.time() - start, 2))
```

---

## Threaded Version — Concurrent Requests

Now we call services concurrently.

While one thread waits on network, others execute.

```python
import threading
import time

def call_service(i):
    print(f"Calling service {i}")
    time.sleep(1)
    print(f"Service {i} done")

threads = []
start = time.time()

for i in range(8):
    t = threading.Thread(target=call_service, args=(i,))
    t.start()
    threads.append(t)

for t in threads:
    t.join()

print("Threaded time:", round(time.time() - start, 2))
```

---

## Architectural Insight

Threads are not about speeding up computation.

They are about **hiding latency from slow external systems**.

This is why threads power:

- web servers
- chat systems
- data pipelines
- monitoring tools
- API gateways

Rule of thumb:

> If your system waits more than it computes → threads help  
> If your system computes more than it waits → multiprocessing helps
