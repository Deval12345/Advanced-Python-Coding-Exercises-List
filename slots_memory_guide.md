
# __slots__ — Controlling Python Object Memory Layout

## Overview

Python objects default to maximum flexibility:

- dynamic attributes
- per-instance dictionaries
- runtime extensibility

This flexibility has a memory cost.

When millions of objects exist, memory overhead dominates performance.

`__slots__` exists to let developers trade flexibility for efficiency.

It is not a micro-optimization — it is architectural control.

You will learn:

✔ How instance attribute storage works  
✔ Why __dict__ consumes memory  
✔ What __slots__ changes internally  
✔ Memory vs flexibility trade-off  
✔ Performance impact at scale  
✔ When __slots__ is appropriate  

---

## Section 1 — How Python Stores Attributes

Normal Python objects store attributes in:

```
instance.__dict__
```

This is a hash table per object.

Each object carries:

✔ dictionary overhead  
✔ pointer indirections  
✔ GC pressure  

This is flexible but expensive.

---

## Section 2 — What __slots__ Does

`__slots__` removes the per-instance dictionary.

It replaces dynamic storage with a fixed layout.

Attributes become indexed storage, not hash lookups.

This reduces:

✔ memory footprint  
✔ garbage collector work  
✔ cache misses  

But removes:

✘ dynamic attribute creation  
✘ per-instance __dict__  

It is a trade-off.

---

## Exercise — Memory Comparison

Create many objects and measure impact.

### Case 1 — Normal class (__dict__)

```python
import time
import tracemalloc

N = 1_000_000

class ReadingDict:
    def __init__(self, t, v):
        self.timestamp = t
        self.value = v


tracemalloc.start()
start_time = time.perf_counter()

data = [ReadingDict(i, i * 0.1) for i in range(N)]
total = sum(r.value for r in data)

end_time = time.perf_counter()
current, peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

print("DICT CLASS")
print("Time:", round(end_time - start_time, 3), "seconds")
print("Peak memory:", round(peak / (1024 * 1024), 2), "MB")
```

---

### Case 2 — __slots__ class

```python
import time
import tracemalloc

N = 1_000_000

class ReadingSlots:
    __slots__ = ("timestamp", "value")

    def __init__(self, t, v):
        self.timestamp = t
        self.value = v


tracemalloc.start()
start_time = time.perf_counter()

data = [ReadingSlots(i, i * 0.1) for i in range(N)]
total = sum(r.value for r in data)

end_time = time.perf_counter()
current, peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

print("SLOTS CLASS")
print("Time:", round(end_time - start_time, 3), "seconds")
print("Peak memory:", round(peak / (1024 * 1024), 2), "MB")
```

---

## Expected Outcomes

✔ Significant memory reduction  
✔ Faster iteration  
✔ Lower GC overhead  
✔ Same semantics  
✔ Better cache locality  

Performance difference grows with object count.

---

## When __slots__ Matters

Important when:

✔ Millions of objects exist  
✔ Memory is the bottleneck  
✔ Objects are simple data carriers  
✔ High-throughput systems  
✔ Simulation engines  
✔ Messaging pipelines  

Not needed for small object counts.

---

## Real-World Impact

Systems affected:

✔ High-volume DTO models  
✔ Telemetry streams  
✔ Sensor simulations  
✔ Financial tick data  
✔ Message brokers  
✔ Data ingestion services  

At scale, memory layout affects runtime speed.

---

## Trade-Off Summary

| Feature | Normal class | __slots__ |
|--------|-------------|----------|
| Dynamic attributes | ✔ | ✘ |
| Memory per object | High | Low |
| Flexibility | Maximum | Restricted |
| GC pressure | Higher | Lower |
| Speed at scale | Slower | Faster |

Python gives you the choice.

---

## Final Takeaways

`__slots__` is:

✔ A memory architecture tool  
✔ A scalability decision  
✔ A controlled loss of flexibility  
✔ A performance multiplier at scale  

Use it when object count dominates your system.

---

## Suggested Extensions

1. Measure GC pauses with millions of objects
2. Combine __slots__ with dataclasses
3. Build memory-optimized event stream
4. Benchmark attribute access speed
5. Simulate high-volume ingestion system

---

End of module.
