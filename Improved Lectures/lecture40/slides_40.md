# Slides — Lecture 40: Big Project Stage 7 — Observability and Introspection

---

## Slide 1 — Stage 7 Goal: Observable Pipeline

**See what's happening inside — at the right granularity**

- Monitoring tells you THAT something is wrong
- Observability tells you WHY: which stage, which metric, which trend
- Three layers:
  1. **Structured logging** — machine-readable event records (JSON)
  2. **Introspection** — pipeline self-describes its own structure
  3. **Health dashboard** — aggregated operational snapshot

---

## Slide 2 — Unstructured vs Structured Logging

**The difference between human-readable and machine-queryable**

Unstructured:
```
2024-01-15 10:23:45 INFO Processed 500 records in normalize stage in 2.3s
```
→ Requires regex to extract "500", "normalize", "2.3"
→ Format changes → regex breaks → monitoring breaks silently

Structured (JSON):
```json
{"timestamp": "2024-01-15T10:23:45Z", "level": "INFO",
 "event": "stage_completed", "stage": "normalize",
 "records": 500, "elapsed_ms": 2300, "throughputRps": 217.4}
```
→ Any field queryable directly: `WHERE stage = "normalize" AND elapsed_ms > 500`
→ Format evolves by adding fields — backward compatible

---

## Slide 3 — Example 40.1: StructuredLogger + InstrumentedStage

**Every pipeline event emitted as a JSON dictionary**

```python
class StructuredLogger:
    def _emit(self, level, event, **fields):
        record = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "level": level,
            "component": self._component,
            "event": event,
            **fields
        }
        self._stream.write(json.dumps(record) + "\n")

class InstrumentedStage:
    def process(self, records):
        # ... do work ...
        self._logger.info("stage_completed",
            recordsIn=len(records),
            recordsOut=len(out),
            elapsedMs=round(elapsedMs, 2),
            throughputRps=round(throughput, 1))
```

- `**fields` forwards any keyword argument as a JSON field — extensible without changing the interface
- `stream.flush()`: essential for real-time log visibility in production containers

---

## Slide 4 — What Structured Logs Enable

**Queryable metrics without a separate metrics system**

With structured logs, a monitoring system can:
```sql
-- Alert when normalize stage is slow
SELECT avg(elapsed_ms)
WHERE event = "stage_completed" AND component = "normalize"
HAVING avg(elapsed_ms) > 500

-- Track circuit breaker activity
SELECT count(*), circuitName
WHERE event = "circuit_state_change" AND newState = "open"
GROUP BY circuitName

-- Monitor filter effectiveness over time
SELECT timestamp, recordsOut::float / recordsIn as passRate
WHERE event = "stage_completed" AND component = "filter"
```

- No separate metrics pipeline required: logs ARE the metrics
- Used by: Datadog Log Management, AWS CloudWatch Logs Insights, Splunk

---

## Slide 5 — Runtime Introspection with inspect

**The pipeline describes its own structure from live objects**

```python
import inspect

def inspectPipeline(stages):
    for stageObj in stages:
        cls = type(stageObj)
        processMethod = getattr(stageObj, "process", None)
        sig = inspect.signature(processMethod)
        params = [
            f"{name}={p.default!r}" if p.default is not inspect.Parameter.empty else name
            for name, p in sig.parameters.items()
            if name != "self"
        ]
        docSummary = (inspect.getdoc(cls) or "No docstring").split("\n")[0]
        print(f"Stage: {cls.__name__}")
        print(f"  Description: {docSummary}")
        print(f"  Signature: process({', '.join(params)})")
```

Output for `ThresholdFilter`:
```
Stage: ThresholdFilter
  Description: Filters records outside [minVal, maxVal] range.
  Signature: process(records, minVal=20.0, maxVal=80.0)
```

---

## Slide 6 — What inspect Enables

**Self-documenting pipelines — documentation cannot go out of sync**

| Use case | Without inspect | With inspect |
|----------|----------------|--------------|
| Documentation | Separate doc file, drifts from code | Generated from live objects |
| Config validation | Manual check, error-prone | Automatic signature verification |
| Plugin discovery | Hard-coded registry | `inspect.getmembers(module, inspect.isclass)` |
| API schema generation | Manual OpenAPI YAML | Automatic from function signatures (FastAPI) |
| Test fixture matching | Hard-coded | Pytest auto-matches by parameter name |

- `inspect.signature(fn)` — parameter names and defaults
- `inspect.getdoc(cls)` — cleaned docstring
- `inspect.getmembers(obj, predicate)` — filtered attribute list
- `inspect.isclass`, `inspect.ismethod`, `inspect.isfunction` — type predicates

---

## Slide 7 — Health Dashboard: Aggregated Operational View

**All pipeline metrics in one structured snapshot**

```json
{
  "status": "DEGRADED",
  "batchCount": 47,
  "totalAlerts": 312,
  "openCircuits": ["enrichmentService"],
  "bottleneckStage": "feature",
  "bottleneckRps": 943.2,
  "meanRecentThroughputRps": 1012.5,
  "cacheHitRates": {"calibration": 0.987},
  "peakMemoryKb": 87.3
}
```

- `status`: HEALTHY or DEGRADED — one-glance triage
- `openCircuits`: tells operator which services are down
- `bottleneckStage`: the stage limiting overall throughput (Little's Law)
- `peakMemoryKb`: memory trend visible per batch

---

## Slide 8 — Example 40.3: PipelineHealthDashboard

**Aggregates metrics per batch, emits structured health report**

```python
class PipelineHealthDashboard:
    __slots__ = ("_stageThroughputs", "_circuitStates", "_cacheHitRates",
                 "_batchCount", "_alertCount", "_recentThroughputs", "_logger")

    def recordBatch(self, stageThroughputs, circuitStates, cacheInfo, alertsThisBatch):
        self._batchCount += 1
        self._stageThroughputs = stageThroughputs
        if stageThroughputs:
            self._recentThroughputs.append(min(stageThroughputs.values()))

    def healthReport(self, peakMemoryKb=None):
        bottleneck = min(self._stageThroughputs, key=self._stageThroughputs.get)
        openCircuits = [n for n, s in self._circuitStates.items() if s == "open"]
        status = "HEALTHY" if not openCircuits else "DEGRADED"
        self._logger.info("pipeline_health", status=status, ...)
```

- `_recentThroughputs = deque(maxlen=10)`: bounded Stage 5 discipline
- `__slots__`: lean dashboard object — Stage 4 discipline
- `_logger.info("pipeline_health", ...)`: structured log — Stage 7 discipline

---

## Slide 9 — Monitoring vs. Observability

**Two timescales, two purposes**

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| Timescale | Batch-level snapshots | Event-level timeline |
| Purpose | Is it healthy NOW? | WHY did it fail? |
| Format | Health dashboard JSON | Structured event log |
| Consumer | Real-time alerting | Forensic investigation |
| Example | "Status: DEGRADED" | Circuit open event + retry counts + failure reason |

**You need both:**
- Dashboard: operator checks health during shift
- Structured logs: engineer diagnoses last night's incident this morning

---

## Slide 10 — Course Integration: What Stage 7 Uses

**Every concept from the course appears in observability**

| Stage 7 component | Earlier concept |
|-------------------|----------------|
| `StructuredLogger` using dict | Data model (Lecture 1–2) |
| `inspect.signature` | Python object model (Lecture 1) |
| `deque(maxlen=10)` in dashboard | Bounded memory (Lecture 38) |
| `__slots__` on dashboard | Memory discipline (Lecture 17 + 38) |
| `@measuredStage` throughput → dashboard | Decorator + generator (Lectures 8–10) |
| Circuit state → dashboard | Resilience layer (Lecture 39) |

---

## Slide 11 — Lecture 40 Key Principles

**What to carry into Stage 8**

- Structured logging: JSON dict per event; fields queryable without parsing
- Every log call: event name + context fields → monitoring can alert on any field
- `inspect.signature`: extract parameter names and defaults from live methods
- Self-describing pipeline: documentation generated from code, always current
- Health dashboard: aggregated snapshot — bottleneck, open circuits, memory, alerts
- Bottleneck = `min(stageThroughputs.values())` — Little's Law applied operationally
- Next: Stage 8 — advanced internals: descriptors, metaclasses, typed generics

---
