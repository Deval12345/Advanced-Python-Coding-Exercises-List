# Speech Source — Lecture 14: Descriptors Part 2 — Lazy Loading, Computed Properties, and Framework Patterns

---

## CONCEPT 0.1 — Transition from Previous Lecture

In the previous lecture, we saw how data descriptors implement reusable attribute validation by intercepting writes through __set__. A data descriptor wins over the instance's __dict__, so every read and every write goes through the descriptor's methods. Now we look at non-data descriptors and the patterns they enable. Non-data descriptors define only __get__ — no __set__. The instance's __dict__ takes priority over them. This priority reversal is not a limitation. It is the mechanism that makes lazy loading possible: a non-data descriptor's __get__ can compute a value, write it into the instance's __dict__, and then step out of the way forever. On the next access, the instance __dict__ answers first and __get__ is never called again.

---

## CONCEPT 1.1 — The Lazy Loading Problem

Some object attributes are expensive to compute: parsing a large file, executing a database query, loading a machine learning model, computing a graph traversal, or deserializing a complex data structure. If you compute these in __init__, you pay the cost upfront — every time the object is created — even when the expensive attribute is never accessed in that execution path.

If you compute them in a property on every access, you repeat the full computation cost every time the attribute is read. Properties recalculate unconditionally. For a word count that reads a 10MB document, that is 10MB of I/O on every access. For a machine learning model that parses and validates a 500MB weights file, that is half a gigabyte of work on every attribute read.

Lazy loading means: compute the value exactly once, the first time it is actually accessed, then cache it and return the cached value on all subsequent accesses. The interface to the caller does not change — it still looks like a regular attribute access. No method calls, no explicit caching logic in the calling code. The caching is structural, invisible to the caller. Descriptors enable this without modifying the class's interface.

---

## CONCEPT 1.2 — Non-Data Descriptors for Lazy Loading

A non-data descriptor defines only __get__, not __set__. Python's attribute lookup priority is: data descriptors first, then instance __dict__, then non-data descriptors. This means a non-data descriptor is shadowed by any value stored in the instance's __dict__ under the same key.

This priority order is the key to lazy loading. The first time the attribute is accessed, the instance __dict__ has no entry for it, so Python falls through to the non-data descriptor and calls its __get__. Inside __get__, we compute the expensive value, write it into instance.__dict__[self.attributeName] = value, and return it. The next time the attribute is accessed, Python checks the instance __dict__ first, finds the cached value, and returns it immediately — __get__ is never called again. The descriptor computed the value once and then stepped out of the way. This is elegant because it requires no flags, no if-already-computed checks, no explicit cache dictionaries. The lookup priority mechanism of Python handles all of it.

---

## EXAMPLE 1.1 — LazyProperty Descriptor

Here we define a class called LazyProperty that implements lazy loading as a non-data descriptor.

The constructor takes a computeFunction — the function to call when the attribute is first accessed. It stores the function and initializes attributeName to None.

__set_name__ is called at class creation and stores the attribute name.

__get__ checks whether instance is None and returns self for class-level access. Otherwise, it calls computeFunction(instance), stores the result in instance.__dict__ under the attribute name, and returns the result. The critical point is writing to instance.__dict__. On the next access, Python finds the value in __dict__ before reaching the descriptor.

We define ReportData with a rawRows list. We decorate processedData with @LazyProperty. This assigns a LazyProperty descriptor to the class attribute processedData. When report.processedData is first accessed, LazyProperty.__get__ runs: it prints "Computing processed data...", processes the rows, stores the result in report.__dict__['processedData'], and returns the list. The second time report.processedData is accessed, Python finds 'processedData' in report.__dict__ directly and returns it — __get__ is never called again.

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

---

## CONCEPT 2.1 — Computed Properties vs Cached Properties

Not all derived values should be cached. Some values must always reflect the current state of the object because the object's state changes. A Rectangle's area — width times height — depends on width and height, which can both change after construction. Caching area when width is 10 and height is 5 gives the wrong answer after the user changes the width to 20. For these values, you want a computed property: recalculate every time, always fresh.

Other values are derived from state that does not change: a Document's word count after loading from disk, or a pre-processed dataset's feature matrix. These are candidates for cached properties: compute once, cache the result.

Python 3.8 introduced functools.cached_property as part of the standard library. It implements exactly the LazyProperty pattern we just wrote: a non-data descriptor whose __get__ writes the result into instance.__dict__. On subsequent accesses, the instance __dict__ answers and __get__ is never called. Knowing the descriptor mechanism means you understand exactly why functools.cached_property works the way it does.

For mutable state, use Python's built-in property — a data descriptor that calls a getter function on every access. For immutable or one-time computed state, use functools.cached_property — a non-data descriptor that computes once and caches.

---

## EXAMPLE 2.1 — Computed Property vs Cached Property

Here we define a DataPipeline class that demonstrates the difference between a live computed property and a once-computed cached property.

recordCount uses @property, which is a data descriptor. Every call to pipeline.recordCount calls the function and returns len(self.sourceData). When we append a new item to sourceData, the next call immediately reflects the updated count.

summary uses @functools.cached_property. The function body prints a message showing which call number it is, sleeps briefly to simulate work, and returns a dictionary. We access pipeline.summary twice. The print message appears only once, proving the function body ran only once. The second access returns the cached result from instance.__dict__ without calling the function again.

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

---

## CONCEPT 3.1 — Descriptors as Class-Level APIs (Framework Patterns)

The most powerful use of descriptors — and the one used by every major Python framework — is building class-level APIs that behave differently depending on whether they are accessed through the class or through an instance.

When you access MyModel.id on a SQLAlchemy model, you get back a column expression object that knows how to build SQL predicates. You use it to write queries: session.query(MyModel).filter(MyModel.id == 5). When you access instance.id on a model instance, you get the actual integer value stored in that row. The exact same attribute access, MyModel.id and instance.id, produces completely different results depending on whether instance is a class or an object. This is the descriptor protocol: return self when instance is None, return the actual value when instance is an object.

Django fields work the same way. dataclasses field descriptors handle default factories. The descriptor protocol is what makes all of these APIs possible. When you see a class-level attribute that behaves like a query builder, it is a descriptor.

---

## EXAMPLE 3.1 — ORM-Style TypedField Descriptor

Here we define TypedField, a descriptor that behaves like a typed model field with a default value.

The constructor accepts a fieldType and a default. __set_name__ stores the attribute name. __get__ returns self when instance is None — this is the class-level access that frameworks use for query building. When instance is not None, it returns the value from the instance __dict__, falling back to the default. __set__ checks that the value is an instance of fieldType, raising TypeError with a precise message if not, then stores the valid value in the instance __dict__.

We define a User model class with name as a TypedField(str), age as a TypedField(int), and score as a TypedField(float), each with a sensible default. We create a user and print their fields. We also show that accessing User.name through the class returns the descriptor object itself — the same behavior that SQLAlchemy and Django use for query building.

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
```

---

## CONCEPT 4.1 — Descriptor Lookup Priority Chain

Python's attribute lookup follows a precise order that explains all attribute behavior. When you write instance.attribute, Python:

First, searches the class and its MRO for a data descriptor — any descriptor with __get__ and __set__ or __delete__. If found, it calls that descriptor's __get__. This is why property always wins over a same-named entry in instance.__dict__.

Second, if no data descriptor was found, checks instance.__dict__ for the attribute name. If found, returns the value directly. This is normal instance attribute access.

Third, if not in instance.__dict__, searches the class and its MRO for a non-data descriptor — any object with __get__ but no __set__. If found, calls its __get__.

Fourth, if nothing was found above, calls __getattr__ if the class defines it.

Understanding this chain explains every surprising attribute behavior in Python. Property beats instance __dict__ because property is a data descriptor. cached_property wins on the second access because it is a non-data descriptor and it stored the result in instance.__dict__, which now takes priority over it. Functions are non-data descriptors — that is why method lookup finds them in the class even when the instance has no matching entry in its __dict__.

---

## CONCEPT 5.1 — Final Takeaway Lecture 14

Descriptors are Python's complete attribute programming interface. The two lectures together cover the full protocol.

Data descriptors define __set__ and take priority over instance __dict__. They are the right tool for reusable validation: the value must pass through the descriptor on every write.

Non-data descriptors define only __get__ and are shadowed by instance __dict__. They are the right tool for lazy loading and caching: the descriptor computes the value once, writes it into instance.__dict__, and the instance dict handles all subsequent accesses.

The full lookup priority chain — data descriptors, then instance __dict__, then non-data descriptors — explains all Python attribute behavior. Python's built-in property is a data descriptor. functools.cached_property is a non-data descriptor. Functions in a class are non-data descriptors — that is how method binding works.

Framework APIs from SQLAlchemy to Django to dataclasses are built on the instance-is-None pattern in __get__: return self when accessed through the class, return the stored value when accessed through an instance. Descriptors are not an advanced feature you may never need. They are the core of Python's object model.

---
