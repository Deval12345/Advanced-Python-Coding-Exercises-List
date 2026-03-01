# Speech Source — Lecture 34: Big Project Stage 2 — Resource Lifecycle Architecture

---

## CONCEPT 1.1 — Adding Context Managers to the Source Protocol

**PROBLEM:**
When a pipeline fails partway through execution — a network error, a bad record, an unhandled exception in a downstream stage — any open file handles, database connections, or socket objects held by the Source remain open. In production systems running 24/7, this is not a theoretical concern. It is a guaranteed eventual leak. You accumulate ghost connections until the operating system refuses to open new ones, or until a database connection pool is exhausted and your entire service goes down. The pipeline had no structured way to say: "when this is done, no matter how it ends, clean this up."

**WHY IT WAS INVENTED:**
Python's context manager protocol — the `__enter__` and `__exit__` dunder methods — was designed precisely for this problem. It guarantees that cleanup code runs whether execution ends normally or through an exception. The `with` statement is a contract: whatever happens inside the block, `__exit__` will be called. By wrapping the entire pipeline execution inside a `PipelineContext` manager, you give the system a single chokepoint where resources are opened, tracked, and definitively closed. This is the same philosophy behind RAII in C++ and try-with-resources in Java — the resource owns its own cleanup.

**WHAT HAPPENS WITHOUT IT:**
Without a context manager, cleanup depends on developers remembering to write `finally` blocks everywhere, and on those `finally` blocks correctly handling the case where the resource was never opened in the first place. In practice, exception paths are the least-tested code paths in any system. A developer tests the happy path, ships it, and three weeks later a network blip causes an exception at record 10,000 of 50,000 — and the database connection is never returned to the pool. The monitoring dashboard shows connections slowly creeping upward. By the time anyone notices, the system has been running degraded for hours.

**INDUSTRY IMPACT:**
Context managers are ubiquitous in production Python infrastructure. Django's database transaction management wraps every request in a context manager. SQLAlchemy's session handling uses them. AWS SDK clients, gRPC channels, file uploads — every serious I/O-intensive library in the Python ecosystem exposes a context manager interface, precisely because it is the only reliable way to guarantee cleanup in an exception-prone environment. In a streaming pipeline running continuously, this is not optional hygiene — it is architectural load-bearing structure.

---

**EXAMPLE 1.1 — PipelineContext Manager Wrapping a Source**

```python
# Example 34.1
import contextlib
import time

class PipelineContext:
    def __init__(self, source, pipelineId="default"):
        self.source = source
        self.pipelineId = pipelineId
        self._startTime = None
        self._recordCount = 0

    def __enter__(self):
        self._startTime = time.perf_counter()
        print(f"[Pipeline {self.pipelineId}] Starting")
        return self

    def __exit__(self, excType, excVal, excTb):
        elapsed = time.perf_counter() - self._startTime
        if excType is not None:
            print(f"[Pipeline {self.pipelineId}] FAILED after {elapsed:.3f}s: {excVal}")
        else:
            print(f"[Pipeline {self.pipelineId}] Completed: {self._recordCount} records in {elapsed:.3f}s")
        return False  # do not suppress exceptions

    def run(self, stages, sink):
        stream = self.source.stream()
        for stage in stages:
            stream = stage.process(stream)
        for record in stream:
            self._recordCount += 1
            sink.receive(record)
```

**NARRATION:**
The `__init__` method stores the source and a pipeline identifier, and initializes two tracking fields. Notice they start as `None` and `0` — not set yet, because the pipeline has not started yet. `__enter__` is called when the `with` block is entered. It records the wall-clock start time using `perf_counter` — a high-resolution timer — and prints a startup message, then returns `self` so the caller can use `ctx.run(...)`. `__exit__` receives three arguments: the exception type, value, and traceback, all of which are `None` if execution completed normally. We compute elapsed time and branch on whether `excType` is `None` to print either a success or failure message. The critical line is `return False` — this tells Python not to suppress the exception. If the pipeline failed, we want the caller to know. The `run` method wires up the lazy generator chain — `source.stream()` produces a generator, each `stage.process(stream)` wraps it in another generator, and the final `for` loop is the only place records actually move through the chain. Every iteration increments `_recordCount` before handing the record to the sink.

---

## CONCEPT 2.1 — Descriptors for Validated Pipeline Configuration

**PROBLEM:**
A pipeline has configuration parameters: sampling rate, filter bounds, window sizes. Without validation, you can construct a `PipelineConfig(samplingRateHz=-5, windowSize=0, filterMin=90.0, filterMax=10.0)` — negative sampling rate, zero window, inverted filter bounds — and no error is raised at construction time. The pipeline runs, produces garbage output, and the error surfaces two stages downstream as a `ZeroDivisionError` inside a statistics computation, with a stack trace that points nowhere near the configuration that caused it. You spend an hour debugging the wrong place.

**WHY IT WAS INVENTED:**
Python descriptors — objects that implement `__get__`, `__set__`, and optionally `__delete__` and `__set_name__` — allow you to attach custom logic to attribute access on a class. When you assign `obj.samplingRateHz = value`, Python checks whether the class has a descriptor at that name and calls its `__set__` method. This means you can enforce type checking, range validation, and invariants at the exact moment of assignment — not when the value is eventually used. Descriptors are how Python implements `property` internally, but they are reusable across multiple attributes and multiple classes, which `property` is not.

**WHAT HAPPENS WITHOUT IT:**
Without validation at assignment time, invalid configuration is a time bomb. It detonates at an arbitrary point during pipeline execution, often inside library code that has nothing to do with configuration, producing an error message that has no obvious connection to the root cause. Validation at the configuration boundary transforms a mysterious runtime crash into an immediate, clear error message pointing at exactly what was wrong and why. This is the principle of fail-fast: surface errors as early and as specifically as possible.

**INDUSTRY IMPACT:**
Descriptor-based validation appears in Django's model field system — every `CharField`, `IntegerField`, and `FloatField` is implemented using descriptors. Pydantic, which has become the de facto Python validation library (used in FastAPI, LangChain, and most modern Python services), uses `__get__` and `__set__` internally. SQLAlchemy column definitions use them. The pattern is so valuable that Python 3.6 added `__set_name__` specifically to make descriptors easier to write — the language itself evolved to make this pattern more ergonomic.

---

**EXAMPLE 2.1 — Descriptor-Based PipelineConfig**

```python
# Example 34.2
class PositiveFloat:
    def __set_name__(self, owner, name):
        self._name = f"_{name}"
    
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return getattr(obj, self._name, None)
    
    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f"{self._name[1:]} must be numeric, got {type(value).__name__}")
        if value <= 0:
            raise ValueError(f"{self._name[1:]} must be positive, got {value}")
        setattr(obj, self._name, float(value))

class BoundedInt:
    def __init__(self, minVal, maxVal):
        self.minVal = minVal
        self.maxVal = maxVal
    
    def __set_name__(self, owner, name):
        self._name = f"_{name}"
    
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return getattr(obj, self._name, None)
    
    def __set__(self, obj, value):
        if not isinstance(value, int):
            raise TypeError(f"{self._name[1:]} must be int")
        if not (self.minVal <= value <= self.maxVal):
            raise ValueError(f"{self._name[1:]} must be in [{self.minVal}, {self.maxVal}]")
        setattr(obj, self._name, value)

class PipelineConfig:
    samplingRateHz = PositiveFloat()
    windowSize = BoundedInt(1, 10000)
    filterMin = PositiveFloat()
    filterMax = PositiveFloat()
    
    def __init__(self, samplingRateHz, windowSize, filterMin, filterMax):
        self.samplingRateHz = samplingRateHz
        self.windowSize = windowSize
        self.filterMin = filterMin
        self.filterMax = filterMax
    
    def __repr__(self):
        return (f"PipelineConfig(rate={self.samplingRateHz}Hz, window={self.windowSize}, "
                f"filter=[{self.filterMin}, {self.filterMax}])")

if __name__ == "__main__":
    config = PipelineConfig(samplingRateHz=100.0, windowSize=10, filterMin=20.0, filterMax=80.0)
    print(f"Valid config: {config}")
    try:
        config.windowSize = 0
    except ValueError as e:
        print(f"Validation caught: {e}")
    try:
        config.samplingRateHz = "fast"
    except TypeError as e:
        print(f"Type check caught: {e}")
```

**NARRATION:**
`PositiveFloat` is a descriptor class. `__set_name__` is called automatically by Python when the descriptor is assigned as a class attribute — it receives the owner class and the attribute name, and stores a mangled version `_name` with an underscore prefix, so the actual value is stored on the instance under a private name that does not collide with the descriptor itself. `__get__` handles the case where `obj is None` — this happens when you access the descriptor on the class rather than an instance, and returning `self` (the descriptor) is the conventional response. `__set__` does the real work: type check first, range check second, then `setattr` to store the validated value under the private name. `BoundedInt` follows the same pattern but accepts constructor arguments for the allowed range — this is why it needs `__init__`, while `PositiveFloat` does not. In `PipelineConfig`, each class attribute is a descriptor instance. When `__init__` assigns `self.samplingRateHz = samplingRateHz`, Python intercepts that assignment and routes it through `PositiveFloat.__set__`. Invalid values never reach storage.

---

## CONCEPT 3.1 — Decorators for Pipeline Metrics

**PROBLEM:**
A pipeline runs for 30 seconds and processes 100,000 records. Is that fast? You have no idea, because you have no baseline. More importantly: which stage is the bottleneck? Is it the sensor reader doing I/O? The normalization stage doing arithmetic? The statistics window doing repeated summation? Without per-stage timing data, performance optimization is pure guesswork — you might spend a week optimizing a stage that contributes 3% of total latency while the real bottleneck sits untouched. Additionally, when a source fails intermittently — a network hiccup, a sensor going offline — without retry logic, that failure propagates immediately to the entire pipeline.

**WHY IT WAS INVENTED:**
Decorators allow you to wrap a function or method in additional behavior without modifying its source code. Applied to generators, a decorator can wrap the generator function and intercept every yielded value, measuring the time between yields. This makes metrics collection transparent to the stage being measured — the `ThresholdFilter` does not need to know it is being timed. Retry decorators similarly wrap the function call in a loop with error handling, making retry behavior reusable across any source without each source implementing its own retry loop.

**WHAT HAPPENS WITHOUT IT:**
Without metrics, performance problems are discovered in production by users experiencing slowdowns, not by engineers proactively monitoring. Without retry, a transient network error that would have succeeded on the second attempt instead crashes the pipeline and requires manual restart. Both are unacceptable in a production analytics system that is expected to run continuously and deliver real-time results.

**INDUSTRY IMPACT:**
Metrics decorators are the backbone of observability frameworks. Prometheus client libraries for Python use decorator syntax to instrument functions. Datadog's APM tracing wraps functions using decorators. OpenTelemetry's Python SDK uses them. Celery, the distributed task queue, instruments task execution through decorators. The pattern is so pervasive that it has become the standard idiom for adding cross-cutting concerns — logging, metrics, tracing, retry — without polluting business logic with infrastructure code.
