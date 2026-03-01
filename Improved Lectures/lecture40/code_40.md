# Code — Lecture 40: Big Project Stage 7 — Observability and Introspection

---

## Example 40.1 — Structured Logger for the Pipeline

```python
# Example 40.1
import logging
import json
import time
import sys
from dataclasses import dataclass, asdict
from typing import Any, Dict

class StructuredLogger:
    def __init__(self, componentName, outputStream=sys.stdout):
        self._component = componentName
        self._stream = outputStream

    def _emit(self, level, event, **fields):
        record = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "level": level,
            "component": self._component,
            "event": event,
            **fields
        }
        self._stream.write(json.dumps(record) + "\n")
        self._stream.flush()

    def info(self, event, **fields):
        self._emit("INFO", event, **fields)

    def warning(self, event, **fields):
        self._emit("WARNING", event, **fields)

    def error(self, event, **fields):
        self._emit("ERROR", event, **fields)

class InstrumentedStage:
    def __init__(self, stageName):
        self._name = stageName
        self._logger = StructuredLogger(stageName)

    def process(self, records):
        startTime = time.perf_counter()
        out = []
        for r in records:
            processedValue = r["value"] * 1.05
            out.append({**r, "processed": processedValue})
        elapsedMs = (time.perf_counter() - startTime) * 1000
        self._logger.info("stage_completed",
            recordsIn=len(records),
            recordsOut=len(out),
            elapsedMs=round(elapsedMs, 2),
            throughputRps=round(len(out) / (elapsedMs / 1000), 1) if elapsedMs > 0 else 0)
        return out

if __name__ == "__main__":
    import random
    stage = InstrumentedStage("normalize")
    for batchNum in range(3):
        batch = [{"sensorId": f"S{i%5}", "value": random.gauss(50, 10)} for i in range(200)]
        result = stage.process(batch)
```

### Line-by-Line Explanation

**`class StructuredLogger:`**
The logger wraps output stream management. `componentName` is included in every emitted record — it identifies which part of the pipeline generated the event. Using `sys.stdout` as default allows redirection to files or log aggregation systems in production (replace with a file handle or a network stream handler).

**`def _emit(self, level, event, **fields):`**
Private method that builds the JSON record and writes it. `**fields` is a keyword-argument dict that unpacks into the record dict using `**fields`. This allows any number of additional context fields to be included without changing the method signature. Adding a new field — `self._logger.info("stage_completed", newField="value")` — requires no changes to `_emit`.

**`"timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())`**
ISO 8601 UTC timestamp. The Z suffix signals UTC (Zulu time). ISO 8601 format is the standard for log timestamps because it is lexicographically sortable and unambiguous across time zones. `time.gmtime()` returns UTC regardless of the server's local timezone — essential for distributed systems where servers may be in different time zones.

**`self._stream.write(json.dumps(record) + "\n") ; self._stream.flush()`**
`json.dumps(record)` serializes the record to a JSON string with all Python types (str, int, float, dict, list) converted to their JSON equivalents. The trailing `"\n"` makes each log record a newline-delimited JSON (NDJSON) record — the standard format for log streaming. `flush()` forces the write to the underlying I/O buffer immediately — without flush, buffered output may be delayed or lost if the process crashes.

**`self._logger.info("stage_completed", recordsIn=len(records), recordsOut=len(out), ...)`**
The structured log call from `InstrumentedStage.process`. "stage_completed" is the event name — a machine-readable identifier for this type of event, distinct from "stage_started", "stage_error", "circuit_opened". Each additional keyword argument becomes a JSON field. A monitoring system can alert on `WHERE event = "stage_completed" AND throughputRps < 1000` without any text parsing.

---

## Example 40.2 — Pipeline Introspection with inspect

```python
# Example 40.2
import inspect
from typing import Any

class ThresholdFilter:
    """Filters records outside [minVal, maxVal] range."""
    def process(self, records, minVal=20.0, maxVal=80.0):
        return [r for r in records if minVal <= r["value"] <= maxVal]

class NormalizationStage:
    """Normalizes record values to [0, 1] range."""
    def process(self, records, minVal=0.0, maxVal=100.0):
        span = maxVal - minVal
        return [{**r, "norm": (r["value"] - minVal) / span} for r in records]

class AnomalyDetector:
    """Detects statistical anomalies using z-score threshold."""
    def process(self, records, zThreshold=2.5):
        import statistics
        if len(records) < 2:
            return records
        mean = statistics.mean(r["value"] for r in records)
        stdev = statistics.stdev(r["value"] for r in records)
        return [{**r, "anomaly": abs(r["value"] - mean) / (stdev or 1) > zThreshold}
                for r in records]

def inspectPipeline(stages):
    print("=" * 60)
    print("PIPELINE STAGE INSPECTION REPORT")
    print("=" * 60)
    for stageObj in stages:
        cls = type(stageObj)
        processMethod = getattr(stageObj, "process", None)
        if processMethod is None:
            print(f"  WARNING: {cls.__name__} has no 'process' method")
            continue
        sig = inspect.signature(processMethod)
        params = [
            f"{name}={p.default!r}" if p.default is not inspect.Parameter.empty else name
            for name, p in sig.parameters.items()
            if name != "self"
        ]
        docSummary = (inspect.getdoc(cls) or "No docstring").split("\n")[0]
        print(f"\nStage: {cls.__name__}")
        print(f"  Description: {docSummary}")
        print(f"  Process signature: process({', '.join(params)})")
        print(f"  Parameters: {len(params)}")
        print(f"  Module: {cls.__module__}")

if __name__ == "__main__":
    pipeline = [ThresholdFilter(), NormalizationStage(), AnomalyDetector()]
    inspectPipeline(pipeline)
```

### Line-by-Line Explanation

**`cls = type(stageObj)`**
`type(obj)` returns the class of an object. For an instance of `ThresholdFilter`, `type(instance)` is `ThresholdFilter`. This is the entry point for class-level introspection: from the class, we can access its `__name__`, `__module__`, `__doc__`, and all methods.

**`processMethod = getattr(stageObj, "process", None)`**
`getattr(obj, name, default)` is the safe attribute access — returns `default` if the attribute does not exist rather than raising `AttributeError`. The `None` default allows us to check `if processMethod is None` and print a warning for stages that do not conform to the expected interface, rather than crashing the inspection.

**`sig = inspect.signature(processMethod)`**
`inspect.signature` returns a `Signature` object representing the function's parameter list. `sig.parameters` is an `OrderedDict` mapping parameter names to `Parameter` objects. Each `Parameter` has a `name`, `kind` (POSITIONAL_OR_KEYWORD, VAR_POSITIONAL, VAR_KEYWORD, KEYWORD_ONLY), `default` (the default value or `inspect.Parameter.empty` if no default), and `annotation` (type annotation or `inspect.Parameter.empty` if unannotated).

**`if p.default is not inspect.Parameter.empty else name`**
`inspect.Parameter.empty` is a sentinel that signals "no default value". Using `is not` (identity comparison) rather than `!= ` is correct because `empty` is a singleton sentinel object. If the parameter has a default, we format it as `"name=default_repr"`. If it has no default, we format it as just `"name"`. `p.default!r` uses `repr()` formatting — for string defaults, this includes the quotes, making the output correct Python syntax.

**`inspect.getdoc(cls)`**
`inspect.getdoc(obj)` returns the cleaned docstring — with indentation removed and leading/trailing whitespace stripped. This differs from `obj.__doc__`, which returns the raw docstring with whatever indentation the developer used. `(... or "No docstring").split("\n")[0]` takes only the first line — the summary line of the docstring.

**`print(f"  Module: {cls.__module__}")`**
`__module__` is the name of the module where the class was defined — e.g. `"__main__"` for classes defined in the executed script, or `"pipeline.stages.filter"` for classes in a package. In a large codebase with many stage definitions across multiple modules, the module name disambiguates classes with similar names.

---

## Example 40.3 — Pipeline Health Dashboard

```python
# Example 40.3
import time
import random
import tracemalloc
from collections import deque

class PipelineHealthDashboard:
    __slots__ = ("_stageThroughputs", "_circuitStates", "_cacheHitRates",
                 "_batchCount", "_alertCount", "_recentThroughputs", "_logger")

    def __init__(self, logger):
        self._stageThroughputs = {}
        self._circuitStates = {}
        self._cacheHitRates = {}
        self._batchCount = 0
        self._alertCount = 0
        self._recentThroughputs = deque(maxlen=10)
        self._logger = logger

    def recordBatch(self, stageThroughputs, circuitStates, cacheInfo, alertsThisBatch):
        self._batchCount += 1
        self._alertCount += alertsThisBatch
        self._stageThroughputs = stageThroughputs
        self._circuitStates = circuitStates
        self._cacheHitRates = cacheInfo
        if stageThroughputs:
            bottleneckRps = min(stageThroughputs.values())
            self._recentThroughputs.append(bottleneckRps)

    def healthReport(self, peakMemoryKb=None):
        openCircuits = [name for name, state in self._circuitStates.items() if state == "open"]
        bottleneck = min(self._stageThroughputs, key=self._stageThroughputs.get) if self._stageThroughputs else None
        meanThroughput = (sum(self._recentThroughputs) / len(self._recentThroughputs)
                          if self._recentThroughputs else 0)
        status = "HEALTHY" if not openCircuits else "DEGRADED"
        report = {
            "status": status,
            "batchCount": self._batchCount,
            "totalAlerts": self._alertCount,
            "openCircuits": openCircuits,
            "bottleneckStage": bottleneck,
            "bottleneckRps": self._stageThroughputs.get(bottleneck, 0) if bottleneck else 0,
            "meanRecentThroughputRps": round(meanThroughput, 1),
            "cacheHitRates": self._cacheHitRates,
            "peakMemoryKb": peakMemoryKb,
        }
        self._logger.info("pipeline_health", **report)
        return report

if __name__ == "__main__":
    logger = StructuredLogger("pipeline")
    dashboard = PipelineHealthDashboard(logger)

    for batchNum in range(5):
        stageThroughputs = {
            "filter": random.uniform(4000, 6000),
            "normalize": random.uniform(2000, 3000),
            "feature": random.uniform(800, 1200),
        }
        circuitStates = {"enrichmentService": "closed", "alertService": "closed"}
        if batchNum == 3:
            circuitStates["enrichmentService"] = "open"

        cacheInfo = {"calibration": round(random.uniform(0.85, 0.99), 3)}
        alerts = random.randint(0, 20)
        dashboard.recordBatch(stageThroughputs, circuitStates, cacheInfo, alerts)

        if batchNum % 2 == 0 or batchNum == 4:
            report = dashboard.healthReport(peakMemoryKb=round(random.uniform(50, 150), 1))
            print(f"Batch {batchNum+1}: {report['status']} | "
                  f"bottleneck={report['bottleneckStage']} @ {report['bottleneckRps']:.0f}rps | "
                  f"openCircuits={report['openCircuits']}")
```

### Line-by-Line Explanation

**`__slots__ = ("_stageThroughputs", ...)`**
The dashboard is a metrics aggregator — instantiated once and held for the pipeline's lifetime. `__slots__` ensures minimal memory overhead and catches attribute typos at the point of error. This applies the Stage 4 memory discipline to the Stage 7 observability infrastructure.

**`self._recentThroughputs = deque(maxlen=10)`**
Bounded by Stage 5 discipline: the dashboard keeps only the last 10 batch throughputs for mean computation. Without `maxlen`, this list would grow indefinitely with the number of batches. The mean of the last 10 is more useful than the mean of all batches — it reflects current behavior, not historical average.

**`def recordBatch(self, stageThroughputs, circuitStates, cacheInfo, alertsThisBatch):`**
Called after each batch completes. Accepts the current state from all pipeline components. Note that this method takes dictionaries by value — it replaces the stored state with the latest snapshot. This is appropriate because the dashboard represents the current state, not a history.

**`bottleneckRps = min(stageThroughputs.values())`**
Little's Law in practice: the pipeline's effective throughput is the minimum of all stage throughputs — the bottleneck stage. A stage processing 10,000 records/second cannot help if the downstream stage processes 500 records/second; the pipeline's output rate is 500 records/second. `min(stageThroughputs, key=stageThroughputs.get)` finds the key (stage name) with the minimum value (throughput) — using `dict.get` as the key function.

**`status = "HEALTHY" if not openCircuits else "DEGRADED"`**
Single boolean field capturing overall pipeline health. Any open circuit breaker means the pipeline is degraded — not all services are accessible, and some functionality is either missing or falling back to cached/stale data. Monitoring systems alert on `status == "DEGRADED"` without needing to understand what "open circuit" means.

**`self._logger.info("pipeline_health", **report)`**
The health report is emitted as a structured log event. The `**report` unpacks the report dict into keyword arguments, which `StructuredLogger.info` captures as `**fields` and includes in the JSON record. The entire dashboard state is captured in one log line — queryable, archivable, and streamable to any log aggregation system.

---
