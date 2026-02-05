
# Stage 3 — Concurrency Layer: Scaling the Pipeline Safely

In Stage 1 we built modular architecture.
In Stage 2 we added lifecycle safety.

Now the system faces real pressure:

traffic increases.

One pipeline instance cannot keep up.

We must scale processing.

But naive concurrency introduces:

- race conditions
- corrupted state
- deadlocks
- resource explosions

Stage 3 introduces safe concurrency architecture.

We combine:

- producer–consumer pipelines
- multiprocessing workers
- async ingestion
- hybrid scaling model

This mirrors real analytics backends.

---

## Real-World Scenario

Our recommendation backend now receives:

- 50,000 events per second
- burst traffic spikes
- unpredictable load

Single-threaded pipeline collapses.

We must:

✔ parallelize processing  
✔ isolate workers  
✔ maintain correctness  
✔ prevent overload  

The architecture must scale safely.

---

# Part 1 — Producer–Consumer Model

We separate:

ingestion → processing

Events enter a queue.
Workers consume independently.

This removes shared state.

---

## Basic Queue Pipeline

```python
import multiprocessing as mp

def producer(q):
    for i in range(10):
        q.put({"event": i})
    q.put(None)

def worker(q):
    while True:
        event = q.get()
        if event is None:
            break
        print("Processed:", event)

if __name__ == "__main__":
    q = mp.Queue()

    p = mp.Process(target=producer, args=(q,))
    w = mp.Process(target=worker, args=(q,))

    p.start(); w.start()
    p.join(); w.join()
```

Workers operate independently.

No shared global memory.

---

# Part 2 — Worker Pool Scaling

We extend to multiple workers.

```python
workers = [mp.Process(target=worker, args=(q,)) for _ in range(4)]
```

Each worker runs the pipeline.

Throughput scales with CPU cores.

Architecture remains modular.

---

# Part 3 — Integrating Our Pipeline Engine

We embed Stage 1 engine inside workers.

```python
def pipeline_worker(q, pipeline):
    while True:
        event = q.get()
        if event is None:
            break
        result = pipeline.run(event)
        print("Output:", result)
```

Workers reuse modular pipeline safely.

No redesign required.

This is architectural payoff.

---

# Part 4 — Async Ingestion Layer

Event ingestion is I/O-bound.

We use async for network handling.

```python
import asyncio

async def ingest(q):
    for i in range(10):
        await asyncio.sleep(0.1)
        q.put({"event": i})
```

Async handles thousands of incoming events
without spawning threads.

Workers handle CPU.

Hybrid model achieved.

---

# Part 5 — Backpressure Protection

Queues must be bounded.

Otherwise memory explodes.

```python
q = mp.Queue(maxsize=5)
```

When full:

producer blocks.

System stabilizes.

This prevents meltdown under spikes.

---

# Part 6 — Graceful Shutdown

Workers must exit cleanly.

We use sentinel signals.

```python
for _ in workers:
    q.put(None)
```

This avoids zombie processes.

Production systems must shut down predictably.

---

# Exercises

1. Increase worker count and measure throughput.
2. Add artificial delay to simulate heavy stages.
3. Implement bounded queue backpressure.
4. Add metrics for queue depth.

---

# Final Insight

Concurrency is architecture.

Not just speed.

Safe scaling requires:

isolation  
message passing  
bounded queues  
controlled lifecycle  

Stage 3 transforms our pipeline
into a scalable system.
