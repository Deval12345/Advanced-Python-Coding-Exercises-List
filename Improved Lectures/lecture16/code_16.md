# Code — Lecture 16: Abstract Base Classes (ABCs) & collections.abc — Validated Duck Typing

---

## Example 16.1 — DataProcessor ABC with Template Method Pattern

```python
# 1  from abc import ABC, abstractmethod
# 2
# 3  class DataProcessor(ABC):
# 4      @abstractmethod
# 5      def load(self, source):
# 6          pass
# 7
# 8      @abstractmethod
# 9      def process(self, data):
# 10         pass
# 11
# 12     @abstractmethod
# 13     def save(self, result, destination):
# 14         pass
# 15
# 16     def runPipeline(self, source, destination):
# 17         data = self.load(source)
# 18         result = self.process(data)
# 19         self.save(result, destination)
# 20         return result
# 21
# 22 class CsvProcessor(DataProcessor):
# 23     def load(self, source):
# 24         print(f"Loading CSV from {source}")
# 25         return [1, 2, 3, 4, 5]
# 26
# 27     def process(self, data):
# 28         return [x * 2 for x in data]
# 29
# 30     def save(self, result, destination):
# 31         print(f"Saving {result} to {destination}")
# 32
# 33 processor = CsvProcessor()
# 34 processor.runPipeline("data.csv", "output.csv")

from abc import ABC, abstractmethod

class DataProcessor(ABC):
    @abstractmethod
    def load(self, source):
        pass

    @abstractmethod
    def process(self, data):
        pass

    @abstractmethod
    def save(self, result, destination):
        pass

    def runPipeline(self, source, destination):
        data = self.load(source)
        result = self.process(data)
        self.save(result, destination)
        return result

class CsvProcessor(DataProcessor):
    def load(self, source):
        print(f"Loading CSV from {source}")
        return [1, 2, 3, 4, 5]

    def process(self, data):
        return [x * 2 for x in data]

    def save(self, result, destination):
        print(f"Saving {result} to {destination}")

processor = CsvProcessor()
processor.runPipeline("data.csv", "output.csv")
```

**Line-by-line explanation:**
- **Line 1:** Imports `ABC` (the base class for all ABCs) and `abstractmethod` (the decorator that marks a method as required) from the `abc` module.
- **Lines 3-4:** `DataProcessor` inherits from `ABC`, making it an abstract base class. It will never be instantiated directly.
- **Lines 4-6:** `@abstractmethod` on `load` declares that any concrete subclass must provide a `load` implementation. The `pass` body is a placeholder — the decorator is what matters.
- **Lines 8-10:** Same pattern for `process` — abstract, must be overridden.
- **Lines 12-14:** Same pattern for `save` — abstract, must be overridden.
- **Lines 16-20:** `runPipeline` is a concrete method — no `@abstractmethod` decorator. It defines the algorithm: load, then process, then save, in that order. All subclasses inherit this for free. This is the template method pattern; the structure is locked in the base class.
- **Lines 22-31:** `CsvProcessor` inherits from `DataProcessor` and provides concrete implementations of all three abstract methods. Because all three are implemented, instantiation is allowed.
- **Line 23-25:** `load` simulates reading a CSV file by returning a hardcoded list.
- **Lines 27-28:** `process` doubles every element using a list comprehension.
- **Lines 30-31:** `save` prints the result and destination, simulating a write operation.
- **Line 33:** `CsvProcessor()` succeeds because all abstract methods are implemented. If any were missing, Python would raise `TypeError: Can't instantiate abstract class CsvProcessor with abstract methods ...` right here.
- **Line 34:** `runPipeline` is inherited from `DataProcessor` — `CsvProcessor` did not define it. Python calls `self.load`, `self.process`, and `self.save` on the `CsvProcessor` instance, dispatching to the concrete implementations.

---

## Example 16.2 — FrozenConfig with collections.abc.Mapping

```python
# 1  from collections.abc import Mapping
# 2
# 3  class FrozenConfig(Mapping):
# 4      def __init__(self, data):
# 5          self._data = dict(data)
# 6
# 7      def __getitem__(self, key):
# 8          return self._data[key]
# 9
# 10     def __iter__(self):
# 11         return iter(self._data)
# 12
# 13     def __len__(self):
# 14         return len(self._data)
# 15
# 16 config = FrozenConfig({"host": "localhost", "port": 5432, "name": "mydb"})
# 17 print(config["host"])
# 18 print(len(config))
# 19 print(list(config.keys()))
# 20 print(isinstance(config, Mapping))

from collections.abc import Mapping

class FrozenConfig(Mapping):
    def __init__(self, data):
        self._data = dict(data)

    def __getitem__(self, key):
        return self._data[key]

    def __iter__(self):
        return iter(self._data)

    def __len__(self):
        return len(self._data)

config = FrozenConfig({"host": "localhost", "port": 5432, "name": "mydb"})
print(config["host"])
print(len(config))
print(list(config.keys()))
print(isinstance(config, Mapping))
```

**Line-by-line explanation:**
- **Line 1:** Imports `Mapping` from `collections.abc`. This is the ABC that represents any read-only key-value container.
- **Line 3:** `FrozenConfig` inherits from `Mapping`. This declares that `FrozenConfig` is a mapping type and commits to implementing its required interface.
- **Lines 4-5:** `__init__` takes any mapping-compatible `data` argument and stores a plain `dict` copy of it internally as `self._data`. Using `dict(data)` ensures we own the data and it cannot be mutated through the original reference.
- **Lines 7-8:** `__getitem__` is the first of the three required abstract methods. It delegates directly to the internal `dict`. This powers `config["host"]` syntax and is also used internally by the mixin methods.
- **Lines 10-11:** `__iter__` is the second required method. It returns an iterator over the keys of `self._data`. The `Mapping` mixin methods use this to implement `keys()`, `values()`, and `items()`.
- **Lines 13-14:** `__len__` is the third and final required method. It returns the number of key-value pairs. This three-method contract is all `Mapping` requires.
- **Line 16:** Instantiation succeeds because all three abstract methods are implemented. The config is immutable — there are no setter methods defined and `Mapping` does not require any.
- **Line 17:** `config["host"]` works via `__getitem__`. Output: `localhost`.
- **Line 18:** `len(config)` works via `__len__`. Output: `3`.
- **Line 19:** `config.keys()` is a free mixin method provided by `Mapping` — `FrozenConfig` never defined it. The ABC derives it from `__iter__`. Output: a view of all keys.
- **Line 20:** `isinstance(config, Mapping)` returns `True` because `FrozenConfig` inherits from `Mapping`. Any library function that accepts a `Mapping` will accept a `FrozenConfig`.

---

## Example 16.3 — Virtual Subclassing with ABC.register()

```python
# 1  from abc import ABC, abstractmethod
# 2
# 3  class Serializable(ABC):
# 4      @abstractmethod
# 5      def serialize(self):
# 6          pass
# 7
# 8      @abstractmethod
# 9      def deserialize(self, data):
# 10         pass
# 11
# 12 class LegacyRecord:
# 13     def serialize(self):
# 14         return {"legacy": True}
# 15
# 16     def deserialize(self, data):
# 17         return data
# 18
# 19 Serializable.register(LegacyRecord)
# 20 record = LegacyRecord()
# 21 print(isinstance(record, Serializable))
# 22 print(issubclass(LegacyRecord, Serializable))

from abc import ABC, abstractmethod

class Serializable(ABC):
    @abstractmethod
    def serialize(self):
        pass

    @abstractmethod
    def deserialize(self, data):
        pass

class LegacyRecord:
    def serialize(self):
        return {"legacy": True}

    def deserialize(self, data):
        return data

Serializable.register(LegacyRecord)
record = LegacyRecord()
print(isinstance(record, Serializable))
print(issubclass(LegacyRecord, Serializable))
```

**Line-by-line explanation:**
- **Line 1:** Imports `ABC` and `abstractmethod` from `abc`.
- **Lines 3-10:** `Serializable` is an ABC with two abstract methods: `serialize` and `deserialize`. Any class that inherits from `Serializable` must implement both before it can be instantiated.
- **Lines 12-17:** `LegacyRecord` is a plain Python class — no inheritance from `Serializable` or anything else. It implements both `serialize` and `deserialize` because its original author needed those methods, but it predates the `Serializable` ABC entirely. We cannot or do not want to modify this class.
- **Lines 13-14:** `serialize` returns a dictionary. This satisfies the behavioral contract of the `Serializable` ABC even though Python does not know that yet.
- **Lines 16-17:** `deserialize` accepts data and returns it. Again, the right behavior, but no declared relationship to `Serializable`.
- **Line 19:** `Serializable.register(LegacyRecord)` is the key call. It tells Python's ABC machinery that `LegacyRecord` should be considered a virtual subclass of `Serializable`. No inheritance is added; no methods are moved. Only the MRO lookup tables for `isinstance` and `issubclass` are updated.
- **Line 20:** `LegacyRecord()` instantiation works normally. There is no ABC enforcement here because `LegacyRecord` does not inherit from `Serializable` — the register call declared compatibility without imposing the abstract method check.
- **Line 21:** `isinstance(record, Serializable)` returns `True` — confirmed by the registration.
- **Line 22:** `issubclass(LegacyRecord, Serializable)` also returns `True`. Critical caveat: if `LegacyRecord` were missing `serialize` or `deserialize`, both of these lines would still return `True`. `register()` does not verify the interface — it trusts the developer.

---
