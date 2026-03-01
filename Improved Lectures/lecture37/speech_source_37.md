# Speech Source — Lecture 37: Big Project Stage 4 — Measurable Pipeline (Profiling and Metrics)

---

## CONCEPT 1.1 — Why Measure? cProfile on the Pipeline

**PROBLEM:**
You have a pipeline processing 50,000 records and it takes 30 seconds. Someone says "it is slow, make it faster." Where do you start? Without measurement, the answer is pure guesswork. You might rewrite the normalization stage — spending two days on a careful optimization — only to discover that normalization was 3% of total runtime. The actual bottleneck was the rolling window computation, which you never touched. You have wasted two days and the pipeline is 2.9% faster. This is the optimization without measurement trap, and it is one of the most common and expensive mistakes in software engineering.

**WHY IT WAS INVENTED:**
`cProfile` is Python's built-in deterministic profiler. It instruments every function call — recording how many times each function was called, total time spent in it, and cumulative time (including all functions called from it). It produces a statistical map of where your program spends its time. The cost is non-trivial — typically 2-10x slowdown during profiling — but you run it once, offline, to diagnose. `pstats` provides the reporting layer: sort by cumulative time to find the slowest call chains, sort by total calls to find tight loops, filter by module to focus on your code rather than standard library internals.

**WHAT HAPPENS WITHOUT IT:**
Without profiling, optimization is cargo-cult engineering. You optimize the parts that look complex, not the parts that are actually slow. Modern software is full of counterintuitive performance characteristics: a simple list comprehension inside a tight loop might be slower than a more complex but vectorized NumPy operation; string formatting might dominate a pipeline that does sophisticated statistics; a function called 10,000 times for 0.001ms each accumulates 10ms of overhead that only shows up in a profiler. Without a profiler, these are invisible.

**INDUSTRY IMPACT:**
cProfile is standard practice before any performance optimization in Python data science and engineering. Companies with Python-heavy infrastructure — Instagram, Dropbox, YouTube — have all published post-mortems that led to using cProfile (or sampling profilers like `py-spy`) to identify unexpected bottlenecks. PySpark performance tuning routinely involves profiling the Python UDF (user-defined function) side separately from the JVM side. Any mature Python engineering team has profiling integrated into their performance investigation workflow.

---

**EXAMPLE 1.1 — Pipeline Profiling with cProfile**

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

**NARRATION:**
`profiler = cProfile.Profile()` — Creates a profiler instance. The profiler hooks into Python's function call machinery using C-level hooks, so it captures every function call precisely.

`profiler.enable()` — Starts profiling. Every subsequent function call is recorded. The overhead is approximately 2-5x slowdown.

`profiler.disable()` — Stops profiling. Only the code between `enable()` and `disable()` is measured. Keep the profiled section as tight as possible — only the pipeline run, not setup code.

`stream = io.StringIO()` — Creates an in-memory text buffer. We redirect `pstats` output here instead of stdout, so we can print it cleanly.

`stats = pstats.Stats(profiler, stream=stream)` — Wraps the profiler data with the reporting interface.

`stats.sort_stats("cumulative")` — Sorts by cumulative time — the total time spent in a function and all functions it calls. This shows which functions represent the most expensive call chains.

`stats.print_stats(8)` — Print the top 8 functions by cumulative time. The output shows: `ncalls` (number of calls), `tottime` (time in this function, excluding callees), `cumtime` (time including callees), `filename:lineno(function)`.

`computeFeatures` — Will typically dominate the profile, because `sum(window)` and `sum((x-mean)**2 for x in window)` recompute the entire window for every record. A rolling mean should be updated incrementally, not recomputed from scratch. The profiler makes this obvious.

---

## CONCEPT 2.1 — Stage-Level Metrics with @timed_stage

**PROBLEM:**
`cProfile` tells you which functions are slow — it is a development-time tool. In production, you need continuous, low-overhead measurement of pipeline performance: how many records did the filter pass vs. reject in the last 5 minutes? What was the p95 latency of the normalization stage over the last hour? Is throughput dropping, suggesting a resource contention problem upstream? You need per-stage metrics that collect continuously without requiring profiler overhead, that can be queried by monitoring systems, and that do not require modifying the stage's business logic.

**WHY IT WAS INVENTED:**
The `@measuredStage` decorator pattern (refined here from Lecture 34) makes metrics collection structural. The decorator wraps the generator method, measuring time at each yield. The `StageMetrics` dataclass accumulates counts, latencies, and derived statistics. By using `@functools.wraps`, the decorated method appears identical to the original in stack traces and documentation — metrics are truly invisible to callers. The pattern is applied at class definition time, not at runtime, so there is no per-record branching cost for deciding whether to measure.

**WHAT HAPPENS WITHOUT IT:**
Without per-stage metrics, operators know the pipeline is slow but not where. They might add timing code ad hoc — `startTime = time.time(); ...; elapsed = time.time() - startTime` scattered throughout the codebase — which is fragile (easy to forget in one place), noisy (every stage author writes slightly different timing code), and not easily machine-readable. Structured, decorator-based metrics produce consistent, queryable data from every stage.

**INDUSTRY IMPACT:**
Every major Python observability library uses decorator-based instrumentation. Prometheus's Python client provides `@histogram.time()` and `@summary.time()` decorators that measure function call duration and expose the data via an HTTP endpoint for scraping. Datadog's `ddtrace` library wraps functions with `@tracer.wrap()`. OpenTelemetry's Python SDK uses `@tracer.start_as_current_span()`. The `@decorator-wraps-generator-and-measures-each-yield` pattern is the idiomatic Python approach to production observability.

---

**EXAMPLE 2.1 — Stage-Level Metrics with a Decorator**

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

**NARRATION:**
`recStart = time.perf_counter()` — Recorded just before `yield record`. This captures the time when the record leaves this stage and enters the downstream consumer.

`yield record` — Suspends this generator and passes control to the downstream stage or the final consumer loop. Execution resumes here when the consumer requests the next record.

`metrics.latenciesMs.append((time.perf_counter() - recStart) * 1000)` — After `yield` resumes, we measure how long the downstream consumer spent before asking for the next record. This captures the consumer's processing time, not the stage's — which is exactly what we want for bottleneck identification. If the filter is fast but the downstream normalizer is slow, the filter's latency list will show high values because it spends most of its time waiting for the normalizer to be ready for the next record.

---

## CONCEPT 3.1 — MetricsCollector with __slots__

**PROBLEM:**
The pipeline processes millions of records. If every record creates a full Python object for tracking — with `__dict__`, with arbitrary attribute expansion, with the full overhead of Python's object model — the memory pressure from the metrics tracking infrastructure itself can become significant. A metrics accumulator that creates new Python dicts for every record, or appends to unbounded lists for every latency measurement, eventually shows up in memory profiles and GC pressure.

**WHY IT WAS INVENTED:**
`__slots__` is a class-level declaration that replaces the per-instance `__dict__` with a fixed-size C-level array of slot descriptors. An instance with `__slots__ = ("x", "y", "z")` has no `__dict__` — it cannot have arbitrary attributes added at runtime — but it uses significantly less memory per instance and has faster attribute access. For a `MetricsCollector` that aggregates stats from all stages and may be instantiated thousands of times (one per pipeline run, or one per stage per batch), the memory savings compound. `__slots__` is the Python mechanism for memory-efficient value objects.

**WHAT HAPPENS WITHOUT IT:**
Without `__slots__`, every Python object instance carries a `__dict__` — a hash table that maps attribute names to values. For an object with 5 attributes, `__dict__` might use 200-300 bytes. A `__slots__` object with the same 5 attributes uses approximately 50-100 bytes. For a metrics system that creates many accumulators, this difference is meaningful. Additionally, `__dict__` objects can have attributes added at runtime — which is a source of subtle bugs when a typo in an attribute name silently creates a new attribute instead of raising `AttributeError`.

**INDUSTRY IMPACT:**
`__slots__` is used throughout Python's high-performance infrastructure. NumPy's Python-level objects use slots. SymPy uses them for expression tree nodes (which are created billions of times in symbolic computations). CPython's own internal objects use the C equivalent. Protocol Buffer generated Python classes (from `protoc`) use `__slots__`. SQLAlchemy uses them for ORM column descriptors. Any Python library that creates millions of small objects — parsers, AST nodes, pipeline records, event objects — uses `__slots__` to control memory usage.
