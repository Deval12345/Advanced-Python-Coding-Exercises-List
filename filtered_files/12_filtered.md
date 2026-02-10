
# Object Memory Layout (__slots__)

## Overview

Python objects default to maximum flexibility.

Every instance carries a dynamic attribute dictionary.

This flexibility has a memory cost.

Large systems hit memory ceilings before CPU limits.

__slots__ exists to let developers opt out of flexibility when needed.

This is not a micro-optimization.
It is an architectural trade-off.

---

## Section 1 — Default Attribute Storage (__dict__)

Normal Python objects store attributes in:

instance.__dict__

This is a per-instance hash table.

Each object carries:

- dictionary overhead
- pointer indirections
- GC pressure

Flexible, but expensive at scale.

---

## Section 2 — What __slots__ Changes

__slots__ removes the per-instance dictionary.

It replaces dynamic storage with fixed layout.

Attributes become indexed storage instead of hash lookups.

This reduces:

- memory footprint
- garbage collector load
- cache misses

But removes:

- dynamic attribute creation
- per-instance __dict__

It is a controlled trade-off.

---

## Exercise — Memory Comparison

Create many objects and measure memory impact.

### Normal class

```python
import tracemalloc

class ReadingDict:
    def __init__(self, t, v):
        self.timestamp = t
        self.value = v

tracemalloc.start()
data = [ReadingDict(i, i * 0.1) for i in range(1_000_000)]
current, peak = tracemalloc.get_traced_memory()
print("Dict peak:", peak)
tracemalloc.stop()
```

### __slots__ class

```python
class ReadingSlots:
    __slots__ = ("timestamp", "value")

    def __init__(self, t, v):
        self.timestamp = t
        self.value = v

tracemalloc.start()
data = [ReadingSlots(i, i * 0.1) for i in range(1_000_000)]
current, peak = tracemalloc.get_traced_memory()
print("Slots peak:", peak)
tracemalloc.stop()
```

Observe large memory reduction.

---

## When __slots__ Matters

Important when:

- millions of objects exist
- memory is the bottleneck
- objects are lightweight carriers
- ingestion pipelines
- simulation systems
- telemetry streams

Not needed for small programs.

---

## Final Takeaways

__slots__ is:

- a memory architecture decision
- a scalability tool
- a trade-off against flexibility
- a performance multiplier at scale

Python gives control back to the developer.

