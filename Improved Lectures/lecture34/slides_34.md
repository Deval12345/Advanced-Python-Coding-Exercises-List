# Slides — Lecture 34: Big Project Stage 2 — Resource Lifecycle Architecture

---

## Slide 1: Where We Are

**Big Project: Pluggable Analytics Engine**

Stage 1 (Lecture 33): Protocols defined
- Source: `stream()` generator
- Stage: `process(inputStream)` generator
- Sink: `consume(stream)` method

Stage 2 (Today): Make it production-ready
- Resource lifecycle management
- Configuration validation
- Metrics and retry

**The gap between "works" and "runs reliably" is filled by these patterns.**

---

## Slide 2: The Leak Problem

**What happens when a pipeline fails mid-run?**

Without cleanup:
- Database connections stay open
- File handles stay open
- Network sockets stay open

Over time:
- Connection pool exhausted
- OS file descriptor limit hit
- New requests refused

**A transient error becomes a permanent degradation.**

---

## Slide 3: Context Manager Protocol

```python
class PipelineContext:
    def __enter__(self):
        self._startTime = time.perf_counter()
        return self

    def __exit__(self, excType, excVal, excTb):
        elapsed = time.perf_counter() - self._startTime
        if excType is not None:
            print(f"FAILED after {elapsed:.3f}s: {excVal}")
        else:
            print(f"Completed: {self._recordCount} records in {elapsed:.3f}s")
        return False  # never suppress exceptions
```

`__exit__` is called **regardless of how the block exits.**

---

## Slide 4: Using PipelineContext

```python
with PipelineContext(source, pipelineId="sensor_ingest") as ctx:
    ctx.run(stages=[filter_stage, normalize_stage], sink=file_sink)
```

- Pipeline starts: `__enter__` records start time, prints startup
- Pipeline runs: `run()` chains generators, counts records
- Pipeline ends: `__exit__` logs duration and record count
- Pipeline crashes: `__exit__` logs error, re-raises exception

**The `with` statement is a contract, not a convention.**

---

## Slide 5: Configuration Without Validation

```python
# This silently creates invalid state
config = PipelineConfig(
    samplingRateHz=-5,    # negative rate: nonsense
    windowSize=0,          # zero window: ZeroDivisionError later
    filterMin=90.0,        # inverted bounds: no records pass
    filterMax=10.0
)
# Error surfaces 3 stages downstream, stack trace useless
```

**The error site and the cause are separated by time and stack frames.**

---

## Slide 6: Descriptor Protocol

```python
class PositiveFloat:
    def __set_name__(self, owner, name):
        self._name = f"_{name}"        # called at class definition time

    def __get__(self, obj, objtype=None):
        if obj is None: return self    # class-level access
        return getattr(obj, self._name, None)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError(...)
        if value <= 0:
            raise ValueError(...)
        setattr(obj, self._name, float(value))
```

**Intercepts assignment. Validates immediately. Stores under private name.**

---

## Slide 7: PipelineConfig with Descriptors

```python
class PipelineConfig:
    samplingRateHz = PositiveFloat()
    windowSize     = BoundedInt(1, 10000)
    filterMin      = PositiveFloat()
    filterMax      = PositiveFloat()
```

```python
config = PipelineConfig(100.0, 10, 20.0, 80.0)  # valid — works

config.windowSize = 0      # ValueError immediately: "must be in [1, 10000]"
config.samplingRateHz = "fast"  # TypeError immediately: "must be numeric"
```

**Error at the boundary. Clear message. No debugging required.**

---

## Slide 8: Why Measure?

**Without metrics, you are guessing:**
- Which stage is the bottleneck?
- What is the throughput?
- How many records are being filtered out?
- Is per-record latency acceptable?

**With metrics, you know:**
- `ThresholdFilter: in=10000 out=7823 throughput=45000rps meanLatency=0.022ms`
- `Normalizer: in=7823 out=7823 throughput=62000rps meanLatency=0.016ms`

**The bottleneck is obvious. Optimization is targeted.**

---

## Slide 9: The @measuredStage Decorator

```python
def measuredStage(stageName):
    def decorator(processMethod):
        @functools.wraps(processMethod)
        def wrapper(self, inputStream):
            metrics = StageMetrics(stageName=stageName)
            for record in processMethod(self, inputStream):
                recStart = time.perf_counter()
                yield record
                metrics.latenciesMs.append(
                    (time.perf_counter() - recStart) * 1000
                )
            # print summary after generator exhausted
        return wrapper
    return decorator
```

**The stage does not change. The metrics are additive and removable.**

---

## Slide 10: Retry Decorator

```python
def retryOnFailure(maxAttempts=3, delaySeconds=1.0):
    def decorator(streamMethod):
        @functools.wraps(streamMethod)
        def wrapper(self):
            for attempt in range(maxAttempts):
                try:
                    yield from streamMethod(self)
                    return
                except IOError as e:
                    if attempt < maxAttempts - 1:
                        time.sleep(delaySeconds)
                    else:
                        raise
        return wrapper
    return decorator
```

**Transient failures become invisible. Permanent failures still propagate.**

---

## Slide 11: The Three Layers

| Layer | Pattern | Problem Solved |
|---|---|---|
| Lifecycle | Context Manager `__enter__`/`__exit__` | Resource leaks on failure |
| Configuration | Descriptor `__get__`/`__set__` | Invalid config causes distant crash |
| Observability | Decorator wrapping generator | No visibility into performance |

**Each layer is independent. All three compose cleanly.**

---

## Slide 12: Lecture 34 Summary

**Context Managers:**
- `__enter__` / `__exit__` guarantee cleanup
- `return False` in `__exit__` — never silently suppress exceptions

**Descriptors:**
- `__set_name__` receives attribute name at class definition
- Validates at assignment, not at use — fail fast
- Store value under `_name` to avoid infinite recursion

**Decorators on generators:**
- Wrap the generator method, intercept each yielded value
- Metrics are additive — stage logic unchanged
- Retry wraps the whole generator with exception handling

**Next: Concurrency — reading 10 sensors simultaneously with asyncio.**
