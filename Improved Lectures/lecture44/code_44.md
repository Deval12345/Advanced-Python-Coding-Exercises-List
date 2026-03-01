# Code — Lecture 44: Course Wrap-Up and What Comes Next

---

## Example 44.1 — The Complete Six-Layer Pipeline Assembly

This example assembles all six architectural layers into a single runnable pipeline. It demonstrates how every pattern from the course composes into a coherent, working system. Each section is labeled with the course phase that introduced it.

```python
# Example 44.1 — Complete six-layer pipeline assembly
# Each section is annotated with the course phase that introduced the pattern

import asyncio
import collections
import json
import random
import time
import tracemalloc
import inspect
from typing import TypeVar, Generic, Iterator, Protocol, TypedDict

# ─── LAYER 1: DATA MODEL AND MEMORY ─────────────────────────────────────────
# Lectures 1–3, 11: Magic methods, __slots__, AutoSlotMeta

class AutoSlotMeta(type):
    def __new__(mcs, name, bases, namespace):
        annotations = namespace.get("__annotations__", {})
        existing = set()
        for base in bases:
            existing.update(getattr(base, "__slots__", ()))
        namespace["__slots__"] = tuple(
            k for k in annotations if k not in existing
        )
        return super().__new__(mcs, name, bases, namespace)

class SensorRecord(metaclass=AutoSlotMeta):
    sensorId: str
    timestamp: float
    value: float
    unit: str

    def __init__(self, sensorId, timestamp, value, unit):
        self.sensorId = sensorId
        self.timestamp = timestamp
        self.value = value
        self.unit = unit

    def __repr__(self):
        return (f"SensorRecord(sensorId={self.sensorId!r}, "
                f"value={self.value:.2f})")

# TypedDict for type-checker contract (zero runtime overhead)
class SensorRecordDict(TypedDict):
    sensorId: str
    timestamp: float
    value: float
    unit: str

# ─── LAYER 2: FUNCTIONS AND CONTROL FLOW ────────────────────────────────────
# Lectures 4–10: Decorators, closures, context managers

import functools, time as _time

def retryWithBackoff(maxAttempts=3, baseDelay=0.05):
    """Decorator factory — closure over maxAttempts and baseDelay."""
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(maxAttempts):
                try:
                    return fn(*args, **kwargs)
                except Exception as exc:
                    if attempt == maxAttempts - 1:
                        raise
                    delay = baseDelay * (2 ** attempt) + random.uniform(0, 0.01)
                    time.sleep(delay)
        return wrapper
    return decorator

class measuredStage:
    """Decorator class — records throughput for each stage invocation."""
    def __init__(self, stageName):
        self.stageName = stageName
        self.totalRecords = 0
        self.totalMs = 0.0

    def __call__(self, fn):
        outer = self
        @functools.wraps(fn)
        def wrapper(self_stage, inputStream):
            t0 = time.perf_counter()
            count = 0
            for record in fn(self_stage, inputStream):
                count += 1
                yield record
            elapsed = (time.perf_counter() - t0) * 1000
            outer.totalRecords += count
            outer.totalMs += elapsed
            rps = (count / elapsed * 1000) if elapsed > 0 else float("inf")
            print(f"[{outer.stageName}] {count} records, {elapsed:.1f}ms, {rps:.0f} rps")
        return wrapper

# ─── LAYER 3: ITERATION AND GENERATORS ──────────────────────────────────────
# Lectures 5–6: Generator composition for lazy, memory-bounded data transport

class SyntheticSource:
    """Generator-based source — pulls records lazily, one at a time."""
    def __init__(self, numRecords, anomalyRate=0.1):
        self.numRecords = numRecords
        self.anomalyRate = anomalyRate

    def stream(self):
        for i in range(self.numRecords):
            value = (random.gauss(50, 10)
                     if random.random() > self.anomalyRate
                     else random.uniform(110, 150))
            yield {
                "sensorId": f"S{i % 10:02d}",
                "timestamp": time.time() + i * 0.1,
                "value": value,
                "unit": "celsius"
            }

_filter_metric = measuredStage("ThresholdFilter")
_norm_metric = measuredStage("WindowNormalizer")

class ThresholdFilter:
    @_filter_metric
    def transform(self, inputStream):
        for r in inputStream:
            if 10.0 <= r["value"] <= 100.0:
                yield r

class WindowNormalizer:
    def __init__(self, windowSize=50):
        self._windowSize = windowSize
        self._window = collections.deque(maxlen=windowSize)

    @_norm_metric
    def transform(self, inputStream):
        for r in inputStream:
            self._window.append(r["value"])
            lo, hi = min(self._window), max(self._window)
            span = hi - lo if hi != lo else 1.0
            normalized = (r["value"] - lo) / span
            yield {**r, "normalized": round(normalized, 6)}

# ─── LAYER 4: CONCURRENCY (simplified synchronous demonstration) ─────────────
# Lectures 12–31: asyncio gather, circuit breaker, retry patterns

class CircuitOpenError(Exception):
    pass

class AsyncCircuitBreaker:
    def __init__(self, failureThreshold=3, resetTimeout=2.0):
        self._state = "closed"
        self._failures = 0
        self._threshold = failureThreshold
        self._resetTimeout = resetTimeout
        self._lastFailure = 0.0

    async def call(self, coroFn, *args, **kwargs):
        if self._state == "open":
            if time.monotonic() - self._lastFailure >= self._resetTimeout:
                self._state = "half_open"
            else:
                raise CircuitOpenError("Circuit open — fast fail")
        try:
            result = await coroFn(*args, **kwargs)
            if self._state == "half_open":
                self._state = "closed"
                self._failures = 0
            return result
        except Exception:
            self._failures += 1
            self._lastFailure = time.monotonic()
            if self._failures >= self._threshold:
                self._state = "open"
            raise

# ─── LAYER 5: DESCRIPTORS AND OBSERVABILITY ──────────────────────────────────
# Lectures 13–14, 40: Descriptor protocol, structured logging

class AuditedAttribute:
    def __set_name__(self, owner, name):
        self._name = name
        self._private = f"_audited_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self._private, None)

    def __set__(self, obj, value):
        old = getattr(obj, self._private, None)
        object.__setattr__(obj, self._private, value)
        if old is not None and old != value:
            print(f"  [AUDIT] {type(obj).__name__}.{self._name}: {old} → {value}")

class MonitoredStage:
    state = AuditedAttribute()

    def __init__(self, name):
        self.name = name
        self.state = "idle"

import io, json as _json

class StructuredLogger:
    def __init__(self, component, stream=None):
        self._component = component
        self._stream = stream or io.StringIO()

    def info(self, event, **fields):
        record = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "level": "INFO",
            "component": self._component,
            "event": event,
            **fields
        }
        self._stream.write(_json.dumps(record) + "\n")

# ─── LAYER 6: DYNAMIC CLASS GENERATION ───────────────────────────────────────
# Lectures 41–43: __init_subclass__, metaclass, Protocol

_STAGE_REGISTRY = {}

class RegistrableStage:
    STAGE_NAME: str = ""

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        name = getattr(cls, "STAGE_NAME", "")
        if name:
            _STAGE_REGISTRY[name] = cls
            print(f"  [REGISTRY] Registered stage: {name!r}")

class ConsoleSink(RegistrableStage):
    STAGE_NAME = "ConsoleSink"

    def __init__(self, limit=5):
        self._limit = limit
        self._count = 0

    def consume(self, inputStream):
        for r in inputStream:
            if self._count < self._limit:
                print(f"  → {r['sensorId']} value={r['value']:.2f} "
                      f"normalized={r.get('normalized', 'N/A')}")
            self._count += 1
        print(f"  [ConsoleSink] Total records consumed: {self._count}")

# ─── ASSEMBLY AND EXECUTION ───────────────────────────────────────────────────

def buildAndRunPipeline(numRecords=200):
    print("=" * 60)
    print("PLUGGABLE ANALYTICS ENGINE — FINAL ASSEMBLY")
    print("=" * 60)

    tracemalloc.start()
    t_start = time.perf_counter()

    source = SyntheticSource(numRecords=numRecords, anomalyRate=0.1)
    stage1 = ThresholdFilter()
    stage2 = WindowNormalizer(windowSize=50)
    sink   = ConsoleSink(limit=5)
    logger = StructuredLogger("Pipeline")

    stream = source.stream()
    stream = stage1.transform(stream)
    stream = stage2.transform(stream)
    sink.consume(stream)

    elapsed_ms = (time.perf_counter() - t_start) * 1000
    _, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()

    logger.info("pipeline_complete",
                elapsed_ms=round(elapsed_ms, 2),
                peak_memory_kb=round(peak / 1024, 2),
                num_records=numRecords)

    print(f"\nElapsed: {elapsed_ms:.1f} ms")
    print(f"Peak memory: {peak / 1024:.1f} KB")
    print(f"Registered stages: {list(_STAGE_REGISTRY)}")

if __name__ == "__main__":
    buildAndRunPipeline(numRecords=200)
```

---

**Line-by-line walkthrough:**

**Lines 1–10:** Standard library imports. `tracemalloc` measures memory allocation; `inspect` enables runtime introspection; `typing` provides generic types and protocols.

**Lines 14–20 — AutoSlotMeta:** Metaclass that reads `__annotations__` from the class body and populates `__slots__` automatically. Any annotated attribute becomes a slot — no `__dict__`, bounded memory per instance.

**Lines 22–34 — SensorRecord:** Uses `AutoSlotMeta`. Six constructor parameters; `__repr__` for readable output. `SensorRecordDict` is the TypedDict counterpart — zero runtime cost, but type checkers use it to verify field access.

**Lines 38–53 — retryWithBackoff:** A decorator factory. The outer function captures `maxAttempts` and `baseDelay` in a closure. The inner `wrapper` retries on any exception, sleeping with exponential backoff plus random jitter to prevent thundering herd.

**Lines 55–70 — measuredStage:** A class-based decorator. `__call__` wraps the decorated generator function: it counts records, measures elapsed time, and prints throughput after the generator is exhausted.

**Lines 75–83 — SyntheticSource.stream:** A generator method. Each `yield` produces one record dictionary. The caller controls consumption rate — no buffering, no upfront allocation.

**Lines 85–100 — ThresholdFilter / WindowNormalizer:** Generator-based stage classes. Each `transform()` is decorated with `measuredStage` for automatic throughput measurement. WindowNormalizer uses `collections.deque(maxlen=50)` — bounded memory regardless of stream length.

**Lines 104–127 — AsyncCircuitBreaker:** Three-state machine (closed/open/half-open). On the `"open"` state, calls fail fast without hitting the downstream. After `resetTimeout` seconds, the state transitions to `"half-open"` and a probe request is allowed through.

**Lines 131–141 — AuditedAttribute:** Descriptor with `__set_name__`, `__get__`, and `__set__`. On every attribute write, if the value changed, it prints an audit log with old and new values.

**Lines 143–155 — StructuredLogger:** Writes every log event as a JSON dictionary with timestamp, level, component, and event name. Machine-readable, queryable by any log aggregation system.

**Lines 159–167 — RegistrableStage / __init_subclass__:** Base class for auto-registration. When any subclass is defined with a non-empty `STAGE_NAME`, it is automatically inserted into `_STAGE_REGISTRY`. No manual registration call required.

**Lines 169–180 — ConsoleSink:** Registers itself (via `RegistrableStage`) as `"ConsoleSink"`. `consume()` drives the generator pipeline to completion by iterating until exhausted.

**Lines 184–212 — buildAndRunPipeline:** Assembles and runs the full six-layer system. `tracemalloc.start()` begins memory tracking. The generator chain is assembled as `f(g(h(source)))`. `sink.consume()` drives execution. After completion, elapsed time and peak memory are logged via `StructuredLogger`.

---

## Example 44.2 — Architecture Self-Inspection at Startup

This example demonstrates the self-describing capability introduced in Stage 7: at startup, the pipeline inspects all registered stages and prints a human-readable manifest without any external documentation.

```python
# Example 44.2 — Pipeline manifest via inspect module

import inspect

def printPipelineManifest(registryDict):
    print("\n" + "=" * 60)
    print("PIPELINE STAGE MANIFEST")
    print("=" * 60)

    for stageName, cls in sorted(registryDict.items()):
        doc = inspect.getdoc(cls) or "(no docstring)"
        try:
            sig = inspect.signature(cls.__init__)
            params = [
                p for name, p in sig.parameters.items()
                if name != "self"
            ]
            param_str = ", ".join(
                f"{p.name}={p.default!r}"
                if p.default is not inspect.Parameter.empty
                else p.name
                for p in params
            )
        except (ValueError, TypeError):
            param_str = "(signature unavailable)"

        print(f"\nStage: {stageName}")
        print(f"  Class:      {cls.__module__}.{cls.__name__}")
        print(f"  Parameters: {param_str or '(none)'}")
        print(f"  Docstring:  {doc[:80]}{'…' if len(doc) > 80 else ''}")

# Usage: call at pipeline startup to validate and document all registered stages
# printPipelineManifest(_STAGE_REGISTRY)
```

**Walkthrough:**

**Line 5 — sorted(registryDict.items()):** Iterates registered stages in alphabetical order for consistent output.

**Lines 8–9 — inspect.getdoc(cls):** Returns the class docstring with indentation normalized. Returns `None` if no docstring exists.

**Lines 10–19 — inspect.signature(cls.__init__):** Returns the full parameter list for the constructor. We filter out `self`. For each parameter, we check `p.default is not inspect.Parameter.empty` to determine if a default value exists.

**Lines 21–25 — Formatted output:** Prints class module path, parameter list with defaults, and truncated docstring for each registered stage. This is the startup manifest — a self-generated inventory of available components.

---
