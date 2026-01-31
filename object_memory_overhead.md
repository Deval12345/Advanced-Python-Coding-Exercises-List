
---

## 📄 `object_memory_overhead.md`

```markdown
# When Object Overhead Dominates Computation

## Topic
Memory cost of Python objects and performance impact.  
(High Performance Python – Chapter 11)

---

## Motivation
Programs slow down not because of computation, but because of memory pressure and garbage collection.

---

## Problem Flow
Many objects → high memory usage → cache misses → GC pressure → execution-time slowdown.

---

## Code Illustration

```python
items = [{"x": i, "y": i*i} for i in range(1_000_000)]
total = sum(item["y"] for item in items)

