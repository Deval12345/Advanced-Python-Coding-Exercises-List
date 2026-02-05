
# Stage 4 — Profiling & Performance Engineering the Pipeline

Our system is now modular, safe, and concurrent.

But under real production load, a new problem appears:

We don’t know where time is being spent.

Some workers are slow.
Some stages dominate CPU.
Memory usage grows unpredictably.

We cannot optimize blindly.

Stage 4 introduces performance engineering as a discipline.

We integrate profiling directly into the architecture.

---

## Real-World Scenario

Traffic doubles after a product launch.

Symptoms:

- queue backlog increases
- latency spikes
- CPU saturates
- workers fall behind

Engineers guess:

“Fraud stage is slow.”
“No, ranking is slow.”
“No, queue is broken.”

Guessing wastes time.

We must measure.

---

# Part 1 — Profiling the Pipeline

We profile stage execution individually.

Each stage becomes measurable.

---

## Timed Stage Wrapper

```python
import time

class TimedStage:
    def __init__(self, stage):
        self.stage = stage

    def process(self, event):
        start = time.time()
        result = self.stage.process(event)
        duration = time.time() - start
        print(f"{self.stage.__class__.__name__}: {duration:.6f}s")
        return result
```

We wrap any stage:

```python
pipeline = Pipeline([
    TimedStage(RankingStage()),
    TimedStage(FraudStage())
])
```

Now we see real cost.

Architecture exposes performance.

---

# Part 2 — CPU Profiling Workers

We profile full worker execution.

```python
import cProfile

def profiled_worker(q, pipeline):
    def run():
        pipeline_worker(q, pipeline)

    cProfile.runctx("run()", globals(), locals())
```

This reveals:

- hot functions
- call frequency
- CPU hotspots

We optimize based on evidence.

---

# Part 3 — Memory Visibility

We monitor memory growth.

```python
import tracemalloc

tracemalloc.start()

pipeline_worker(q, pipeline)

current, peak = tracemalloc.get_traced_memory()
print("Memory:", current, "Peak:", peak)
```

This exposes hidden retention.

Long-running pipelines must stay memory stable.

---

# Part 4 — Benchmark Harness

We build repeatable tests.

```python
import time

def benchmark(pipeline, runs=5):
    durations = []
    for _ in range(runs):
        start = time.time()
        pipeline.run({"user": "alice", "amount": 100})
        durations.append(time.time() - start)
    return sum(durations) / len(durations)

print("Average:", benchmark(pipeline))
```

Benchmarks prevent regression.

Performance becomes testable.

---

# Part 5 — Architectural Performance Rule

Never optimize unmeasured code.

Measured architecture:

- reduces complexity
- prevents wasted effort
- stabilizes scaling decisions

Profiling becomes a permanent layer.

Not a debugging hack.

---

# Exercises

1. Wrap every stage with TimedStage.
2. Identify the slowest stage.
3. Create a synthetic heavy stage and profile it.
4. Build a benchmark that fails if latency increases.

---

# Final Insight

Performance engineering is architecture.

Systems that scale successfully:

measure continuously  
optimize intentionally  
verify improvements  

Stage 4 makes performance visible.
