# Slides — Lecture 42: Big Project Stage 8 Part 2 — Advanced Internals: Metaclasses, Class Decorators, and Dynamic Generation

---

## Slide 1 — Title

**Lecture 42: Big Project Stage 8 Part 2**
Advanced Internals: Metaclasses, Class Decorators, and Dynamic Generation

*Intercepting class creation — not just attribute access*

---

## Slide 2 — How Python Creates a Class

**Class creation is a function call**

```
class Foo:
    x = 1
```

Python does this internally:
```
namespace = {"x": 1}
Foo = type("Foo", (object,), namespace)
```

- `type` is the default **metaclass** — the callable that creates class objects
- You can substitute a custom callable at this step
- Everything in the class body passes through it

---

## Slide 3 — Three Levels of Class Interception

| Tool | When it fires | Use for |
|------|--------------|---------|
| Class decorator | After class is assembled | Post-processing, injecting methods |
| `__init_subclass__` | When any subclass is defined | Auto-registration, plugin systems |
| Metaclass | During class construction | Structural enforcement, slot generation |

**Rule:** Use the mildest tool that solves the problem

---

## Slide 4 — `__init_subclass__`: The Self-Building Registry

**Problem:** maintain a dict mapping string names to stage classes

**Old way:** manually register every new stage — tedious and error-prone

**Better way:**
```python
class PipelineStage:
    _registry = {}
    def __init_subclass__(cls, stageType=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if stageType:
            PipelineStage._registry[stageType] = cls
```

Subclasses self-register:
```python
class ThresholdFilter(PipelineStage, stageType="threshold"):
    ...
```

**The registry builds itself — at import time, automatically**

---

## Slide 5 — Factory from the Registry

```python
def buildStageFromConfig(config):
    stageType = config["type"]
    cls = PipelineStage._registry.get(stageType)
    if cls is None:
        raise ValueError(f"Unknown stage type: {stageType}")
    return cls(**{k: v for k, v in config.items() if k != "type"})
```

- No `if/elif` chain
- No explicit imports required
- Adding a new stage: just define the class with `stageType=`
- Removing a stage: delete the class

*This is how Django models, SQLAlchemy tables, and Pydantic validators are registered*

---

## Slide 6 — Class Decorators: Post-Assembly Transformation

**A class decorator is just a function:**
```python
def monitor_fields(cls):
    for name in cls.__annotations__:
        setattr(cls, name, AuditedAttribute(label=name))
    return cls

@monitor_fields
class MonitoredStage:
    recordsProcessed: int
    errorCount: int
```

- `cls.__annotations__` contains the annotated names
- `setattr` injects the descriptor into the class
- The original class is returned, enhanced

**Identical to `@dataclass` — which generates `__init__`, `__repr__`, `__eq__` this way**

---

## Slide 7 — Metaclass: Enforcement at Definition Time

**ABCMeta:** raises error at instantiation time
**Custom metaclass:** raises error at class definition time

```python
class PipelineMeta(type):
    def __new__(mcs, name, bases, namespace):
        if bases and "transform" not in namespace:
            raise TypeError(
                f"{name} must implement transform()"
            )
        return super().__new__(mcs, name, bases, namespace)

class PipelineStage(metaclass=PipelineMeta):
    pass
```

Broken class fails at **import time**, not at runtime

---

## Slide 8 — When Metaclass vs ABC?

| | ABCMeta | Custom Metaclass |
|--|---------|-----------------|
| Error timing | Instantiation | Class definition / import |
| Complexity | Low | Higher |
| Use case | Abstract methods | Structural rules, registry, slot generation |
| Production examples | Django abstract models | SQLAlchemy `DeclarativeMeta`, marshmallow |

**Choose metaclass when definition-time errors matter**

---

## Slide 9 — The Duplication Problem with `__slots__`

```python
class SensorRecord:
    __slots__ = ("sensorId", "timestamp", "value")  # ← must match annotations
    sensorId: str
    timestamp: float
    value: float
```

- Field names written **twice** — annotation and slot declaration
- Adding a field? Must update both
- Forgetting `__slots__`? No error — silently falls back to `__dict__`

**A metaclass can eliminate this entirely**

---

## Slide 10 — AutoSlotMeta: Generating `__slots__` from Annotations

```python
class AutoSlotMeta(type):
    def __new__(mcs, name, bases, namespace):
        annotations = namespace.get("__annotations__", {})
        namespace["__slots__"] = tuple(annotations.keys())
        # Remove annotations to avoid conflict with slots
        namespace.pop("__annotations__", None)
        return super().__new__(mcs, name, bases, namespace)
```

Usage — no `__slots__` line needed:
```python
class SensorRecord(metaclass=AutoSlotMeta):
    sensorId: str
    timestamp: float
    value: float
    unit: str
```

**Annotate once — slots generated automatically**

---

## Slide 11 — Dynamic `__init__` Generation

```python
class AutoSlotMeta(type):
    def __new__(mcs, name, bases, namespace):
        annotations = namespace.get("__annotations__", {})
        fields = list(annotations.keys())
        namespace["__slots__"] = tuple(fields)

        def __init__(self, *args):
            for field, value in zip(fields, args):
                setattr(self, field, value)

        namespace["__init__"] = __init__
        namespace.pop("__annotations__", None)
        return super().__new__(mcs, name, bases, namespace)
```

- `__init__` generated from field list at class creation time
- No manual `__init__` body required
- Adding/removing a field: update the annotation, everything else follows

---

## Slide 12 — Stage 8 Part 2 Summary: What We Built

**Three class-level interception tools:**

1. `__init_subclass__` → self-building stage registry
   - Any stage defined anywhere auto-registers with a string key

2. Class decorator `@monitor_fields` → automatic descriptor injection
   - Annotate a field once, get audited attribute tracking automatically

3. Metaclass `PipelineMeta` → definition-time protocol enforcement
   - Missing `transform()` raises error at import, not runtime

4. `AutoSlotMeta` → annotation-driven slot + init generation
   - Annotate fields once, get memory-efficient slotted class automatically

*Next: Lecture 43 — generics, type-safe containers, and final project assembly*

