# Code — Lecture 37: Big Project Stage 4 — Measurable Pipeline (Profiling and Metrics)

---

## Example 37.1 — Pipeline Profiling with cProfile

```python
# Example 37.1
import cProfile
import pstats
import io
import math
import random
import time
from collections import deque

def generateTestRecords(n):
    return [{"sensorId": f"S{i%5}", "timestamp": time.time(), "value": random.gauss(50, 15), "unit": "C"}
            for i in range(n)]

def thresholdFilter(records, minVal, maxVal):
    return [r for r in records if minVal <= r["value"] <= maxVal]

def normalizeRecords(records, minVal, maxVal):
    r = maxVal - minVal
    return [{**rec, "value": (rec["value"] - minVal) / r} for rec in records]

def computeFeatures(records, windowSize):
    window = deque(maxlen=windowSize)
    results = []
    for rec in records:
        window.append(rec["value"])
        mean = sum(window) / len(window)
        variance = sum((x - mean)**2 for x in window) / len(window)
        results.append({**rec, "mean": round(mean, 6), "variance": round(variance, 6)})
    return results

def fullPipeline(n):
    records = generateTestRecords(n)
    filtered = thresholdFilter(records, 20.0, 80.0)
    normalized = normalizeRecords(filtered, 20.0, 80.0)
    featured = computeFeatures(normalized, windowSize=10)
    return featured

if __name__ == "__main__":
    profiler = cProfile.Profile()
    profiler.enable()
    result = fullPipeline(50000)
    profiler.disable()

    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats("cumulative")
    stats.print_stats(8)
    print(stream.getvalue())
    print(f"Processed {len(result)} records")
```

### Line-by-Line Explanation

**`def generateTestRecords(n):`**
Creates N sensor records as dictionaries with sensorId, timestamp, value, and unit. `random.gauss(50, 15)` generates normally-distributed sensor readings — approximately 83% will fall in the [20, 80] filter range. `time.time()` records the wall-clock time at generation. Each record is a plain dict — no __slots__ here, because records are short-lived and their schema varies across pipeline stages.

**`def thresholdFilter(records, minVal, maxVal):`**
First pipeline stage. Filters records outside the valid sensor range. The list comprehension processes all records in a single pass and produces a new list. At N=50,000 with a normal distribution, roughly 41,500 records pass (83%). This stage is fast — one comparison per record, no allocations inside the check.

**`def normalizeRecords(records, minVal, maxVal):`**
Maps values from [minVal, maxVal] to [0, 1]. `{**rec, "value": ...}` creates a new dictionary merging all existing keys with the overwritten "value" key. This is O(N) with a small constant — one division and one dict spread per record.

**`def computeFeatures(records, windowSize):`**
The performance hotspot. For each record, appends to a `deque(maxlen=windowSize)` which automatically drops the oldest element when full. `mean = sum(window) / len(window)` — this is the critical flaw. `sum(window)` iterates over up to 10 elements on every record. For 41,500 records with window size 10, this is 415,000 additions for mean alone. `variance = sum((x - mean)**2 for x in window) / len(window)` adds another 415,000 operations. Total: ~830,000 arithmetic operations that could be reduced to 82,000 with an incremental rolling sum.

**`profiler = cProfile.Profile()`**
Creates a profiler instance that uses C-level hooks to intercept every Python function call. This is the "deterministic" profiler — it captures every call, unlike "sampling" profilers that sample at intervals. The cost is non-trivial (2–5× slowdown) but the precision is exact.

**`profiler.enable()` / `profiler.disable()`**
Bracket the code section to profile. Code outside this bracket is not measured. Keep the section tight — exclude test data setup and result printing. Only measure the core pipeline logic.

**`stats.sort_stats("cumulative")`**
Sorts the profiler report by cumulative time — total time in each function plus all functions it calls. This identifies expensive call chains. For this pipeline, `fullPipeline` → `computeFeatures` → `sum` will show as the dominant chain. Alternative: `sort_stats("tottime")` to find functions that are intrinsically slow (not just sitting at the top of a slow chain).

**`stats.print_stats(8)`**
Limits output to the 8 most expensive functions. In real profiling sessions, start with 10–15 to get a full picture, then narrow to the top 3–5 for action. The `filename:lineno(function)` column identifies exactly which file and line defines the slow function.

---

## Example 37.2 — Stage-Level Metrics with a Decorator

```python
# Example 37.2
import time
import functools
from dataclasses import dataclass, field
from typing import List

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

def measuredStage(stageName):
    def decorator(processMethod):
        @functools.wraps(processMethod)
        def wrapper(self, inputStream):
            metrics = StageMetrics(stageName=stageName)
            startTotal = time.perf_counter()
            for record in processMethod(self, inputStream):
                metrics.recordsIn += 1
                recStart = time.perf_counter()
                yield record
                metrics.recordsOut += 1
                metrics.latenciesMs.append((time.perf_counter() - recStart) * 1000)
            metrics.totalTimeMs = (time.perf_counter() - startTotal) * 1000
            print(f"[{stageName}] in={metrics.recordsIn} out={metrics.recordsOut} "
                  f"throughput={metrics.throughputRps:.0f}rps "
                  f"meanLatency={metrics.meanLatencyMs:.3f}ms")
        return wrapper
    return decorator
```

### Line-by-Line Explanation

**`@dataclass class StageMetrics:`**
A dataclass provides `__init__`, `__repr__`, and `__eq__` automatically from the field declarations. `recordsIn` and `recordsOut` count separately: the difference is the filter ratio — how many records this stage dropped. `latenciesMs: List[float] = field(default_factory=list)` uses `field(default_factory=list)` rather than a default `[]` — using `[]` as a default would share one list across all instances (a classic Python gotcha with mutable defaults).

**`@property def throughputRps(self):`**
Derived property computed on demand from `recordsOut` and `totalTimeMs`. No separate storage needed — the property computes from stored primitives. `totalTimeMs / 1000` converts to seconds for the rate calculation.

**`def measuredStage(stageName):`**
Outer function receiving the stage name — the decorator factory. Returns a decorator that, when applied to a method, returns a wrapper. Three levels of nesting: factory → decorator → wrapper.

**`@functools.wraps(processMethod)`**
Copies the wrapped method's `__name__`, `__doc__`, `__qualname__`, `__module__`, and `__dict__` to the wrapper. Without this, the wrapped method appears as "wrapper" in stack traces and `help()` output. With it, the instrumented method is indistinguishable from the original to introspection tools.

**`metrics.recordsIn += 1` / `yield record` / `metrics.recordsOut += 1`**
The three-beat rhythm of generator-based measurement. `recordsIn += 1` counts how many records were produced by the wrapped method. `yield record` suspends the wrapper generator, passes the record to the consumer, and waits for the consumer to request the next record. When execution resumes, `recordsOut += 1` counts the record that was successfully delivered and consumed.

**`recStart = time.perf_counter()` before yield; `append(...)` after yield**
The elapsed time between `yield` and resumption is the downstream consumer's processing time — not this stage's processing time. If the downstream stage is slow, this generator spends most of its time suspended after `yield`, waiting. The measured "latency" is therefore the downstream latency, making this decorator ideal for identifying pipeline bottlenecks (where processing time accumulates in the pipeline chain).

---

## Example 37.3 — MetricsCollector with __slots__ and Memory Verification

```python
# Example 37.3
import sys
import time
import tracemalloc
from collections import deque

class MetricsCollectorDict:
    def __init__(self, stageName):
        self.stageName = stageName
        self.count = 0
        self.totalMs = 0.0
        self.recentLatencies = deque(maxlen=100)

    def record(self, latencyMs):
        self.count += 1
        self.totalMs += latencyMs
        self.recentLatencies.append(latencyMs)

    def report(self):
        mean = self.totalMs / self.count if self.count > 0 else 0
        return f"[{self.stageName}] count={self.count} meanMs={mean:.3f}"

class MetricsCollectorSlots:
    __slots__ = ("stageName", "count", "totalMs", "recentLatencies")

    def __init__(self, stageName):
        self.stageName = stageName
        self.count = 0
        self.totalMs = 0.0
        self.recentLatencies = deque(maxlen=100)

    def record(self, latencyMs):
        self.count += 1
        self.totalMs += latencyMs
        self.recentLatencies.append(latencyMs)

    def report(self):
        mean = self.totalMs / self.count if self.count > 0 else 0
        return f"[{self.stageName}] count={self.count} meanMs={mean:.3f}"

if __name__ == "__main__":
    N = 10000
    tracemalloc.start()

    snapshot1 = tracemalloc.take_snapshot()
    dictCollectors = [MetricsCollectorDict(f"stage_{i}") for i in range(N)]
    snapshot2 = tracemalloc.take_snapshot()

    slotsCollectors = [MetricsCollectorSlots(f"stage_{i}") for i in range(N)]
    snapshot3 = tracemalloc.take_snapshot()

    dictSize = sys.getsizeof(dictCollectors[0]) + sys.getsizeof(dictCollectors[0].__dict__)
    slotsSize = sys.getsizeof(slotsCollectors[0])

    stats2 = snapshot2.compare_to(snapshot1, "lineno")
    stats3 = snapshot3.compare_to(snapshot2, "lineno")

    dictMem = sum(s.size_diff for s in stats2) / 1024
    slotsMem = sum(s.size_diff for s in stats3) / 1024

    print(f"Dict collector: ~{dictSize} bytes/instance, {dictMem:.1f} KB for {N} instances")
    print(f"Slots collector: ~{slotsSize} bytes/instance, {slotsMem:.1f} KB for {N} instances")
    print(f"Memory reduction: {dictMem/slotsMem:.1f}x fewer KB with __slots__")
    tracemalloc.stop()
```

### Line-by-Line Explanation

**`__slots__ = ("stageName", "count", "totalMs", "recentLatencies")`**
Class-level attribute declaration. Python replaces the per-instance `__dict__` with a C-level array of slot descriptors — one fixed-size slot per declared name. The tuple must contain the exact strings of all attributes that instances will use. Any `self.undeclaredAttribute = x` raises `AttributeError` — enforcing the attribute contract at the class level.

**`sys.getsizeof(dictCollectors[0]) + sys.getsizeof(dictCollectors[0].__dict__)`**
`sys.getsizeof(obj)` returns the shallow size of an object — the memory for the object's internal structure but not the objects it references. For a dict-based instance, the total size is the base object (the instance header) plus the `__dict__` (the hash table). The `__dict__` starts at 200+ bytes even for a small dict.

**`sys.getsizeof(slotsCollectors[0])`**
For a slots instance, there is no `__dict__`. The single `getsizeof` call captures the entire instance memory. The slot array is part of the base object.

**`tracemalloc.start()`**
Activates Python's memory allocator tracing. All subsequent allocations record their call stack. Must be called before taking the first snapshot.

**`snapshot1 = tracemalloc.take_snapshot()`**
Captures the current state of all live allocations — their size and the source line that allocated them. Takes a complete picture of all Python-managed memory at this moment.

**`snapshot2.compare_to(snapshot1, "lineno")`**
Returns a list of `StatisticDiff` objects sorted by the total size difference between the two snapshots. The `"lineno"` key groups allocations by file and line number. `s.size_diff` is the net bytes allocated at that line between the two snapshots. Summing all `size_diff` values gives the total net allocation between the two snapshots.

**`tracemalloc.stop()`**
Deactivates tracing and frees the tracing data. Call when memory profiling is complete — tracemalloc itself has overhead (storing allocation call stacks).

---
