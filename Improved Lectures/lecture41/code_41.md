# Code — Lecture 41: Big Project Stage 8 Part 1 — Advanced Internals: Descriptors, Memory, and Protocol Enforcement

---

## Example 41.1 — AuditedAttribute descriptor for change tracking

```python
# Example 41.1
import time                                            # line 1
from typing import Any, List                           # line 2

class AuditedAttribute:                                # line 3
    """Descriptor that logs every attribute change.""" # line 4
    def __init__(self, label=None):                    # line 5
        self._label = label                            # line 6
        self._name = None                              # line 7
        self._privateName = None                       # line 8
        self.changeLog: List[dict] = []                # line 9

    def __set_name__(self, owner, name):               # line 10
        self._name = name                              # line 11
        self._privateName = f"_audited_{name}"         # line 12

    def __get__(self, obj, objtype=None):              # line 13
        if obj is None:                                # line 14
            return self                                # line 15
        return getattr(obj, self._privateName, None)   # line 16

    def __set__(self, obj, value):                     # line 17
        oldValue = self.__get__(obj, type(obj))        # line 18
        setattr(obj, self._privateName, value)         # line 19
        event = {                                      # line 20
            "attribute": self._name,                   # line 21
            "oldValue": oldValue,                      # line 22
            "newValue": value,                         # line 23
            "timestamp": time.time(),                  # line 24
            "label": self._label or self._name         # line 25
        }                                              # line 26
        self.changeLog.append(event)                   # line 27
        print(f"[AUDIT] {self._label or self._name}: {oldValue} → {value}")  # line 28

class MonitoredPipelineStage:                          # line 29
    recordsProcessed = AuditedAttribute(label="records_processed")  # line 30
    meanLatencyMs = AuditedAttribute(label="mean_latency_ms")  # line 31
    errorCount = AuditedAttribute(label="error_count")  # line 32

    def __init__(self, stageName):                     # line 33
        self.stageName = stageName                     # line 34
        self.recordsProcessed = 0                      # line 35
        self.meanLatencyMs = 0.0                       # line 36
        self.errorCount = 0                            # line 37

    def simulateProcessing(self, numRecords):          # line 38
        for i in range(numRecords):                    # line 39
            self.recordsProcessed = i + 1              # line 40  triggers descriptor __set__
            if i % 10 == 9:                            # line 41
                self.meanLatencyMs = round((i + 1) * 0.01, 3)  # line 42

if __name__ == "__main__":                             # line 43
    stage = MonitoredPipelineStage("ThresholdFilter")  # line 44
    stage.simulateProcessing(25)                       # line 45
    print(f"\n--- Audit log: {len(MonitoredPipelineStage.recordsProcessed.changeLog)} entries ---")  # line 46
    for entry in MonitoredPipelineStage.recordsProcessed.changeLog[:5]:  # line 47
        print(f"  {entry['attribute']}: {entry['oldValue']} → {entry['newValue']}")  # line 48
```

**Line-by-line explanation:**

- **Line 9:** `self.changeLog: List[dict] = []` — the change log is on the descriptor INSTANCE, not on each monitored object. This means all instances of the class that use this descriptor share one log. In production, you might make this per-instance by storing the log in the monitored object instead.
- **Line 10-12:** `__set_name__` is called by Python at class definition time — when the class body is processed and the descriptor is assigned to a class variable. At this moment, `name` is the attribute name as written in the class body. `self._privateName = f"_audited_{name}"` creates a mangled storage key to avoid name collisions.
- **Line 13-16:** `__get__` handles reading. The `if obj is None` check handles the case where the descriptor is accessed on the CLASS itself (not an instance): `MonitoredPipelineStage.recordsProcessed` returns the descriptor object, not a value. When accessed on an instance, it reads the stored value from the instance's `__dict__` using the private key.
- **Line 17-28:** `__set__` handles writing. It captures the old value before overwriting it, stores the new value, and appends a structured event to the change log. The `print` on line 28 simulates real logging — in production this would be a structured logger call.
- **Line 30-32:** Three class-level descriptor instances — one per monitored attribute. Each descriptor has its own `changeLog` list. Assigning a descriptor instance to a class variable automatically triggers `__set_name__` during class definition.
- **Lines 35-37:** `self.recordsProcessed = 0` in `__init__` triggers the descriptor's `__set__`, creating the first audit log entry with `oldValue=None` and `newValue=0`.
- **Line 40:** `self.recordsProcessed = i + 1` — every assignment inside `simulateProcessing` triggers the descriptor. Each triggers `__set__`, which logs a change event. For 25 records, this creates 25 entries in the change log.
- **Line 46:** `MonitoredPipelineStage.recordsProcessed.changeLog` — accessing the descriptor on the CLASS (not an instance) returns the descriptor itself (line 14-15), and then we access its `changeLog` attribute.

---

## Example 41.2 — Comparing dict, regular class, and __slots__ class memory usage

```python
# Example 41.2
import sys                                             # line 1
import tracemalloc                                     # line 2
import time                                            # line 3

class DictBasedRecord:                                 # line 4
    def __init__(self, sensorId, timestamp, value, unit, normalized, rollingAvg):  # line 5
        self.data = {                                  # line 6
            "sensorId": sensorId, "timestamp": timestamp,  # line 7
            "value": value, "unit": unit,              # line 8
            "normalized": normalized, "rollingAvg": rollingAvg  # line 9
        }                                              # line 10

class RegularClassRecord:                              # line 11
    def __init__(self, sensorId, timestamp, value, unit, normalized, rollingAvg):  # line 12
        self.sensorId = sensorId                       # line 13
        self.timestamp = timestamp                     # line 14
        self.value = value                             # line 15
        self.unit = unit                               # line 16
        self.normalized = normalized                   # line 17
        self.rollingAvg = rollingAvg                   # line 18

class SlottedRecord:                                   # line 19
    __slots__ = ("sensorId", "timestamp", "value", "unit", "normalized", "rollingAvg")  # line 20

    def __init__(self, sensorId, timestamp, value, unit, normalized, rollingAvg):  # line 21
        self.sensorId = sensorId                       # line 22
        self.timestamp = timestamp                     # line 23
        self.value = value                             # line 24
        self.unit = unit                               # line 25
        self.normalized = normalized                   # line 26
        self.rollingAvg = rollingAvg                   # line 27

def benchmark(RecordClass, n=500_000):                 # line 28
    tracemalloc.start()                                # line 29
    start = time.perf_counter()                        # line 30
    records = [RecordClass("TEMP_01", i * 0.001, 50.0 + i * 0.001, "C", 0.5, 0.52)  # line 31
               for i in range(n)]                      # line 32
    elapsed = time.perf_counter() - start              # line 33
    current, peak = tracemalloc.get_traced_memory()    # line 34
    tracemalloc.stop()                                 # line 35
    singleSize = sys.getsizeof(records[0])             # line 36
    return {                                           # line 37
        "class": RecordClass.__name__,                 # line 38
        "creationTimeSec": round(elapsed, 3),          # line 39
        "peakMemoryMB": round(peak / 1024 / 1024, 2), # line 40
        "singleInstanceBytes": singleSize              # line 41
    }                                                  # line 42

if __name__ == "__main__":                             # line 43
    for RecordClass in [DictBasedRecord, RegularClassRecord, SlottedRecord]:  # line 44
        result = benchmark(RecordClass)                # line 45
        print(f"{result['class']:25}: size={result['singleInstanceBytes']}B  "  # line 46
              f"peak={result['peakMemoryMB']}MB  "      # line 47
              f"time={result['creationTimeSec']}s")     # line 48
```

**Line-by-line explanation:**

- **Line 4-10:** `DictBasedRecord` wraps all attributes in a single dictionary — the worst case. The instance has a `__dict__` (for the instance itself) AND a separate dictionary for `self.data`. Double dictionary overhead.
- **Line 11-18:** `RegularClassRecord` uses normal Python attributes. Each instance has a `__dict__` containing all six attributes. Better than nested dicts, but still carries full dictionary overhead.
- **Line 19-27:** `SlottedRecord` with `__slots__`. No `__dict__` on instances. Python allocates a compact C-level slot vector. The constructor body is identical to `RegularClassRecord` — the only difference is the class-level `__slots__` declaration and the absence of a per-instance dictionary.
- **Line 29:** `tracemalloc.start()` enables Python's memory allocation tracer. This is more accurate than `sys.getsizeof` for measuring total allocation because it tracks all allocations, including internal C-level objects.
- **Line 34:** `tracemalloc.get_traced_memory()` returns `(current, peak)` — current allocated memory and peak allocated memory since `tracemalloc.start()`. The peak is the right metric because it captures the maximum memory in use during the list comprehension.
- **Line 36:** `sys.getsizeof(records[0])` returns the size of the object itself — its shell. For `DictBasedRecord`, this is the instance size WITHOUT the embedded dictionary (which is a separate allocation). For a more complete picture, you would add `sys.getsizeof(records[0].data)` for `DictBasedRecord`.

**Expected output on 64-bit Python:**
```
DictBasedRecord          : size=48B   peak=134.xx MB   time=0.xxx s
RegularClassRecord       : size=48B   peak=85.xx MB    time=0.xxx s
SlottedRecord            : size=72B   peak=54.xx MB    time=0.xxx s
```
Note: `sys.getsizeof` reports the shell size, which doesn't include referenced objects. The `tracemalloc` peak memory is the more meaningful number for comparing total allocation.
