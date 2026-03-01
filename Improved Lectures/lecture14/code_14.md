# Code — Lecture 14: Descriptors Part 2 — Lazy Loading, Computed Properties, and Framework Patterns

---

## Example 14.1 — LazyProperty Non-Data Descriptor

```python
# Example 14.1
class LazyProperty:                                    # line 1
    def __init__(self, computeFunction):               # line 2
        self.computeFunction = computeFunction         # line 3
        self.attributeName = None                      # line 4

    def __set_name__(self, ownerClass, name):          # line 5
        self.attributeName = name                      # line 6

    def __get__(self, instance, ownerClass):           # line 7
        if instance is None:                           # line 8
            return self                                # line 9
        value = self.computeFunction(instance)         # line 10
        instance.__dict__[self.attributeName] = value  # line 11
        return value                                   # line 12

class ReportData:                                      # line 13
    def __init__(self, rawRows):                       # line 14
        self.rawRows = rawRows                         # line 15

    @LazyProperty                                      # line 16
    def processedData(self):                           # line 17
        print("Computing processed data...")           # line 18
        return [row.strip().upper() for row in self.rawRows]  # line 19

report = ReportData(["  alice  ", "  bob  ", "  carol  "])  # line 20
print("Report created")                                # line 21
print(report.processedData)                            # line 22
print(report.processedData)                            # line 23
```

**Line-by-line explanation:**

- **Lines 1–4:** `LazyProperty` stores the function to call when the attribute is first accessed. `attributeName` will be filled in by `__set_name__`.
- **Lines 5–6:** `__set_name__` is called automatically when the class body is processed. Python passes the name of the class attribute this descriptor is assigned to. We store it for use in `__dict__` lookups.
- **Lines 7–9:** `__get__` is the only special method we define. Because there is no `__set__`, this is a non-data descriptor. When accessed through the class (`instance is None`), we return `self`.
- **Lines 10–12:** The core of lazy loading. We call the stored function with the instance, write the result directly into `instance.__dict__`, and return it. On the next access, Python finds the value in `__dict__` before reaching this descriptor.
- **Lines 13–15:** `ReportData` stores raw row strings.
- **Lines 16–19:** `@LazyProperty` applies the descriptor. The `processedData` function prints a message, strips and uppercases each row, and returns the result list.
- **Line 20:** Creating the report does not compute `processedData`. Computation is deferred.
- **Line 21:** Prints "Report created" — confirms no computation happened at construction time.
- **Line 22:** First access triggers `LazyProperty.__get__`. "Computing processed data..." prints once. Result is stored in `report.__dict__['processedData']`.
- **Line 23:** Second access finds `processedData` in `report.__dict__` directly. `__get__` is never called again. "Computing..." does not appear a second time.

---

## Example 14.2 — Computed Property vs Cached Property

```python
# Example 14.2
import functools                                       # line 1
import time                                            # line 2

class DataPipeline:                                    # line 3
    def __init__(self, sourceData):                    # line 4
        self.sourceData = sourceData                   # line 5
        self._processingCount = 0                      # line 6

    @property                                          # line 7
    def recordCount(self):                             # line 8
        return len(self.sourceData)                    # line 9

    @functools.cached_property                         # line 10
    def summary(self):                                 # line 11
        self._processingCount += 1                     # line 12
        print(f"Computing summary (call #{self._processingCount})")  # line 13
        time.sleep(0.01)                               # line 14
        return {"count": len(self.sourceData), "total": sum(self.sourceData)}  # line 15

pipeline = DataPipeline([10, 20, 30, 40, 50])         # line 16
print(f"Count: {pipeline.recordCount}")               # line 17
pipeline.sourceData.append(60)                         # line 18
print(f"Count after add: {pipeline.recordCount}")     # line 19
print(pipeline.summary)                                # line 20
print(pipeline.summary)                                # line 21
```

**Line-by-line explanation:**

- **Lines 1–2:** Import `functools` for `cached_property` and `time` for simulating work.
- **Lines 3–6:** `DataPipeline` holds a mutable list of source data and a processing counter.
- **Lines 7–9:** `@property` makes `recordCount` a data descriptor. Every access calls `len(self.sourceData)` freshly. If `sourceData` changes, `recordCount` reflects the change immediately.
- **Lines 10–15:** `@functools.cached_property` makes `summary` a non-data descriptor. The body runs once, prints its processing count, sleeps to simulate work, and returns a dictionary.
- **Line 16:** Create a pipeline with five integers.
- **Line 17:** `recordCount` prints 5.
- **Line 18:** Append 60 to `sourceData`, mutating the list.
- **Line 19:** `recordCount` prints 6 — it recalculated because it is a live computed property.
- **Line 20:** First access to `summary`. "Computing summary (call #1)" prints. The dictionary is returned and cached.
- **Line 21:** Second access to `summary`. No print, no sleep — the cached dict from `pipeline.__dict__['summary']` is returned instantly.

---

## Example 14.3 — ORM-Style TypedField Descriptor

```python
# Example 14.3
class TypedField:                                      # line 1
    def __init__(self, fieldType, default=None):       # line 2
        self.fieldType = fieldType                     # line 3
        self.default = default                         # line 4
        self.attributeName = None                      # line 5

    def __set_name__(self, ownerClass, name):          # line 6
        self.attributeName = name                      # line 7

    def __get__(self, instance, ownerClass):           # line 8
        if instance is None:                           # line 9
            return self                                # line 10
        return instance.__dict__.get(self.attributeName, self.default)  # line 11

    def __set__(self, instance, value):                # line 12
        if not isinstance(value, self.fieldType):      # line 13
            raise TypeError(                           # line 14
                f"{self.attributeName} expects {self.fieldType.__name__}, "  # line 15
                f"got {type(value).__name__}"          # line 16
            )
        instance.__dict__[self.attributeName] = value  # line 17

class User:                                            # line 18
    name = TypedField(str, default="unknown")          # line 19
    age = TypedField(int, default=0)                   # line 20
    score = TypedField(float, default=0.0)             # line 21

    def __init__(self, name, age, score):              # line 22
        self.name = name                               # line 23
        self.age = age                                 # line 24
        self.score = score                             # line 25

user = User("alice", 30, 98.5)                         # line 26
print(f"{user.name}, age {user.age}, score {user.score}")  # line 27
print(type(User.name))                                 # line 28
```

**Line-by-line explanation:**

- **Lines 1–5:** `TypedField` is a data descriptor (it defines both `__get__` and `__set__`). It stores the expected type and a default value.
- **Lines 6–7:** `__set_name__` records the attribute name at class creation.
- **Lines 8–11:** `__get__` returns `self` when `instance is None` — this is the class-level API pattern. Frameworks access `User.name` to get the descriptor object and call methods on it for query building. When accessed through an instance, returns the stored value or the default.
- **Lines 12–17:** `__set__` enforces the type. If the value is not an instance of `fieldType`, it raises `TypeError` with a precise message naming the attribute and the types involved. Valid values are stored in `instance.__dict__`.
- **Lines 18–21:** `User` defines three typed fields as class attributes. The class body runs at import time; `__set_name__` fires for each field, setting their names to `'name'`, `'age'`, and `'score'`.
- **Lines 22–25:** `__init__` assigns all three attributes. Each assignment triggers `TypedField.__set__`, which validates the type before storing.
- **Line 26:** Create a valid user. All three type checks pass.
- **Line 27:** Print the user's fields — reads go through `TypedField.__get__`, which retrieves from `instance.__dict__`.
- **Line 28:** Access `User.name` through the class. Returns the `TypedField` descriptor object itself — instance is `None` so `__get__` returns `self`. This demonstrates the framework pattern: ORM query builders call methods on this descriptor object to construct SQL.

---
