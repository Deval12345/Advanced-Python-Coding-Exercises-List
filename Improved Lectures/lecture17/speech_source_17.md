# Speech Source — Lecture 17: Object Memory Layout & __slots__ — Lightweight Objects at Scale

---

## Transition from Previous Lecture

### CONCEPT 0.1

Welcome back.

Over the last several lectures, we have been building a complete picture of Python's attribute system. We have seen descriptors, which let you intercept attribute reads and writes with arbitrary logic. We have seen __getattr__ and __getattribute__, which let you handle missing attributes and control every access. We have worked with ABCs, which let you formalize behavioral contracts across class hierarchies.

But all of that work focused on how Python finds attributes. Today we step back and ask a different question. How does Python actually store attributes in memory? Where do they live? What does that cost? And what happens when you have not one object, not ten, but ten million? This is the lecture where Python development stops being about elegant APIs and starts being about real engineering discipline.

---

## Section 1 — The Default __dict__ Storage

### CONCEPT 1.1

Let us start with a fact that surprises many developers: every Python object, by default, carries a dictionary. Not a conceptual dictionary. An actual Python dict, with all of its overhead, stored inside every single instance you create.

This dictionary is called __dict__. It is what makes Python objects so flexible. When you write self.name equals something in __init__, Python is not writing to a fixed memory slot. It is inserting a key-value pair into that dictionary. And because it is a real dictionary, you can add new attributes to any object at runtime. You can write myObject.newAttribute equals "hello" and Python simply inserts a new key. This is powerful. It is also expensive.

A Python dictionary, on a 64-bit system, costs at minimum 232 bytes just to exist. That is before you store a single thing in it. And every instance of every class pays this cost. For a class with ten million instances, you are spending 2.3 gigabytes on dictionaries alone, before a single attribute value is stored. If your objects represent sensor readings, financial records, or simulation particles, you are wasting most of your memory on plumbing.

This is why __dict__ was eventually recognized as a bottleneck in large-scale Python systems. The Python language designers needed a mechanism to say: I know exactly what attributes this class will ever have. Let me opt out of the dictionary and use something cheaper.

### EXAMPLE 1.1

Let me show you the size difference concretely so you can see real numbers rather than just theory.

We import sys so we can measure object sizes. We define two classes that do exactly the same thing: store three floating-point coordinates named x, y, and z. The first class, PointWithDict, is an ordinary Python class. The second class, PointWithSlots, has one additional line at the top of the class body. The line reads: __slots__ equals, and then a tuple containing the strings 'x', 'y', and 'z'.

We then create one instance of each, passing the same values. To measure the dict-based point, we use sys.getsizeof on the instance itself and add the size of its __dict__. We need both because sys.getsizeof only measures the object's outer shell. The __dict__ is a separate allocation that getsizeof misses. For the slotted point, there is no __dict__, so we measure only the instance.

The output will typically show the dict-based point consuming somewhere between 232 and 280 bytes, while the slotted point sits around 56 bytes. The ratio will be roughly four to one. Four times the memory for the same logical data.

```python
# Example 17.1
import sys                                             # line 1

class PointWithDict:                                   # line 2
    def __init__(self, x, y, z):                       # line 3
        self.x = x                                     # line 4
        self.y = y                                     # line 5
        self.z = z                                     # line 6

class PointWithSlots:                                  # line 7
    __slots__ = ('x', 'y', 'z')                        # line 8

    def __init__(self, x, y, z):                       # line 9
        self.x = x                                     # line 10
        self.y = y                                     # line 11
        self.z = z                                     # line 12

dictPoint = PointWithDict(1.0, 2.0, 3.0)               # line 13
slotPoint = PointWithSlots(1.0, 2.0, 3.0)              # line 14

dictSize = sys.getsizeof(dictPoint) + sys.getsizeof(dictPoint.__dict__)  # line 15
slotSize = sys.getsizeof(slotPoint)                    # line 16

print(f"Dict-based point: {dictSize} bytes")           # line 17
print(f"Slots-based point: {slotSize} bytes")          # line 18
print(f"Memory ratio: {dictSize / slotSize:.1f}x")    # line 19
```

---

## Section 2 — __slots__: Fixed-Layout Objects

### CONCEPT 1.2

So what does __slots__ actually do? It tells the Python interpreter: do not create __dict__ for instances of this class. Instead, allocate a fixed-length array of storage cells when each instance is created. One cell per named slot.

This is conceptually similar to how a C struct works. A C struct is a fixed-size block of memory with named fields at known offsets. Python's slot mechanism gives you something similar at the Python level. When you write self.x in a slotted class, Python does not do a dictionary lookup. It goes directly to the fixed offset for x within the object's memory layout. This is both faster and smaller.

The trade-off is the loss of flexibility. You can no longer add arbitrary attributes to a slotted instance at runtime. Try it, and Python raises an AttributeError. The attributes that exist are the ones you declared in __slots__, and that set is fixed forever at class definition time.

This is a deliberate engineering choice. When you declare __slots__, you are saying: I know the shape of this object. I am committing to it. Give me the performance in return. This is why high-performance Python libraries use __slots__ heavily. Pandas uses slotted internal structures. NumPy uses slotted descriptor objects. High-frequency trading systems use slotted order and quote objects that are created and destroyed millions of times per second.

---

## Section 3 — __slots__ in Inheritance

### CONCEPT 2.1

Slots become more complex when inheritance enters the picture, and this is where most developers who know about slots still get it wrong.

When you subclass a slotted class, the subclass inherits the parent's slots. The subclass can also define its own __slots__, which adds new slots without duplicating the parent's. But here is the critical rule: if the subclass does not define __slots__, Python automatically creates a __dict__ for the subclass instances. Not for the parent's slots, but as an additional dict bolted onto the object. This brings back most of the memory overhead you were trying to avoid.

Why does Python do this? Because the default Python behavior is to always have __dict__. By omitting __slots__ in the subclass, you are falling back to that default. The parent's slots still exist, but the subclass adds a __dict__ on top of them.

The rule to remember is: every class in an inheritance chain that you want to benefit from slots must define __slots__. If any class in the chain omits __slots__, that class and all its descendants get __dict__ again.

### EXAMPLE 2.1

Let me show you a well-structured inheritance example with proper slots at every level.

We define BaseRecord with __slots__ containing 'recordId' and 'timestamp'. Then we define SensorRecord inheriting from BaseRecord, and it defines its own __slots__ with 'sensorId', 'value', and 'unit'.

When we create a SensorRecord instance, it has all five slots: two from the parent and three of its own. We measure its size, and we check whether it has __dict__. If we did everything right, hasattr on __dict__ returns False. There is no dictionary. The object is a tight, fixed-layout structure in memory.

We can still read and write all five attributes normally. The interface is identical to a regular class. The difference is invisible to the caller but measurable at scale.

```python
# Example 17.2
import sys                                             # line 1

class BaseRecord:                                      # line 2
    __slots__ = ('recordId', 'timestamp')              # line 3

    def __init__(self, recordId, timestamp):           # line 4
        self.recordId = recordId                       # line 5
        self.timestamp = timestamp                     # line 6

class SensorRecord(BaseRecord):                        # line 7
    __slots__ = ('sensorId', 'value', 'unit')          # line 8

    def __init__(self, recordId, timestamp, sensorId, value, unit):  # line 9
        super().__init__(recordId, timestamp)           # line 10
        self.sensorId = sensorId                       # line 11
        self.value = value                             # line 12
        self.unit = unit                               # line 13

record = SensorRecord("r001", 1700000000, "sensor_A", 23.5, "celsius")  # line 14
print(f"SensorRecord size: {sys.getsizeof(record)} bytes")  # line 15
print(f"Has __dict__: {hasattr(record, '__dict__')}")  # line 16
print(f"Value: {record.value} {record.unit}")          # line 17
```

---

## Section 4 — When to Use __slots__ and When Not To

### CONCEPT 3.1

__slots__ is not the right tool for every situation. Let me give you a clear framework for when to use it and when to leave it out.

Use __slots__ when you have many instances — thousands to millions — and the attributes are known at design time. Use it when memory or attribute access speed is a concern. The canonical examples are records in a data pipeline, particles in a physics simulation, orders in a trading system, nodes in a tree with millions of elements.

Do not use __slots__ when you need dynamic attribute assignment. Configuration objects that pick up arbitrary settings at runtime cannot be slotted without losing that flexibility. Test mocks that set arbitrary attributes on the fly will break. Frameworks that use __dict__ directly, like some serialization and ORM libraries, will not work without additional configuration.

Do not use __slots__ when the class is rarely instantiated. If you create ten instances total, the memory saving is negligible. Use it when you are paying a real cost that is measurable in your profiling.

There is also a middle ground. If you include '__dict__' as a string inside your __slots__ tuple, Python gives you both: named slots for the declared attributes, plus a __dict__ for anything dynamic. The named slots still get the fast fixed-offset access. The __dict__ catches everything else. This is a deliberate escape hatch for cases where you need both performance and flexibility.

---

## Section 5 — Memory Profiling with tracemalloc

### CONCEPT 4.1

Measuring memory usage correctly in Python is harder than it looks. sys.getsizeof shows you the size of one object. But an object is just a shell. The strings it references, the lists it points to, the other objects it holds — those are all separate allocations that getsizeof does not include.

The right tool for measuring total memory allocation across a block of code is tracemalloc. The tracemalloc module is part of Python's standard library. It traces every memory allocation made by the Python interpreter. You call tracemalloc.start, run your code, call tracemalloc.take_snapshot, and then analyze the snapshot to see exactly how much memory was allocated and where.

In industry, this is how teams find memory regressions. A data pipeline that used to run in 4 GB now needs 6 GB after someone added a new field to the record class. Tracemalloc tells you exactly which line caused the allocation spike. It is an essential tool in any Python performance investigation.

### EXAMPLE 3.1

Let me show you tracemalloc in action, comparing the total allocation of 100,000 dict-based records against 100,000 slotted records.

We define both classes with the same three fields: rid, name, and value. We set COUNT to 100,000. Then we run the dict version inside a tracemalloc measurement, take a snapshot, and stop. Then we run the slot version, take another snapshot, and stop. We sum up the total bytes across all statistics in each snapshot and print the results in kilobytes.

The numbers will typically show the dict version allocating several times more memory than the slot version for the same 100,000 records. The exact ratio depends on the platform and Python version, but the difference is always substantial.

```python
# Example 17.3
import tracemalloc                                     # line 1

class RecordDict:                                      # line 2
    def __init__(self, rid, name, value):              # line 3
        self.rid = rid                                 # line 4
        self.name = name                               # line 5
        self.value = value                             # line 6

class RecordSlot:                                      # line 7
    __slots__ = ('rid', 'name', 'value')               # line 8
    def __init__(self, rid, name, value):              # line 9
        self.rid = rid                                 # line 10
        self.name = name                               # line 11
        self.value = value                             # line 12

COUNT = 100_000                                        # line 13

tracemalloc.start()                                    # line 14
dictRecords = [RecordDict(i, f"item_{i}", i * 1.5) for i in range(COUNT)]  # line 15
snapshot1 = tracemalloc.take_snapshot()                # line 16
tracemalloc.stop()                                     # line 17

tracemalloc.start()                                    # line 18
slotRecords = [RecordSlot(i, f"item_{i}", i * 1.5) for i in range(COUNT)]  # line 19
snapshot2 = tracemalloc.take_snapshot()                # line 20
tracemalloc.stop()                                     # line 21

stats1 = snapshot1.statistics('lineno')                # line 22
stats2 = snapshot2.statistics('lineno')                # line 23
print(f"Dict records total: {sum(s.size for s in stats1) / 1024:.1f} KB")  # line 24
print(f"Slot records total: {sum(s.size for s in stats2) / 1024:.1f} KB")  # line 25
```

---

## Closing

### CONCEPT 5.1

Let us bring this together.

Python objects use __dict__ by default. That gives you runtime flexibility at the cost of memory overhead: at minimum 232 bytes per instance just for the dictionary, before any attribute values. When you have millions of objects, that overhead becomes the dominant cost in your system's memory footprint.

__slots__ replaces __dict__ with a fixed-layout array of named storage cells. It reduces memory usage by a factor of two to four, and it makes attribute access faster by eliminating the dictionary lookup. The cost is the loss of dynamic attribute assignment. Attributes must be declared at class definition time.

In inheritance, the rule is strict: every class in the hierarchy must define __slots__. One class missing __slots__ brings __dict__ back for that entire branch.

Use __slots__ when you have many instances, known attributes, and a real performance constraint. Measure with tracemalloc to confirm the savings before and after. Do not use slots for prototyping, configuration objects, or classes instantiated only a handful of times.

In the next lecture, we open the concurrency chapter. We stop talking about how individual objects are built and start talking about how programs handle multiple things at once. The first question — always — is what kind of work you are doing. And that is where we begin.

---
