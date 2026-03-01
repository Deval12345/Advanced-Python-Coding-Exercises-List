# Code — Lecture 34: Big Project Stage 2 — Resource Lifecycle Architecture

---

## Example 34.1 — PipelineContext Manager

```python
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
        return False

    def run(self, stages, sink):
        stream = self.source.stream()
        for stage in stages:
            stream = stage.process(stream)
        for record in stream:
            self._recordCount += 1
            sink.receive(record)
```

**Line-by-line explanation:**

`class PipelineContext:` — A class that implements the context manager protocol. It is not a generator, not a decorator — it is a resource manager that wraps an entire pipeline execution.

`def __init__(self, source, pipelineId="default"):` — Stores the source and an identifier for log messages. Two tracking fields: `_startTime` is `None` because the clock has not started yet, `_recordCount` is `0` because no records have moved yet. State is initialized lazily in `__enter__`, not in `__init__`.

`def __enter__(self):` — Called by Python when execution enters the `with PipelineContext(...) as ctx:` block. This is the start-of-life hook.

`self._startTime = time.perf_counter()` — `perf_counter()` returns a high-resolution float in seconds, suitable for short-duration measurements. We use it rather than `time.time()` because `perf_counter` is guaranteed monotonic — it never goes backward, even during NTP adjustments.

`return self` — The `as ctx` variable in `with PipelineContext(...) as ctx:` receives whatever `__enter__` returns. Returning `self` allows `ctx.run(...)` to work naturally.

`def __exit__(self, excType, excVal, excTb):` — Called by Python when the `with` block exits, for any reason. `excType` is the exception class if one occurred (e.g., `ConnectionError`), `excVal` is the exception instance with its message, `excTb` is the traceback object. All three are `None` if the block completed normally.

`elapsed = time.perf_counter() - self._startTime` — Computes total wall-clock duration. This runs before the conditional so it appears in both the success and failure log message.

`if excType is not None:` — Check whether we are in an error path. Using `is not None` rather than just `if excType` because exception classes are always truthy, but this is the conventional and clear form.

`return False` — This is the most important line. `False` means "do not suppress the exception." If an exception occurred, Python will re-raise it after `__exit__` returns. `True` would suppress it — almost never what you want in a production system, because it silently discards the error.

`def run(self, stages, sink):` — The method that actually executes the pipeline. Called inside the `with` block.

`stream = self.source.stream()` — Calls the source's generator method. This returns a generator object but does not start execution — no records move yet. Python generators are lazy.

`for stage in stages:` — Iterates through the list of pipeline stages in order.

`stream = stage.process(stream)` — Each stage wraps the current stream in a new generator. After this loop, `stream` is a chain of nested generators. Still no records have moved.

`for record in stream:` — This is the pull point. Iterating this loop forces execution through the entire generator chain — records are pulled from the source, passed through each stage, and arrive here one at a time.

`self._recordCount += 1` — Increment before sending to the sink, so the count reflects records that entered the sink, not records that the sink successfully processed. (If the sink raises, we still know how many records reached it.)

`sink.receive(record)` — The sink receives each record. This could be writing to a file, inserting into a database, sending over a network — the context manager does not care.

---

## Example 34.2 — Descriptor-Based PipelineConfig

```python
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

**Line-by-line explanation:**

`class PositiveFloat:` — A non-data descriptor class. It implements `__get__` and `__set__`, which makes it a data descriptor — Python will route attribute access through it even if the instance `__dict__` has an entry at the same name.

`def __set_name__(self, owner, name):` — Called by Python's metaclass during class body processing, when the descriptor instance is assigned as a class attribute. `owner` is the class being defined (e.g., `PipelineConfig`), `name` is the attribute name (e.g., `"samplingRateHz"`). This runs once at class definition time, not at instance creation.

`self._name = f"_{name}"` — Stores `"_samplingRateHz"` as the private storage name. The underscore prefix ensures this does not collide with the descriptor's own name on the class, which would cause infinite recursion.

`def __get__(self, obj, objtype=None):` — Called when the attribute is read. `obj` is the instance (or `None` if accessed on the class). `objtype` is the class.

`if obj is None: return self` — Class-level access: `PipelineConfig.samplingRateHz` returns the descriptor itself. This is the conventional behavior and allows `isinstance(PipelineConfig.samplingRateHz, PositiveFloat)` to work.

`return getattr(obj, self._name, None)` — Read the private attribute from the instance. The default `None` handles the case where the attribute has not been set yet.

`def __set__(self, obj, value):` — Called when `obj.samplingRateHz = value` executes. `obj` is the instance receiving the assignment.

`if not isinstance(value, (int, float)):` — Type guard. Accepts both `int` and `float` because Python's numeric tower allows `int` where `float` is semantically appropriate, and users might pass `100` instead of `100.0`.

`raise TypeError(f"{self._name[1:]} must be numeric, got {type(value).__name__}")` — `self._name[1:]` strips the leading underscore to show the public attribute name in the error message.

`setattr(obj, self._name, float(value))` — Stores the validated value as a `float` under the private name. Calling `float()` normalizes ints to floats for consistency.

`class BoundedInt:` — A descriptor for integer values within a specific range. Unlike `PositiveFloat`, it requires constructor arguments for the bounds, so it has `__init__`.

`def __init__(self, minVal, maxVal):` — Stores the bounds as instance attributes of the descriptor. Each `BoundedInt()` call creates a separate descriptor instance with its own bounds — this is how `BoundedInt(1, 10000)` and `BoundedInt(1, 100)` can coexist as different class attributes.

`class PipelineConfig:` — The configuration class. Its class body assigns descriptor instances as class attributes.

`samplingRateHz = PositiveFloat()` — Creates a `PositiveFloat` descriptor instance and assigns it to the class. When Python processes this line, it calls `PositiveFloat().__set_name__(PipelineConfig, "samplingRateHz")`.

`def __init__(self, samplingRateHz, windowSize, filterMin, filterMax):` — Each assignment in `__init__` routes through the corresponding descriptor's `__set__`. The descriptors validate all four values before any are stored.

`def __repr__(self):` — Human-readable string representation. Uses f-string with the public attribute names, which route through `__get__` to return the stored values.

---

## Example 34.3 — Measured Stage Decorator (from Concept 3.1)

```python
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

class MeasuredThresholdFilter:
    def __init__(self, minVal, maxVal):
        self.minVal = minVal
        self.maxVal = maxVal
    
    @measuredStage("ThresholdFilter")
    def process(self, inputStream):
        for record in inputStream:
            if self.minVal <= record["value"] <= self.maxVal:
                yield record
```

**Line-by-line explanation:**

`@dataclass class StageMetrics:` — A dataclass to hold all metrics for one stage execution. Using `@dataclass` generates `__init__`, `__repr__`, and `__eq__` automatically from the field definitions.

`latenciesMs: List[float] = field(default_factory=list)` — `field(default_factory=list)` is required for mutable defaults in dataclasses. Without it, all instances would share the same list object, which is a classic Python gotcha.

`@property def throughputRps(self):` — Computed property: records per second. Divides `recordsOut` by total time in seconds. Guards against zero total time.

`def measuredStage(stageName):` — Outer function of a two-level decorator factory. Takes the stage name as a configuration argument.

`def decorator(processMethod):` — Middle level. Receives the method being decorated (`process`).

`@functools.wraps(processMethod)` — Copies `__name__`, `__doc__`, `__module__`, `__qualname__`, and `__annotations__` from the original method to the wrapper. Without this, `wrapper.__name__` would be `"wrapper"` instead of `"process"`, which breaks debugging and introspection.

`def wrapper(self, inputStream):` — The replacement method. Has the same signature as `process` — `self` because it wraps a method, `inputStream` because that is what `process` receives.

`metrics = StageMetrics(stageName=stageName)` — Fresh metrics object for each invocation. This is important: if the stage is called multiple times, each call gets its own metrics.

`for record in processMethod(self, inputStream):` — Calls the original `process` method and iterates through the records it yields. `processMethod(self, inputStream)` passes the instance explicitly because `processMethod` is an unbound function at this point.

`recStart = time.perf_counter()` — Record the time just before yielding each record to the downstream consumer.

`yield record` — This is what makes `wrapper` a generator. The record passes through to whoever is iterating the measured stage.

`metrics.latenciesMs.append((time.perf_counter() - recStart) * 1000)` — After `yield` returns (when the consumer requests the next item), we measure the time elapsed since the yield and store it. This measures how long the consumer spent processing the record before asking for another.

`metrics.totalTimeMs = (time.perf_counter() - startTotal) * 1000` — Set after the inner generator is exhausted. This runs after the `for` loop completes.

`@measuredStage("ThresholdFilter")` — Applies the two-level decorator. Python evaluates `measuredStage("ThresholdFilter")` first, which returns `decorator`. Then `decorator(process)` is called, which returns `wrapper`. `MeasuredThresholdFilter.process` is now `wrapper`.
