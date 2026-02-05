
# Stage 5 — Caching & Memory Discipline: Stabilizing the Pipeline

Our system is now scalable and measurable.

But under sustained load, another issue appears:

We are recomputing expensive work repeatedly.

And memory usage grows over time.

Stage 5 introduces:

- caching architecture
- memoization discipline
- bounded memory strategy
- object lifetime control

This stage transforms the pipeline from fast
to sustainable.

---

## Real-World Scenario

The recommendation engine repeatedly scores:

- the same user
- the same product
- the same ranking query

Workers recompute identical results.

CPU usage explodes.

Meanwhile:

temporary buffers accumulate
and memory drifts upward.

The system becomes unstable after hours.

We must:

✔ avoid repeated work  
✔ bound memory growth  
✔ preserve correctness  

---

# Part 1 — Stage-Level Caching

We add caching to expensive stages.

But caching must be safe.

---

## Cached Stage Wrapper

```python
from functools import lru_cache

class CachedStage:
    def __init__(self, stage):
        self.stage = stage

    @lru_cache(maxsize=128)
    def cached_process(self, key):
        event = {"user": key}
        return self.stage.process(event)

    def process(self, event):
        return self.cached_process(event["user"])
```

Now repeated inputs reuse results.

CPU load drops dramatically.

---

# Part 2 — Cache Safety Rule

Caching must guarantee:

same input → same output

Otherwise correctness breaks.

Stages must be deterministic.

Architecture must enforce this.

---

# Part 3 — Memory Discipline

Long-running systems fail from slow accumulation.

We must avoid:

- global lists
- unbounded buffers
- hidden retention

Streaming design prevents growth.

---

## Streaming Pattern

```python
def stream_stage():
    for i in range(1_000_000):
        yield i * 2

total = sum(stream_stage())
```

No giant list.

Memory stays flat.

---

# Part 4 — Bounded Cache Architecture

Caches must be limited.

Unlimited caches become leaks.

```python
@lru_cache(maxsize=256)
def score(user):
    return len(user) * 2
```

Bounded cache = controlled memory.

---

# Part 5 — Memory Visibility Integration

We monitor memory continuously.

```python
import tracemalloc

tracemalloc.start()
pipeline.run({"user": "alice"})
current, peak = tracemalloc.get_traced_memory()
print("Memory:", current, "Peak:", peak)
```

Memory becomes observable.

Architecture stays stable.

---

# Exercises

1. Add caching to ranking stage.
2. Measure CPU reduction.
3. Simulate memory leak and detect it.
4. Replace list accumulation with streaming.

---

# Final Insight

Caching is not optimization.

It is architecture.

Memory discipline prevents slow collapse.

Stage 5 makes the system sustainable.
