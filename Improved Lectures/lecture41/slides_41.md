# Slides — Lecture 41: Big Project Stage 8 Part 1 — Advanced Internals: Descriptors, Memory, and Protocol Enforcement

---

## Slide 1 — Lecture Overview

**Production Hardening Through Python Internals**

- The pipeline works — now we make it efficient, self-auditing, and self-enforcing
- Three advanced mechanisms: descriptor-based monitoring, memory layout, protocol verification
- Descriptors: intercept attribute access to add automatic audit trails
- `__slots__`: reduce per-record memory overhead from ~70 bytes to ~40 bytes at scale
- ABCs / `@runtime_checkable`: enforce protocols at startup, not deep in call stacks

---

## Slide 2 — Descriptor Protocol: The Machinery Behind Python Attributes

**How `property`, `classmethod`, `staticmethod` actually work**

- A descriptor is any class that defines `__get__`, `__set__`, or `__delete__`
- When a descriptor instance is a class attribute, Python routes all attribute access through those methods
- `obj.attr` → `Descriptor.__get__(obj, type(obj))`
- `obj.attr = val` → `Descriptor.__set__(obj, val)`
- One descriptor class handles monitoring for ALL attributes that use it — zero repetition

---

## Slide 3 — AuditedAttribute: Automatic Change Tracking

**One descriptor, all metrics covered**

- `AuditedAttribute.__set__` captures old value, new value, and timestamp
- Appends a structured change event to a shared `changeLog` list
- Applied to `recordsProcessed`, `meanLatencyMs`, `errorCount` in `MonitoredPipelineStage`
- Every assignment — from inside a method or from external code — is automatically logged
- Adding a new monitored attribute takes one line; monitoring logic lives in zero places other than the descriptor

---

## Slide 4 — Industry Use of Descriptors

**Where descriptors power production systems**

- **Django models**: `MyModel.name = "value"` triggers validation and marks field as dirty — descriptor
- **SQLAlchemy**: `row.column` generates SQL when accessed — descriptor
- **Python's `property`**: a built-in descriptor factory
- **dataclasses with validators**: descriptors under the hood
- In our pipeline: every metric attribute change generates an audit event — automatically, without any method calls

---

## Slide 5 — Memory Overhead: Why Dictionaries Are Expensive at Scale

**50–80 bytes overhead per record × millions of records = GB of waste**

- Python dict: hash table + type pointer + ref count + allocator metadata = ~50–80 bytes overhead per instance
- 10,000 records/second × 80 bytes overhead = 800 KB/s of pure bookkeeping overhead
- Over one hour: ~2.8 GB of overhead memory that stores NO useful data
- The garbage collector cleans it up — but frequent GC causes pauses
- Solution: replace per-instance dicts with fixed-layout `__slots__` objects

---

## Slide 6 — `__slots__`: Fixed-Layout Instances

**What `__slots__` does at the Python interpreter level**

- `__slots__ = ("sensorId", "timestamp", "value", "unit")` → pre-declares all attributes
- Python allocates a compact fixed-size C struct per instance instead of a per-instance dict
- No `__dict__` attribute on instances → cannot add new attributes dynamically
- Result: instance size drops from ~150 bytes (dict overhead) to ~56 bytes (slot vector)
- Construction is also faster — no dict initialization needed

---

## Slide 7 — `__slots__` Trade-offs

**When to use and when NOT to**

| Use `__slots__` | Don't use `__slots__` |
|---|---|
| Internal record types | Public-facing extensible objects |
| Fixed-schema data objects | Classes where external code adds attributes |
| High-volume pipeline records | Base classes in inheritance hierarchies |
| Known attribute set at design time | Classes needing `__dict__` for serialization |

- Pipeline's internal `SensorRecord` type: perfect candidate for `__slots__`
- Source, Stage, Sink classes: keep regular `__dict__` — external code may extend them

---

## Slide 8 — Protocol Enforcement: The Current Problem

**AttributeError at runtime, deep in the call stack**

```
AttributeError: 'MySensor' object has no attribute 'stream'
  at pipeline/coordinator.py, line 47, in buildPipeline
```

- The error appears 47 frames deep; the bug is in the Source class definition
- Developer must trace back through the stack to find the missing method
- All data setup and context manager entry happened before the error
- Resources may be partially initialized when the error fires

---

## Slide 9 — Abstract Base Classes: Moving Errors to Instantiation

**`TypeError` at object creation, not deep in the call stack**

- Define `SourceABC(ABC)` with `@abstractmethod def stream(self):`
- Any class inheriting from `SourceABC` that doesn't implement `stream()` → `TypeError` on `MySensor()`
- Error fires at the point the object is created — clear message, correct location
- No class changes needed for existing implementations that already have the right methods
- `abc.register(ExistingClass)` to mark existing classes as implementing the ABC without modification

---

## Slide 10 — `@runtime_checkable` Protocol: isinstance Without Inheritance

**Duck typing with explicit verification**

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class SourceProtocol(Protocol):
    def stream(self): ...

isinstance(my_source, SourceProtocol)  # True if my_source has .stream()
```

- No inheritance required: any object with a `stream` attribute passes the check
- Pipeline coordinator calls `isinstance(source, SourceProtocol)` at registration time
- Fail immediately with a clear error if protocol is not met — before any data flows
- Works with `typing.get_type_hints` for static analysis tools

---

## Slide 11 — Combining All Three: Self-Auditing Pipeline Components

**The production analytics engine's component model**

- Metrics attributes → `AuditedAttribute` descriptors: automatic change log, validation at assignment
- Internal record type → `SlottedRecord` with `__slots__`: minimum memory footprint
- Component registration → `isinstance(component, SourceProtocol)`: clear errors at startup
- Together: the pipeline is self-monitoring, memory-efficient, and robustly typed
- All three are zero-cost abstractions at the call site — no performance penalty for the monitoring

---

## Slide 12 — Lecture 41 Key Principles

**Advanced internals as production requirements**

- Descriptors centralize attribute-level logic: validation, logging, auditing — one place for all
- `__slots__` reduces per-instance overhead by eliminating the per-instance `__dict__`
- ABCs and `@runtime_checkable` Protocols move protocol violations from runtime surprises to startup-time assertions
- These mechanisms do not add features — they make existing features correct, efficient, and debuggable
- Next: advanced internals Part 2 — metaclasses, the import system, and compiler hooks

---
