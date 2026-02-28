# Code — Lecture 17: Object Memory Layout & __slots__ — Lightweight Objects at Scale

---

# Example 17.1 — Comparing __dict__ vs __slots__ Memory Usage

```python
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

**Expected output (values are platform-dependent):**
```
Dict-based point: 280 bytes
Slots-based point: 56 bytes
Memory ratio: 5.0x
```

**Line-by-line explanation:**
- Line 1: Import sys for memory measurement with sys.getsizeof.
- Lines 2-6: PointWithDict is a plain Python class. Every instance will automatically receive a __dict__ that stores x, y, and z as key-value pairs.
- Lines 7-12: PointWithSlots declares __slots__ as a tuple of attribute names. Python does not create __dict__ for these instances. Instead, it allocates three fixed-offset storage cells within the object.
- Line 13: Create a dict-based instance with three float values.
- Line 14: Create a slot-based instance with the same values.
- Line 15: Measure the dict-based instance correctly. sys.getsizeof of the instance alone does not count the separately-allocated __dict__. Both sizes must be added.
- Line 16: For the slotted instance, there is no __dict__. sys.getsizeof of the instance alone captures everything.
- Lines 17-19: Print both sizes and their ratio. The dict-based version is typically 4x to 5x larger.

---

# Example 17.2 — __slots__ in Inheritance with Correct Multi-Level Declaration

```python
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

**Expected output:**
```
SensorRecord size: 72 bytes
Has __dict__: False
Value: 23.5 celsius
```

**Line-by-line explanation:**
- Line 1: Import sys for size measurement.
- Lines 2-6: BaseRecord declares __slots__ with two attribute names. Instances of BaseRecord receive no __dict__, only the two named storage cells.
- Lines 7-13: SensorRecord inherits from BaseRecord and also declares __slots__ with its own three attribute names. This is the critical step. If this __slots__ declaration were omitted, SensorRecord instances would receive a __dict__ in addition to the inherited slots, eliminating most memory savings.
- Line 9-10: __init__ calls super().__init__ to populate the parent slots before setting its own.
- Line 14: Create a SensorRecord instance with all five attribute values.
- Line 15: sys.getsizeof captures the compact slotted layout. The number is small because there is no dictionary allocation.
- Line 16: hasattr on '__dict__' returns False. This confirms that no dictionary was created. The object uses only fixed-layout slots.
- Line 17: Normal attribute access works identically to a dict-based class. The difference is invisible to the caller.

---

# Example 17.3 — Bulk Instantiation Memory Comparison with tracemalloc

```python
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

**Expected output (approximate, platform-dependent):**
```
Dict records total: 28400.3 KB
Slot records total: 8200.7 KB
```

**Line-by-line explanation:**
- Line 1: Import tracemalloc from the standard library. This module traces memory allocations at the Python level.
- Lines 2-6: RecordDict is a plain class. Each instance allocates one object plus one __dict__.
- Lines 7-12: RecordSlot uses __slots__. Each instance allocates only the compact fixed-layout object.
- Line 13: Set the count to 100,000. This is large enough to produce a clear and measurable difference.
- Line 14: Start tracemalloc tracing. From this point, every allocation is recorded.
- Line 15: Create 100,000 dict-based records in a list comprehension. Both the instances and their dictionaries are allocated here.
- Line 16: Take a snapshot of all allocations made since tracemalloc.start.
- Line 17: Stop tracing to avoid measuring the slot-based pass.
- Line 18: Restart tracing for the second measurement.
- Line 19: Create 100,000 slotted records. Allocations are much smaller because no __dict__ is created.
- Line 20: Take a second snapshot for the slot version.
- Line 21: Stop tracing again.
- Lines 22-23: Retrieve statistics grouped by source line. Each statistic has a .size attribute giving total bytes for that line's allocations.
- Lines 24-25: Sum all .size values across all statistics in each snapshot and convert to kilobytes. The dict version consistently allocates three to four times more total memory.

---

# Example 17.4 — Demonstrating the Inheritance __slots__ Trap

```python
import sys                                             # line 1

class SlottedBase:                                     # line 2
    __slots__ = ('baseField',)                         # line 3

class SubclassMissingSlots(SlottedBase):               # line 4
    def __init__(self, baseField, extraField):         # line 5
        self.baseField = baseField                     # line 6
        self.extraField = extraField                   # line 7

class SubclassWithSlots(SlottedBase):                  # line 8
    __slots__ = ('extraField',)                        # line 9
    def __init__(self, baseField, extraField):         # line 10
        self.baseField = baseField                     # line 11
        self.extraField = extraField                   # line 12

bad = SubclassMissingSlots("base_val", "extra_val")    # line 13
good = SubclassWithSlots("base_val", "extra_val")      # line 14

print(f"Missing slots subclass has __dict__: {hasattr(bad, '__dict__')}")  # line 15
print(f"Proper slots subclass has __dict__: {hasattr(good, '__dict__')}")  # line 16
print(f"Missing slots size: {sys.getsizeof(bad) + sys.getsizeof(bad.__dict__)} bytes")  # line 17
print(f"Proper slots size: {sys.getsizeof(good)} bytes")  # line 18

bad.dynamicAttr = "this works because __dict__ exists"  # line 19
print(f"Dynamic attribute on bad: {bad.dynamicAttr}")   # line 20

try:                                                   # line 21
    good.dynamicAttr = "this will fail"               # line 22
except AttributeError as e:                            # line 23
    print(f"Blocked on good: {e}")                    # line 24
```

**Expected output:**
```
Missing slots subclass has __dict__: True
Proper slots subclass has __dict__: False
Missing slots size: 288 bytes
Proper slots size: 48 bytes
Dynamic attribute on bad: this works because __dict__ exists
Blocked on good: 'SubclassWithSlots' object has no attribute 'dynamicAttr'
```

**Line-by-line explanation:**
- Lines 2-3: SlottedBase defines one slot. Instances have no __dict__.
- Lines 4-7: SubclassMissingSlots inherits from SlottedBase but does not define __slots__. Python falls back to the default behavior and creates __dict__ for this subclass's instances. The parent's slot still exists, but a dictionary is added on top.
- Lines 8-12: SubclassWithSlots properly declares __slots__ at its own level, adding only the new attribute. No __dict__ is created.
- Line 13: Create the bad subclass instance — it will silently have __dict__.
- Line 14: Create the good subclass instance — no __dict__.
- Line 15: hasattr on '__dict__' returns True for the bad instance. The inheritance trap has triggered.
- Line 16: hasattr returns False for the good instance. Slots are working correctly.
- Lines 17-18: The size comparison shows that omitting __slots__ in the subclass restores most of the dictionary overhead.
- Line 19: Because bad has __dict__, dynamic attribute assignment works — the behavior you wanted to avoid.
- Lines 21-24: Trying to set an undeclared attribute on good raises AttributeError immediately, enforcing the fixed layout.
