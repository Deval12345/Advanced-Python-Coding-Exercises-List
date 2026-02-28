# Speech Source — Lecture 16: Abstract Base Classes (ABCs) & collections.abc — Validated Duck Typing

---

## CONCEPT 0.1 — Transition from Previous Lecture

We've seen that duck typing lets objects participate in protocols without inheriting from a specific class. We saw that descriptors let us validate attribute values. In the last lecture, __getattr__ and __getattribute__ gave us fine-grained control over how individual attribute accesses are handled. Now we tackle a different gap: how do you enforce that a class implements a complete interface — all required methods — and get a clear, early error if it doesn't? That's precisely what Abstract Base Classes do.

---

## CONCEPT 1.1 — The Late Failure Problem

Duck typing is powerful but silent on missing implementations. If you build a plugin system where all plugins must implement process(), validate(), and cleanup(), a plugin that forgets cleanup() will pass every import check, every initialization check, and every unit test that doesn't happen to call cleanup(). The error only surfaces at runtime — potentially deep in a production system, on a large dataset, after hours of processing. The cost of late failures is enormous: lost computation, debugging time, difficult-to-reproduce production incidents, and real customer impact.

Abstract Base Classes solve this with early failure: the moment you try to instantiate a class that inherits from an ABC but hasn't implemented all required abstract methods, Python raises a TypeError with an explicit message naming the missing methods. You find the bug at the earliest possible moment — object creation — not when the missing method is first invoked.

Industry impact: plugin systems in data processing pipelines, ML frameworks like scikit-learn, and web frameworks all use ABCs or ABC-like contracts to ensure that new contributors implement complete interfaces. The error message tells you exactly what is missing, making onboarding faster and more reliable.

---

## CONCEPT 1.2 — Defining ABCs with abc.ABC and @abstractmethod

You define an ABC by subclassing abc.ABC (which uses ABCMeta as its metaclass) and decorating each required method with @abstractmethod. Any concrete class that subclasses your ABC must implement every abstract method before it can be instantiated. This is a class-creation-time check: Python inspects the class when it is defined and raises TypeError when you call the class constructor if any abstract methods are unimplemented.

The ABC can also define concrete methods — non-abstract methods with a full implementation. Subclasses inherit these for free. This enables the template method pattern: the ABC defines the algorithm skeleton (calling abstract methods in order), and each subclass fills in the specific steps.

Industry: scikit-learn's BaseEstimator, Django's View, and data pipeline frameworks like Apache Beam all use this pattern. The framework author writes the orchestration logic once. Plugin authors implement only the domain-specific steps.

---

## EXAMPLE 1.1 — DataProcessor ABC

Narration: DataProcessor defines three abstract methods — load, process, and save — that any concrete subclass must implement. It also defines a concrete runPipeline method that calls all three in sequence. CsvProcessor provides implementations for all three. When we instantiate CsvProcessor(), Python checks that all three abstract methods are present — they are, so instantiation succeeds. Calling runPipeline("data.csv", "output.csv") runs load, then process, then save, printing the results. If CsvProcessor were missing any of the three, instantiation would raise TypeError immediately, before runPipeline is ever called.

```python
# Example 16.1
from abc import ABC, abstractmethod                    # line 1

class DataProcessor(ABC):                             # line 2
    @abstractmethod                                    # line 3
    def load(self, source):                            # line 4
        pass                                           # line 5

    @abstractmethod                                    # line 6
    def process(self, data):                           # line 7
        pass                                           # line 8

    @abstractmethod                                    # line 9
    def save(self, result, destination):               # line 10
        pass                                           # line 11

    def runPipeline(self, source, destination):        # line 12
        data = self.load(source)                       # line 13
        result = self.process(data)                    # line 14
        self.save(result, destination)                 # line 15
        return result                                  # line 16

class CsvProcessor(DataProcessor):                    # line 17
    def load(self, source):                            # line 18
        print(f"Loading CSV from {source}")            # line 19
        return [1, 2, 3, 4, 5]                         # line 20

    def process(self, data):                           # line 21
        return [x * 2 for x in data]                  # line 22

    def save(self, result, destination):               # line 23
        print(f"Saving {result} to {destination}")     # line 24

processor = CsvProcessor()                             # line 25
processor.runPipeline("data.csv", "output.csv")        # line 26
```

---

## CONCEPT 2.1 — ABCs vs Protocols: When to Use Each

Both ABCs and Protocols (from the typing module) define interfaces. The critical difference: ABCs use inheritance and enforce compliance through instantiation errors at runtime. Protocols use structural subtyping — any class with the right methods satisfies the protocol, with no inheritance required. Type checkers like mypy and pyright verify protocol conformance statically without running the code.

ABCs are the right tool when you want explicit inheritance as a signal ("this class is a DataProcessor"), runtime enforcement, and mixin methods that subclasses inherit. Protocols are the right tool when you want duck typing with type checker support, when you can't modify the classes you're checking, or when you're writing library code that should work with any object that happens to have the right shape.

In practice: framework APIs use ABCs; library utility code and type annotations often use Protocols. Python's own collections.abc module provides ABCs for all built-in container types — Sequence, Mapping, Set, Iterator, Callable, and more.

---

## EXAMPLE 2.1 — FrozenConfig Using collections.abc.Mapping

Narration: Mapping is an ABC from collections.abc that defines the interface for read-only mapping types. To implement it, you must provide __getitem__, __iter__, and __len__. In exchange, you get all the mixin methods for free: get(), keys(), values(), items(), __contains__, and __eq__. FrozenConfig provides the three required methods, each delegating to an internal dict. After instantiation, we can use all Mapping mixin methods without implementing them ourselves. isinstance(config, Mapping) returns True because FrozenConfig inherits from Mapping.

```python
# Example 16.2
from collections.abc import Mapping                   # line 1

class FrozenConfig(Mapping):                          # line 2
    def __init__(self, data):                         # line 3
        self._data = dict(data)                       # line 4

    def __getitem__(self, key):                       # line 5
        return self._data[key]                        # line 6

    def __iter__(self):                               # line 7
        return iter(self._data)                       # line 8

    def __len__(self):                                # line 9
        return len(self._data)                        # line 10

config = FrozenConfig({"host": "localhost", "port": 5432, "name": "mydb"})  # line 11
print(config["host"])                                  # line 12
print(len(config))                                     # line 13
print(list(config.keys()))                             # line 14
print(isinstance(config, Mapping))                     # line 15
```

---

## CONCEPT 3.1 — ABC Registration: isinstance Without Inheritance

ABCs support virtual subclassing via the .register() class method. You can declare that an existing class satisfies an ABC's interface without modifying that class or adding it to the class hierarchy. isinstance() and issubclass() return True after registration. This is essential for integrating third-party code: if a library provides a class with the right interface but doesn't inherit from your ABC, you can register it without forking the library. Python's standard library uses this internally: certain numpy types and other third-party types are registered with ABCs like Sequence even though they don't inherit from it.

The trade-off: .register() does not verify that the class actually implements the required methods. It is a declaration of intent, not a verification. If you register a class that's missing methods, isinstance returns True but calling those methods will still fail.

---

## EXAMPLE 3.1 — Virtual Subclassing with ABC.register()

Narration: Serializable is an ABC with two required abstract methods: serialize and deserialize. LegacyRecord is a class that already implements both methods but was written before Serializable existed — perhaps it's from a third-party library. We can't modify LegacyRecord to inherit from Serializable. Instead, we call Serializable.register(LegacyRecord). After registration, isinstance(record, Serializable) returns True, and issubclass(LegacyRecord, Serializable) returns True. The integration is complete without touching LegacyRecord.

```python
# Example 16.3
from abc import ABC, abstractmethod                   # line 1

class Serializable(ABC):                              # line 2
    @abstractmethod                                   # line 3
    def serialize(self):                              # line 4
        pass                                          # line 5

    @abstractmethod                                   # line 6
    def deserialize(self, data):                      # line 7
        pass                                          # line 8

class LegacyRecord:                                   # line 9
    def serialize(self):                              # line 10
        return {"legacy": True}                       # line 11

    def deserialize(self, data):                      # line 12
        return data                                   # line 13

Serializable.register(LegacyRecord)                   # line 14

record = LegacyRecord()                               # line 15
print(isinstance(record, Serializable))               # line 16
print(issubclass(LegacyRecord, Serializable))         # line 17
```

---

## CONCEPT 4.1 — Template Method Pattern with ABCs

ABCs naturally express the template method pattern. The ABC defines the algorithm structure — the sequence of steps and their order — by implementing a concrete method that calls abstract methods. Subclasses fill in the specific behavior of each step. The ABC author controls the orchestration; subclass authors implement only the domain-specific pieces.

Django's class-based views are the canonical Python example. Django's View ABC defines the HTTP dispatch loop: it handles routing, calls the appropriate method (get, post, put, delete) based on the HTTP method, catches exceptions, and formats the response. You subclass View and implement only the methods you need. Django handles everything else. This is a massive productivity win: you write tens of lines, not hundreds.

---

## CONCEPT 5.1 — Final Takeaway Lecture 16

ABCs enforce interface compliance at instantiation time, not at first use. This converts silent runtime failures — a plugin that forgets a method, detected hours into a production run — into immediate, informative TypeErrors at object creation. collections.abc provides ABCs for every Python container type: Sequence, Mapping, Set, Iterator, MutableMapping, MutableSequence, and more. Implementing a collections.abc ABC gives your custom container all mixin methods for free with only three to five required methods. ABC.register() adds virtual subclassing for third-party types without modifying them. ABCs excel at plugin systems, framework APIs, and template method patterns. In the next lecture, we examine object memory layout and __slots__ — controlling how Python stores object attributes to reduce memory overhead at scale.
