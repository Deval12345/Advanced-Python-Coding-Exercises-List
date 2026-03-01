# Slides — Lecture 17: Object Memory Layout & __slots__ — Lightweight Objects at Scale

---

## Slide 1
**Title:** From Attribute Access to Attribute Storage
- Previous lectures covered how Python finds attributes: descriptors, __getattr__, ABCs
- Today we ask a different question: where does Python store attributes in memory?
- At small scale the answer does not matter; at millions of instances it determines whether your system fits in RAM
- This lecture is where design discipline meets engineering discipline

---

## Slide 2
**Title:** Every Python Object Has a __dict__
- By default, every instance carries a Python dictionary called __dict__
- __dict__ stores all instance attributes as key-value pairs
- This gives Python its dynamic power: add any attribute to any object at runtime
- A Python dict costs at minimum 232 bytes on a 64-bit system, before storing any values
- For 10 million objects: 2.3 GB of overhead just for the dictionaries

---

## Slide 3
**Title:** Why __dict__ Overhead Matters at Scale
- Data pipelines, simulations, and trading systems routinely hold millions of small objects
- The cost is not theoretical: 232 bytes per instance times 10 million is 2.3 GB before any data
- Memory pressure causes cache misses, GC pressure, and OOM errors in production
- Python needed a mechanism to opt out of __dict__ for known-shape objects
- That mechanism is __slots__

---

## Slide 4
**Title:** __slots__ — Fixed-Layout Objects
- Declaring __slots__ tells Python: do not create __dict__; allocate a fixed array of named cells instead
- Each named slot is a fixed-offset storage cell within the object's memory layout
- Attribute access becomes a direct memory read, not a dictionary lookup
- Typical memory reduction: 2x to 4x compared to a dict-based equivalent
- Trade-off: dynamic attribute assignment is no longer allowed

---

## Slide 5
**Title:** __slots__ Syntax and Effect
- Declare __slots__ as a tuple of strings at class body level: __slots__ = ('x', 'y', 'z')
- The __init__ method looks identical; self.x = x still works
- Slotted instances have no __dict__: hasattr(obj, '__dict__') returns False
- Attempting to set an undeclared attribute raises AttributeError immediately
- Industry users: Pandas internal objects, NumPy descriptors, HFT order objects

---

## Slide 6
**Title:** Example 17.1 — Measuring __dict__ vs __slots__
- PointWithDict and PointWithSlots both store x, y, z
- Measure dict-based: sys.getsizeof(instance) + sys.getsizeof(instance.__dict__)
- Measure slotted: sys.getsizeof(instance) only — no __dict__ exists
- Typical result: 240+ bytes vs 56 bytes, a ratio of roughly 4x
- For 1 million objects: 240 MB vs 56 MB — a saving of ~184 MB

---

## Slide 7
**Title:** __slots__ in Inheritance — The Critical Rule
- Subclasses inherit parent slots automatically
- A subclass may add its own __slots__ to extend the layout with new cells
- If a subclass omits __slots__, Python adds __dict__ to that subclass's instances
- The __dict__ appears in addition to the inherited slots — most savings are lost
- Rule: every class in the inheritance chain must define __slots__ to preserve the benefit

---

## Slide 8
**Title:** Example 17.2 — Correct Multi-Level Slots
- BaseRecord defines __slots__ = ('recordId', 'timestamp')
- SensorRecord(BaseRecord) defines __slots__ = ('sensorId', 'value', 'unit')
- Instances have all five slots, no __dict__, and a compact memory layout
- Verify correctness: hasattr(record, '__dict__') returns False
- All attributes read and write normally; the difference is invisible to the caller

---

## Slide 9
**Title:** When to Use __slots__ and When Not To
- Use when: many instances (thousands+), attributes known at design time, memory/speed is a concern
- Do not use when: dynamic attributes needed, class rarely instantiated, prototyping
- Do not use when: libraries access __dict__ directly (some ORMs, serializers, test mocks)
- Middle ground: include '__dict__' in __slots__ to get both named slots and a dict
- Always measure before and after to confirm the savings are real for your use case

---

## Slide 10
**Title:** Memory Profiling with tracemalloc
- sys.getsizeof measures one object's outer shell only — it does not follow references
- tracemalloc traces all memory allocations made during a block of code
- Workflow: tracemalloc.start() → run code → tracemalloc.take_snapshot() → analyze
- snapshot.statistics('lineno') shows allocation by source line for precise attribution
- Standard practice in production Python services to catch memory regressions between releases

---

## Slide 11
**Title:** Example 17.3 — Bulk Allocation Comparison
- Create 100,000 RecordDict instances; measure total allocation with tracemalloc snapshot
- Create 100,000 RecordSlot instances; measure with a second snapshot
- Sum s.size for all statistics in each snapshot; compare in kilobytes
- Slotted version consistently allocates significantly less total memory
- This pattern — measure before and after — is the standard profiling workflow

---

## Slide 12
**Title:** Lecture 17 Summary
- Python objects use __dict__ by default: flexible but expensive at 232+ bytes per instance
- __slots__ replaces __dict__ with a fixed array of named storage cells: 2x to 4x smaller
- Every class in an inheritance chain must define __slots__ to get the full benefit
- Use slots for classes with many instances and known attributes; measure with tracemalloc
- Next lecture: concurrency — how Python programs handle multiple things at once

---
