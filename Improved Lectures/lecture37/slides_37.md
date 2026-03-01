# Slides — Lecture 37: Big Project Stage 4 — Measurable Pipeline (Profiling and Metrics)

---

## Slide 1 — Stage 4 Goal: Observable Pipeline

**Adding Measurement Infrastructure to the Analytics Engine**

- Problem: pipeline works but is opaque — no visibility into performance
- Stage 4 adds observability at two timescales:
  - Development-time: cProfile for offline hotspot identification
  - Production-time: @measuredStage decorator for continuous per-stage metrics
- Memory discipline: MetricsCollector with __slots__ keeps infrastructure lean
- Tools: cProfile, pstats, functools.wraps, tracemalloc, __slots__

---

## Slide 2 — The Optimization Without Measurement Trap

**Why intuition-based optimization fails — Amdahl's Law**

Scenario: pipeline takes 30 seconds; developer "optimizes" the feature stage (looks complex)
- Feature stage: 3% of runtime → optimized 40% faster → saves 0.9 seconds
- Actual bottleneck: `sum(window)` called per record (85% of feature stage time)
- Result: 2 days of work → 3% total speedup

**Rule:** Never write optimization code without profiling first.

| Stage | Runtime % | Fix if improved 2× | System speedup |
|-------|-----------|---------------------|----------------|
| Filter | 5% | 2.6% | Not worth it |
| Normalize | 10% | 5.3% | Moderate |
| Feature (sum bug) | **85%** | **46%** | **Optimize this** |

---

## Slide 3 — cProfile: The Deterministic Profiler

**Maps every function call to its time contribution**

```python
profiler = cProfile.Profile()
profiler.enable()
result = fullPipeline(50000)      # the code you want to measure
profiler.disable()

stats = pstats.Stats(profiler, stream=stream)
stats.sort_stats("cumulative")    # by total time including callees
stats.print_stats(8)              # top 8 functions
```

Output columns:
- `ncalls` — how many times function was called
- `tottime` — time in this function (excluding callees)
- `cumtime` — time in this function + all functions it calls
- **Sort by `cumtime` to find the expensive call chains**

Cost: 2–5× slowdown during profiling → run offline, not in production

---

## Slide 4 — Example 37.1: Profiling the Sensor Pipeline

**computeFeatures dominates — sum(window) is the bottleneck**

```python
def computeFeatures(records, windowSize):
    window = deque(maxlen=windowSize)
    results = []
    for rec in records:
        window.append(rec["value"])
        mean = sum(window) / len(window)           # O(windowSize) per record
        variance = sum((x - mean)**2 for x in window) / len(window)  # O(windowSize)
        results.append({**rec, "mean": round(mean, 6), "variance": round(variance, 6)})
    return results
```

Expected profiler output (50,000 records, window=10):
- `computeFeatures`: ~90% of cumtime
- `sum` called ~100,000 times within it

**Fix:** Maintain running sum — subtract leaving element, add entering element:
```python
runningSum += newValue - (droppedValue if full else 0)
mean = runningSum / len(window)
```

---

## Slide 5 — Stage Metrics: Production-Time Observability

**The @measuredStage decorator — continuous, zero-modification metrics**

```python
def measuredStage(stageName):
    def decorator(processMethod):
        @functools.wraps(processMethod)
        def wrapper(self, inputStream):
            metrics = StageMetrics(stageName=stageName)
            startTotal = time.perf_counter()
            for record in processMethod(self, inputStream):
                metrics.recordsIn += 1
                recStart = time.perf_counter()
                yield record                       # suspend: downstream runs
                metrics.recordsOut += 1
                metrics.latenciesMs.append(...)    # resume: measure downstream time
            metrics.totalTimeMs = ...
            print(metrics.report())
        return wrapper
    return decorator
```

- Applied at class definition: `@measuredStage("normalize")` on the stage method
- Zero modification to stage's business logic
- `@functools.wraps`: stack traces show original method name, not "wrapper"

---

## Slide 6 — What the Timing Point Measures

**The latency after yield captures the DOWNSTREAM stage's time**

```
Stage A (Filter)               Stage B (Normalizer)
    │                                  │
    ├── recStart = perf_counter()       │
    │                                  │
    ├── yield record ─────────────────► ├── receives record
    │                                  │    processes it...
    │   (suspended, waiting)           │
    │ ◄─────────────────────────────── ├── requests next record
    ├── recEnd = perf_counter()         │
    └── latency = recEnd - recStart     │
```

- Filter's measured "latency" = time Normalizer took to process its record
- High filter latency → downstream is slow, not filter itself
- The decorator finds pipeline bottlenecks; cProfile finds function bottlenecks

---

## Slide 7 — Example 37.2: StageMetrics Dataclass

**StageMetrics: accumulates counts, latencies, derived statistics**

```python
@dataclass
class StageMetrics:
    stageName: str
    recordsIn: int = 0
    recordsOut: int = 0
    totalTimeMs: float = 0.0
    latenciesMs: List[float] = field(default_factory=list)

    @property
    def throughputRps(self):
        return (self.recordsOut / (self.totalTimeMs / 1000)) if self.totalTimeMs > 0 else 0

    @property
    def meanLatencyMs(self):
        return sum(self.latenciesMs) / len(self.latenciesMs) if self.latenciesMs else 0
```

- `recordsIn` / `recordsOut`: filter ratio visible (recordsIn - recordsOut = records dropped)
- `throughputRps`: records per second — key capacity metric
- `meanLatencyMs`: from the downstream-time measurement; reveals downstream bottleneck

---

## Slide 8 — MetricsCollector with __slots__

**Memory-efficient accumulator: __slots__ replaces __dict__**

```python
class MetricsCollectorSlots:
    __slots__ = ("stageName", "count", "totalMs", "recentLatencies")
    #                ↑ no __dict__ — fixed C-level slot array instead

    def __init__(self, stageName):
        self.stageName = stageName
        self.count = 0
        self.totalMs = 0.0
        self.recentLatencies = deque(maxlen=100)
```

Memory comparison for 10,000 instances:
- `MetricsCollectorDict`: ~250–350 bytes/instance (~3.2 MB total)
- `MetricsCollectorSlots`: ~80–120 bytes/instance (~1.0 MB total)
- **Reduction: 2–3× less memory**

Bonus: `self.totlaTime = x` (typo) → `AttributeError` immediately with __slots__

---

## Slide 9 — Example 37.3: tracemalloc Memory Profiling

**Quantifying memory usage of infrastructure objects**

```python
tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()
dictCollectors = [MetricsCollectorDict(f"stage_{i}") for i in range(10000)]
snapshot2 = tracemalloc.take_snapshot()
slotsCollectors = [MetricsCollectorSlots(f"stage_{i}") for i in range(10000)]
snapshot3 = tracemalloc.take_snapshot()

# Compare snapshots to measure allocation between them
stats = snapshot2.compare_to(snapshot1, "lineno")
```

- `tracemalloc.take_snapshot()` — records current allocation state (file + line)
- `snapshot2.compare_to(snapshot1, "lineno")` — returns StatisticDiff per line
- Sum `size_diff` for total bytes allocated between snapshots
- Use to confirm __slots__ savings, catch unexpected allocations, monitor GC pressure

---

## Slide 10 — Stage 4 Summary: What Measurability Adds

**Before Stage 4 vs. After Stage 4**

| Aspect | Before Stage 4 | After Stage 4 |
|--------|---------------|---------------|
| Bottleneck identification | Guesswork / inspection | cProfile → data-driven |
| Per-stage throughput | Unknown | @measuredStage reports rps |
| Per-stage latency | Unknown | @measuredStage reports meanMs |
| Filter effectiveness | Unknown | recordsIn vs recordsOut ratio |
| Memory of infrastructure | Unknown | tracemalloc snapshot comparison |
| Metrics object memory | __dict__ (heavy) | __slots__ (3× lighter) |

**The discipline:** measure first → optimize the hotspot → measure again to confirm.

---

## Slide 11 — Lecture 37 Key Principles

**What to carry into Stage 5**

- cProfile: run offline before any optimization; sort by cumtime; fix the 80% contributor
- @measuredStage: wraps generator, measures downstream latency, reports throughput
- The timing trick: latency after `yield` = downstream processing time (pipeline bottleneck finder)
- __slots__: 2–3× memory reduction + type safety for frequently instantiated objects
- tracemalloc: snapshot comparison for memory profiling of infrastructure
- Rule: measurement infrastructure must itself be lean — don't pay 5MB to measure 1MB
- Next: Stage 5 — caching expensive computations, TTL expiry, bounded memory budgets

---
