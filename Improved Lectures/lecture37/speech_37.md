# Lecture 37: Big Project Stage 4 — Measurable Pipeline (Profiling and Metrics)
## speech_37.md — Instructor Spoken Narrative

---

We have a pipeline. It ingests sensor data through an async event loop, processes batches through a process pool, composes stages through generator chains, and manages resources through context managers. Stage 3 is complete. The system works.

Now comes a question that separates good engineers from great ones. Does your system actually work correctly under load? And when it slows down in production — and it will slow down in production — how will you know where to look?

Today is Stage 4: making the pipeline measurable.

---

Let me tell you about a real pattern I have seen many times. A team has a working pipeline. It processes 50,000 records and takes 30 seconds. Someone says "we need to make it faster." A developer looks at the code and says "the feature computation stage looks complex — let me optimize that." Two days of careful work. The feature stage is now 40% faster.

The pipeline still takes 29.1 seconds.

The feature stage was 3% of total runtime. Making it infinitely fast — replacing it with nothing — would save 0.9 seconds. The actual bottleneck was the rolling window computation in the feature stage — specifically, it was calling `sum(window)` inside the loop, recomputing the entire 10-element window sum from scratch for every single record. That was 85% of feature stage runtime. But the developer optimized the wrong part because they were optimizing by inspection, not by measurement.

This is the single most expensive mistake in performance engineering. You optimize what looks slow instead of what is actually slow. The fix is dead simple: run a profiler before writing a single line of optimization code.

---

`cProfile` is Python's built-in deterministic profiler. You wrap your pipeline run between `profiler.enable()` and `profiler.disable()`, then pass the profiler to `pstats.Stats` and sort by cumulative time. Cumulative time — the time spent in a function and everything it calls — tells you which functions represent the most expensive call chains. You look at the top five functions. The bottleneck is almost always there, because the profiler has done the accounting that your intuition cannot.

In Example 37.1, we profile the full pipeline: generate records, filter, normalize, compute features. With 50,000 records, the output is unambiguous: `computeFeatures` dominates, `sum` inside `computeFeatures` appears thousands of times, and the fix is obvious — use a running sum that updates incrementally rather than recomputing from scratch.

This is the measure-analyze-optimize-re-measure cycle. You do not skip steps. You do not optimize without measuring. You do not call an optimization done without re-measuring to confirm the improvement.

`cProfile` has overhead — typically 2 to 5 times slowdown during profiling. That is acceptable for offline diagnosis. You run it once, in development, against a realistic workload. You do not run it in production. The measurement cost is the price of diagnostic precision.

---

Now — cProfile is a development-time tool. It tells you what was slow during one run. But in production, you need continuous visibility. You need to know: right now, at this moment, how many records is the filter stage passing per second? How does that compare to five minutes ago? Is the normalization stage's per-record latency trending upward — suggesting memory pressure or GIL contention?

This is where the `@measuredStage` decorator comes in.

The pattern from Example 37.2 wraps generator methods. The decorator creates a `StageMetrics` object for each run, measures time at each yield point, accumulates per-record latency, and reports throughput and mean latency when the generator exhausts. The decorator uses `@functools.wraps` so the wrapped method looks identical to the original in stack traces and documentation — the measurement is structurally invisible to callers.

There is a subtlety worth understanding. The timing point. We record `recStart` just before `yield record`. After the `yield`, execution suspends — the downstream stage or consumer takes over. Execution resumes at the `yield` only when the downstream consumer asks for the next record. So the time between the `yield` and the resume is not the stage's processing time — it is the downstream consumer's processing time. This is exactly what we want: the filter's "latency" as measured by this decorator reflects how long the downstream normalizer takes to consume each record. If the filter is fast but the normalizer is slow, the filter's recorded latency will be high — because it is waiting for the normalizer. The profiler finds the intrinsic bottleneck; the decorator finds the pipeline bottleneck.

---

The third concept in this lecture is `__slots__` for metrics objects.

Our metrics system creates one `StageMetrics` object per pipeline run, per stage. For a long-running pipeline making many passes, that is many accumulators. Each accumulator, without `__slots__`, carries a `__dict__` — a hash table that maps attribute names to values. For five attributes, the `__dict__` alone uses 200 to 300 bytes. A plain Python object instance adds another 50 bytes. Total: 250 to 350 bytes per accumulator.

With `__slots__`, we declare exactly which attributes the class can have. Python allocates C-level slot descriptors at the class level — fixed-size arrays rather than hash tables. The per-instance memory drops to 50 to 80 bytes. For a metrics system that creates thousands of accumulators, the memory difference is real.

There is a second benefit that is easy to miss: `__slots__` catches attribute typos at the point of error. Without slots, `self.totlaTime = x` — misspelled — silently creates a new attribute. With slots, it raises `AttributeError` immediately. In a metrics system where attribute names matter precisely, this is valuable protection.

Example 37.3 demonstrates the comparison. We create 10,000 `MetricsCollectorDict` instances and 10,000 `MetricsCollectorSlots` instances and measure actual memory consumption using `tracemalloc`. The `tracemalloc` module is Python's built-in memory profiler: `tracemalloc.take_snapshot()` records the current allocation state; comparing two snapshots reveals exactly how much memory was allocated between them and by which lines of code.

The output typically shows the slots version using 2 to 3 times less memory for the accumulator objects, with the difference more pronounced when the pipeline creates many collectors.

---

Let me now put Stage 4 in context. What does "measurable" add to the pipeline?

Before Stage 4: the pipeline works but is opaque. You can observe the final output — records processed, time elapsed. You cannot observe the internal state: which stage is the bottleneck? How does throughput change under load? Is memory growing?

After Stage 4: the pipeline is observable at two timescales. Development-time: run cProfile, read the hotspot report, fix the bottleneck. Production-time: the `@measuredStage` decorator collects per-stage metrics automatically, every run, with negligible overhead. Memory: `tracemalloc` snapshots and `__slots__` on metrics objects keep the measurement infrastructure itself lean.

The discipline — measure first, optimize second, measure again — is not bureaucracy. It is the difference between spending two days optimizing the wrong thing and spending two hours fixing the right thing. The profiler is the tool that makes that distinction possible.

In the next stage, we add caching. Some computations in the pipeline are expensive and repeated: configuration validation, calibration lookups, schema checks. Caching these results amortizes the computation cost across many calls. We will use `functools.lru_cache` for exact match caching and build a custom TTL cache for cases where cached results should expire after a time period. We will measure the memory cost of the cache itself — using what we learned today — and set appropriate size limits.

