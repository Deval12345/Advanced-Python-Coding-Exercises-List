# Code — Lecture 42: Big Project Stage 8 Part 2 — Advanced Internals: Metaclasses, Class Decorators, and Dynamic Generation

---

## Example 42.1 — Self-building stage registry with __init_subclass__

```python
# Example 42.1
from typing import Iterator                                # line 1

class PipelineStage:                                       # line 2
    """Base class for all pipeline stages. Subclasses     # line 3
    self-register by passing stageType= in their          # line 4
    class statement."""                                   # line 5
    _registry: dict = {}                                  # line 6

    def __init_subclass__(cls, stageType=None, **kwargs): # line 7
        super().__init_subclass__(**kwargs)               # line 8
        if stageType is not None:                         # line 9
            PipelineStage._registry[stageType] = cls      # line 10
            print(f"[REGISTRY] Registered: {stageType!r} → {cls.__name__}")  # line 11

    def transform(self, stream: Iterator) -> Iterator:    # line 12
        raise NotImplementedError                         # line 13


class ThresholdFilter(PipelineStage, stageType="threshold"):  # line 14
    def __init__(self, minVal=0.0, maxVal=100.0):         # line 15
        self.minVal = minVal                              # line 16
        self.maxVal = maxVal                              # line 17

    def transform(self, stream):                          # line 18
        return (r for r in stream if self.minVal <= r["value"] <= self.maxVal)  # line 19


class NormalizeTransform(PipelineStage, stageType="normalize"):  # line 20
    def __init__(self, minVal=0.0, maxVal=100.0):         # line 21
        self.minVal = minVal                              # line 22
        self.maxVal = maxVal                              # line 23

    def transform(self, stream):                          # line 24
        span = self.maxVal - self.minVal or 1.0           # line 25
        for r in stream:                                  # line 26
            yield {**r, "normalized": (r["value"] - self.minVal) / span}  # line 27


class MovingAverageTransform(PipelineStage, stageType="movingAverage"):  # line 28
    def __init__(self, windowSize=5):                     # line 29
        self.windowSize = windowSize                      # line 30
        self._window = []                                 # line 31

    def transform(self, stream):                          # line 32
        for r in stream:                                  # line 33
            self._window.append(r["value"])               # line 34
            if len(self._window) > self.windowSize:       # line 35
                self._window.pop(0)                       # line 36
            yield {**r, "rollingAvg": sum(self._window) / len(self._window)}  # line 37


def buildStageFromConfig(config: dict) -> PipelineStage: # line 38
    stageType = config["type"]                            # line 39
    cls = PipelineStage._registry.get(stageType)         # line 40
    if cls is None:                                       # line 41
        raise ValueError(f"Unknown stage type: {stageType!r}. "  # line 42
                         f"Available: {list(PipelineStage._registry)}")  # line 43
    kwargs = {k: v for k, v in config.items() if k != "type"}  # line 44
    return cls(**kwargs)                                  # line 45


if __name__ == "__main__":                               # line 46
    print("\n--- Registry contents ---")                 # line 47
    for key, cls in PipelineStage._registry.items():     # line 48
        print(f"  {key!r}: {cls.__name__}")              # line 49

    print("\n--- Build stages from config ---")          # line 50
    configs = [                                          # line 51
        {"type": "threshold", "minVal": 10.0, "maxVal": 90.0},  # line 52
        {"type": "normalize", "minVal": 10.0, "maxVal": 90.0},  # line 53
        {"type": "movingAverage", "windowSize": 3},      # line 54
    ]                                                    # line 55
    stages = [buildStageFromConfig(c) for c in configs]  # line 56
    print(f"Built {len(stages)} stages: {[type(s).__name__ for s in stages]}")  # line 57

    print("\n--- Test unknown type ---")                 # line 58
    try:                                                 # line 59
        buildStageFromConfig({"type": "unknown_stage"}) # line 60
    except ValueError as e:                              # line 61
        print(f"Expected error: {e}")                   # line 62
```

**Line-by-line explanation:**

- **Line 6:** `_registry: dict = {}` — class variable shared by all subclasses. The dict maps string keys to class objects. Declared at the base class level so all subclasses share one registry.
- **Line 7:** `__init_subclass__(cls, stageType=None, **kwargs)` — `cls` is the subclass being defined (not the base class). `stageType` receives the keyword argument passed in the class statement. `**kwargs` is required to forward any other keyword arguments to `super().__init_subclass__`.
- **Line 8:** `super().__init_subclass__(**kwargs)` — mandatory forwarding. Skipping this breaks cooperative multiple inheritance, which is why `**kwargs` must always be passed through.
- **Line 9-10:** Only register if `stageType` was actually provided. This allows intermediate abstract base classes to inherit from `PipelineStage` without accidentally registering with `stageType=None`.
- **Line 14:** `class ThresholdFilter(PipelineStage, stageType="threshold")` — the keyword argument in the class statement is routed to `__init_subclass__` automatically. Python extracts keyword arguments from the class bases and passes them to `__init_subclass__`. This is standard Python 3.6+ behavior.
- **Lines 20, 28:** Same pattern for `NormalizeTransform` and `MovingAverageTransform`. Each stage announces its own registry key. Adding a new stage type requires only defining the class with the right keyword argument.
- **Line 44:** `kwargs = {k: v for k, v in config.items() if k != "type"}` — strip the `type` key before passing to the constructor. The constructor doesn't accept `type` as a parameter.

---

## Example 42.2 — AutoSlotMeta: generating __slots__ and __init__ from annotations

```python
# Example 42.2
import sys                                               # line 1
import tracemalloc                                       # line 2


class AutoSlotMeta(type):                               # line 3
    """Metaclass that generates __slots__ and __init__  # line 4
    automatically from class annotations."""            # line 5

    def __new__(mcs, name, bases, namespace):           # line 6
        annotations = namespace.get("__annotations__", {})  # line 7
        fields = list(annotations.keys())               # line 8

        if fields:                                      # line 9
            namespace["__slots__"] = tuple(fields)      # line 10

            def make_init(field_list):                  # line 11
                def __init__(self, *args, **kwargs):    # line 12
                    if len(args) > len(field_list):     # line 13
                        raise TypeError(               # line 14
                            f"{name}() takes {len(field_list)} args")  # line 15
                    for field, value in zip(field_list, args):  # line 16
                        object.__setattr__(self, field, value)  # line 17
                    for field, value in kwargs.items(): # line 18
                        if field not in field_list:    # line 19
                            raise TypeError(f"Unknown field: {field}")  # line 20
                        object.__setattr__(self, field, value)  # line 21
                return __init__                         # line 22

            namespace["__init__"] = make_init(fields)  # line 23
            namespace.pop("__annotations__", None)      # line 24

        return super().__new__(mcs, name, bases, namespace)  # line 25


class SensorRecord(metaclass=AutoSlotMeta):             # line 26
    sensorId: str                                       # line 27
    timestamp: float                                    # line 28
    value: float                                        # line 29
    unit: str                                           # line 30
    normalized: float                                   # line 31
    rollingAvg: float                                   # line 32


class ManualSlottedRecord:                              # line 33
    __slots__ = ("sensorId", "timestamp", "value",     # line 34
                 "unit", "normalized", "rollingAvg")   # line 35

    def __init__(self, sensorId, timestamp, value,     # line 36
                 unit, normalized, rollingAvg):        # line 37
        self.sensorId = sensorId                       # line 38
        self.timestamp = timestamp                     # line 39
        self.value = value                             # line 40
        self.unit = unit                               # line 41
        self.normalized = normalized                   # line 42
        self.rollingAvg = rollingAvg                  # line 43


def benchmark(RecordClass, n=200_000):                 # line 44
    tracemalloc.start()                                # line 45
    records = [RecordClass("TEMP_01", i * 0.001,      # line 46
                            50.0 + i * 0.001, "C",    # line 47
                            0.5, 0.52)                # line 48
               for i in range(n)]                     # line 49
    _, peak = tracemalloc.get_traced_memory()          # line 50
    tracemalloc.stop()                                 # line 51
    return {                                           # line 52
        "class": RecordClass.__name__,                # line 53
        "peakMB": round(peak / 1024 / 1024, 2),       # line 54
        "shellBytes": sys.getsizeof(records[0]),       # line 55
        "hasDict": hasattr(records[0], "__dict__"),    # line 56
        "slots": getattr(RecordClass, "__slots__", None),  # line 57
    }                                                  # line 58


if __name__ == "__main__":                             # line 59
    print("--- Slot verification ---")                 # line 60
    r = SensorRecord("TEMP_01", 1.234, 50.1, "C", 0.5, 0.52)  # line 61
    print(f"sensorId: {r.sensorId}, value: {r.value}")  # line 62
    print(f"Has __dict__: {hasattr(r, '__dict__')}")   # line 63
    print(f"Slots: {SensorRecord.__slots__}")           # line 64

    print("\n--- Memory benchmark ---")                # line 65
    for cls in [SensorRecord, ManualSlottedRecord]:    # line 66
        result = benchmark(cls)                        # line 67
        print(f"{result['class']:25}: shell={result['shellBytes']}B  "  # line 68
              f"peak={result['peakMB']}MB  "           # line 69
              f"dict={result['hasDict']}  "            # line 70
              f"slots={result['slots']}")              # line 71
```

**Line-by-line explanation:**

- **Line 6:** `def __new__(mcs, name, bases, namespace)` — `mcs` is the metaclass itself (conventionally named `mcs` instead of `cls` to distinguish from the class being created). `name` is the string name of the new class. `bases` is a tuple of base classes. `namespace` is the dictionary of names defined in the class body.
- **Line 7:** `namespace.get("__annotations__", {})` — annotations are stored in the class body namespace under the key `"__annotations__"`. Using `.get` with a default handles classes with no annotations without error.
- **Line 10:** `namespace["__slots__"] = tuple(fields)` — injecting `__slots__` into the namespace before calling `type.__new__` is the key operation. If `__slots__` were added after the class was created, it would be too late — slots are processed during class construction by the C runtime.
- **Lines 11-22:** `make_init(field_list)` uses a closure to capture `field_list` so the generated `__init__` has access to the correct field names. Without the closure, `fields` could be mutated or go out of scope.
- **Line 17:** `object.__setattr__(self, field, value)` — using `object.__setattr__` bypasses any custom `__setattr__` that might be defined, ensuring the slot is set directly at the C level.
- **Line 23:** `namespace["__init__"] = make_init(fields)` — the generated function is injected into the namespace before `type.__new__` is called. The resulting class has this function as its `__init__`.
- **Line 24:** `namespace.pop("__annotations__", None)` — removing annotations from the namespace prevents conflicts with slots. When `__slots__` is declared, Python creates slot descriptors for those names. If annotations were still present, Python would also try to create class variables from them, which conflicts.
- **Line 26-32:** `class SensorRecord(metaclass=AutoSlotMeta)` — the class body contains only annotations. No `__slots__` line. No `__init__` body. The metaclass generates both. The resulting class is identical in behavior to `ManualSlottedRecord`.
- **Line 56:** `hasattr(records[0], "__dict__")` — verifies that instances have no `__dict__`. If this returns `True`, slots were not applied correctly.

