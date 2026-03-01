# Code — Lecture 43: Big Project Stage 8 Part 3 — Advanced Internals: Generics, Type-Safe Containers, and Final Assembly

---

## Example 43.1 — TypedDict schemas and typed stage signatures

```python
# Example 43.1
from typing import TypedDict, Iterator, TypeVar, Generic  # line 1
import json                                               # line 2
import time                                              # line 3
import tracemalloc                                       # line 4


class SensorRecord(TypedDict):                           # line 5
    sensorId: str                                        # line 6
    timestamp: float                                     # line 7
    value: float                                         # line 8
    unit: str                                            # line 9


class NormalizedRecord(SensorRecord, total=True):        # line 10
    normalized: float                                    # line 11


class FullRecord(NormalizedRecord, total=True):          # line 12
    rollingAvg: float                                    # line 13
    derivative: float                                    # line 14


RecordT = TypeVar("RecordT", bound=dict)                 # line 15


class TypedPipelineStage(Generic[RecordT]):              # line 16
    """Base class for stages that carry type information."""  # line 17
    def transform(self, stream: Iterator[RecordT]) -> Iterator[RecordT]:  # line 18
        raise NotImplementedError                        # line 19


class TypedThresholdFilter(TypedPipelineStage[SensorRecord]):  # line 20
    def __init__(self, minVal: float = 0.0, maxVal: float = 100.0):  # line 21
        self.minVal = minVal                             # line 22
        self.maxVal = maxVal                             # line 23

    def transform(                                       # line 24
        self, stream: Iterator[SensorRecord]             # line 25
    ) -> Iterator[SensorRecord]:                         # line 26
        return (r for r in stream if self.minVal <= r["value"] <= self.maxVal)  # line 27


class TypedNormalizeTransform(TypedPipelineStage[SensorRecord]):  # line 28
    def __init__(self, minVal: float = 0.0, maxVal: float = 100.0):  # line 29
        self.minVal = minVal                             # line 30
        self.maxVal = maxVal                             # line 31

    def transform(                                       # line 32
        self, stream: Iterator[SensorRecord]             # line 33
    ) -> Iterator[NormalizedRecord]:                     # line 34
        span = self.maxVal - self.minVal or 1.0          # line 35
        for r in stream:                                 # line 36
            yield {**r, "normalized": (r["value"] - self.minVal) / span}  # line 37


class TypedMovingAverageTransform(TypedPipelineStage[NormalizedRecord]):  # line 38
    def __init__(self, windowSize: int = 5):             # line 39
        self.windowSize = windowSize                     # line 40
        self._window: list = []                          # line 41

    def transform(                                       # line 42
        self, stream: Iterator[NormalizedRecord]         # line 43
    ) -> Iterator[FullRecord]:                           # line 44
        for r in stream:                                 # line 45
            self._window.append(r["value"])              # line 46
            if len(self._window) > self.windowSize:      # line 47
                self._window.pop(0)                      # line 48
            yield {**r, "rollingAvg": sum(self._window) / len(self._window),  # line 49
                   "derivative": 0.0}                   # line 50


def typed_build_pipeline(source, stages):               # line 51
    stream = source.stream()                             # line 52
    for stage in stages:                                 # line 53
        stream = stage.transform(stream)                 # line 54
    return stream                                        # line 55


def verify_schema(records: list, required_keys: list) -> bool:  # line 56
    for i, record in enumerate(records):                 # line 57
        for key in required_keys:                        # line 58
            if key not in record:                        # line 59
                print(f"FAIL: record[{i}] missing key '{key}'")  # line 60
                return False                             # line 61
    print(f"PASS: {len(records)} records, all required keys present")  # line 62
    return True                                          # line 63


if __name__ == "__main__":                              # line 64
    import random                                       # line 65

    class SimpleSyntheticSource:                        # line 66
        def __init__(self, n=50):                       # line 67
            self.n = n                                  # line 68
        def stream(self) -> Iterator[SensorRecord]:     # line 69
            for i in range(self.n):                     # line 70
                yield {"sensorId": "TEMP_01",           # line 71
                       "timestamp": i * 0.1,            # line 72
                       "value": random.gauss(50, 15),   # line 73
                       "unit": "C"}                     # line 74

    source = SimpleSyntheticSource(n=100)               # line 75
    stages = [                                          # line 76
        TypedThresholdFilter(minVal=10.0, maxVal=90.0), # line 77
        TypedNormalizeTransform(minVal=10.0, maxVal=90.0),  # line 78
        TypedMovingAverageTransform(windowSize=5),      # line 79
    ]                                                   # line 80
    pipeline = typed_build_pipeline(source, stages)    # line 81
    records = list(pipeline)                            # line 82
    print(f"Records through pipeline: {len(records)}")  # line 83

    verify_schema(records, ["sensorId", "timestamp", "value",  # line 84
                             "unit", "normalized", "rollingAvg", "derivative"])  # line 85
```

**Line-by-line explanation:**

- **Lines 5-9:** `SensorRecord(TypedDict)` — the base schema. `TypedDict` is imported from `typing`. At runtime, `SensorRecord` is just `dict`. The type checker reads these annotations to validate every key access. `total=True` (the default) means all keys are required.
- **Lines 10-11:** `NormalizedRecord(SensorRecord, total=True)` — inherits all keys from `SensorRecord` and adds `normalized`. TypedDict inheritance works by composition: a `NormalizedRecord` must have all five keys. Type checkers verify that only code that has seen a `NormalizeTransform` stage can pass a `NormalizedRecord` where a `FullRecord` is expected.
- **Lines 12-14:** `FullRecord` adds `rollingAvg` and `derivative`. In a real typed pipeline, the type annotations on each stage's `transform` return type document the schema growth precisely.
- **Line 15:** `TypeVar("RecordT", bound=dict)` — `bound=dict` means `RecordT` can be any type that is a subtype of `dict` (i.e., `SensorRecord`, `NormalizedRecord`, `FullRecord`). Without `bound`, any type could be used.
- **Line 16-19:** `TypedPipelineStage(Generic[RecordT])` — declaring the class as `Generic[RecordT]` lets type checkers parameterize it. `TypedThresholdFilter(TypedPipelineStage[SensorRecord])` means this stage works specifically with `SensorRecord` records.
- **Lines 33-34:** Note that the input and output types are both explicitly annotated. `Iterator[SensorRecord]` in, `Iterator[NormalizedRecord]` out. The type checker can verify that downstream stages accept `NormalizedRecord` and not just `SensorRecord`.
- **Lines 56-62:** `verify_schema` is the runtime complement to static type checking — it runs after the pipeline produces records and checks that all required keys are actually present. This is the defense-in-depth model: static analysis catches errors at development time, runtime verification catches any gaps at test or production time.

---

## Example 43.2 — Full end-to-end pipeline run with observability

```python
# Example 43.2
import json                                              # line 1
import time                                             # line 2
import tracemalloc                                      # line 3
import random                                           # line 4
from typing import Iterator                             # line 5


# ── Minimal self-contained version of all pipeline components ──

class PipelineStage:                                    # line 6
    _registry: dict = {}                                # line 7
    def __init_subclass__(cls, stageType=None, **kwargs):  # line 8
        super().__init_subclass__(**kwargs)              # line 9
        if stageType:                                   # line 10
            PipelineStage._registry[stageType] = cls    # line 11
    def transform(self, stream): raise NotImplementedError  # line 12


class SyntheticSource:                                  # line 13
    def __init__(self, sensorId="SENSOR_01", numRecords=100,  # line 14
                 mean=50.0, stddev=15.0, unit="C"):     # line 15
        self.sensorId = sensorId                        # line 16
        self.numRecords = numRecords                    # line 17
        self.mean = mean                                # line 18
        self.stddev = stddev                            # line 19
        self.unit = unit                                # line 20

    def stream(self) -> Iterator[dict]:                 # line 21
        for i in range(self.numRecords):               # line 22
            yield {"sensorId": self.sensorId,           # line 23
                   "timestamp": i * 0.1,               # line 24
                   "value": round(random.gauss(self.mean, self.stddev), 4),  # line 25
                   "unit": self.unit}                   # line 26


class ThresholdFilter(PipelineStage, stageType="threshold"):  # line 27
    def __init__(self, minVal=0.0, maxVal=100.0): self.minVal=minVal; self.maxVal=maxVal  # line 28
    def transform(self, stream): return (r for r in stream if self.minVal<=r["value"]<=self.maxVal)  # line 29


class NormalizeTransform(PipelineStage, stageType="normalize"):  # line 30
    def __init__(self, minVal=0.0, maxVal=100.0): self.minVal=minVal; self.maxVal=maxVal  # line 31
    def transform(self, stream):  # line 32
        span = self.maxVal - self.minVal or 1.0         # line 33
        for r in stream: yield {**r, "normalized": round((r["value"]-self.minVal)/span, 4)}  # line 34


class MovingAverageTransform(PipelineStage, stageType="movingAverage"):  # line 35
    def __init__(self, windowSize=5): self.windowSize=windowSize; self._window=[]  # line 36
    def transform(self, stream):  # line 37
        for r in stream:  # line 38
            self._window.append(r["value"]); self._window = self._window[-self.windowSize:]  # line 39
            yield {**r, "rollingAvg": round(sum(self._window)/len(self._window), 4)}  # line 40


class DerivativeTransform(PipelineStage, stageType="derivative"):  # line 41
    def __init__(self): self._prevValue=None; self._prevTimestamp=None  # line 42
    def transform(self, stream):  # line 43
        for r in stream:  # line 44
            if self._prevValue is None:  # line 45
                deriv = 0.0  # line 46
            else:  # line 47
                dt = r["timestamp"] - self._prevTimestamp or 1e-9  # line 48
                deriv = round((r["value"] - self._prevValue) / dt, 4)  # line 49
            self._prevValue = r["value"]  # line 50
            self._prevTimestamp = r["timestamp"]  # line 51
            yield {**r, "derivative": deriv}  # line 52


class ConsoleSink:  # line 53
    def __init__(self, limit=5): self.limit = limit  # line 54
    def consume(self, stream):  # line 55
        count = 0  # line 56
        for r in stream:  # line 57
            if count < self.limit:  # line 58
                print(f"  [{count+1}] id={r['sensorId']} val={r['value']:.3f}")  # line 59
            count += 1  # line 60
        print(f"ConsoleSink: consumed {count} records (preview: {min(count, self.limit)}).")  # line 61
        return count  # line 62


class JsonFileSink:  # line 63
    def __init__(self, filePath): self.filePath = filePath; self._count = 0  # line 64
    def consume(self, stream):  # line 65
        with open(self.filePath, "w") as f:  # line 66
            for r in stream:  # line 67
                f.write(json.dumps(r) + "\n")  # line 68
                self._count += 1  # line 69
        print(f"JsonFileSink: wrote {self._count} records to {self.filePath}.")  # line 70
        return self._count  # line 71


class ThresholdAlertSink:  # line 72
    def __init__(self, threshold=0.7, callbackFn=None):  # line 73
        self.threshold = threshold  # line 74
        self.callbackFn = callbackFn or (lambda r: None)  # line 75
        self._alertCount = 0  # line 76
    def consume(self, stream):  # line 77
        count = 0  # line 78
        for r in stream:  # line 79
            count += 1  # line 80
            if r.get("normalized", 0) > self.threshold:  # line 81
                self._alertCount += 1  # line 82
                self.callbackFn(r)  # line 83
        print(f"ThresholdAlertSink: {self._alertCount} alerts (threshold={self.threshold:.2f}).")  # line 84
        return self._alertCount  # line 85


class MultiSink:  # line 86
    def __init__(self, sinks): self.sinks = sinks  # line 87
    def consume(self, stream):  # line 88
        records = list(stream)  # line 89
        for sink in self.sinks:  # line 90
            sink.consume(iter(records))  # line 91


def buildPipeline(source, stages):  # line 92
    stream = source.stream()  # line 93
    for stage in stages: stream = stage.transform(stream)  # line 94
    return stream  # line 95


def verifyOutputFile(jsonPath, requiredFields):  # line 96
    with open(jsonPath) as f:  # line 97
        records = [json.loads(line) for line in f if line.strip()]  # line 98
    for i, r in enumerate(records):  # line 99
        for key in requiredFields:  # line 100
            if key not in r:  # line 101
                print(f"FAIL: record[{i}] missing '{key}'")  # line 102
                return  # line 103
    print(f"PASS: {len(records)} records verified, all fields present.")  # line 104


def printPipelineReport(stagenames, recordCount, elapsedMs, peakMemMB, changeEvents):  # line 105
    print("\n=== Pipeline Report ===")  # line 106
    print(f"Stages:    {' → '.join(stagenames)}")  # line 107
    print(f"Records:   {recordCount}")  # line 108
    print(f"Elapsed:   {elapsedMs:.1f} ms")  # line 109
    print(f"Peak mem:  {peakMemMB:.2f} MB")  # line 110
    print(f"Audit:     {changeEvents} change events (N/A in this minimal build)")  # line 111
    print("=======================")  # line 112


if __name__ == "__main__":  # line 113
    random.seed(42)  # line 114

    source = SyntheticSource(sensorId="TEMP_01", numRecords=200, mean=50.0, stddev=20.0)  # line 115
    stages = [  # line 116
        ThresholdFilter(minVal=5.0, maxVal=95.0),  # line 117
        NormalizeTransform(minVal=5.0, maxVal=95.0),  # line 118
        MovingAverageTransform(windowSize=5),  # line 119
        DerivativeTransform(),  # line 120
    ]                                                   # line 121
    sinks = [  # line 122
        ConsoleSink(limit=5),  # line 123
        JsonFileSink("final_output.json"),  # line 124
        ThresholdAlertSink(threshold=0.7),  # line 125
    ]  # line 126
    multi = MultiSink(sinks)  # line 127

    tracemalloc.start()  # line 128
    start = time.perf_counter()  # line 129
    pipeline = buildPipeline(source, stages)  # line 130
    multi.consume(pipeline)  # line 131
    elapsed = (time.perf_counter() - start) * 1000  # line 132
    _, peakBytes = tracemalloc.get_traced_memory()  # line 133
    tracemalloc.stop()  # line 134

    jsonSink = sinks[1]  # line 135
    required = ["sensorId", "timestamp", "value", "unit",  # line 136
                "normalized", "rollingAvg", "derivative"]  # line 137
    verifyOutputFile("final_output.json", required)  # line 138

    stageNames = [type(s).__name__ for s in stages]  # line 139
    printPipelineReport(stageNames, jsonSink._count, elapsed,  # line 140
                        peakBytes / 1024 / 1024, 0)  # line 141
```

**Line-by-line explanation:**

- **Lines 6-12:** Minimal self-contained `PipelineStage` with the self-building registry from Lecture 42. All four concrete stage classes register themselves.
- **Lines 27-52:** All four stage classes in compact form. Each uses the `stageType=` keyword to register. `DerivativeTransform` on lines 41-51 carries per-instance state (`_prevValue`, `_prevTimestamp`) to compute rate of change between consecutive records.
- **Lines 86-90:** `MultiSink.consume` materializes the stream into a list first (line 89), then delivers the same records to all sinks independently. This is necessary because generators can only be consumed once — after the first sink iterates the stream, it is exhausted.
- **Lines 92-94:** `buildPipeline` — the composed generator chain. Each stage wraps the previous stage's output as a new generator. No data flows until something iterates the final stream.
- **Lines 96-103:** `verifyOutputFile` opens the JSON file, parses each line, and checks for all required field names. If any record is missing a field, it prints a failure message and returns early. This is the runtime verification complement to static type checking.
- **Line 113:** `random.seed(42)` — ensures reproducible results across runs for testing and demonstration.
- **Lines 128-133:** `tracemalloc.start()` / `tracemalloc.get_traced_memory()` — measures peak memory during the pipeline run. The pipeline runs while tracemalloc is active, so all allocations (including the materialized record list in MultiSink) are captured.
- **Lines 135-138:** The JSON sink object is retrieved by index to access its record count. In a production system, each sink would expose a `stats()` method returning a structured result.
- **Lines 139-141:** `printPipelineReport` receives all the collected metrics and formats them into a structured summary. The `changeEvents=0` is a placeholder — in the full system with `AuditedAttribute`, this would come from the descriptor's change log count.

