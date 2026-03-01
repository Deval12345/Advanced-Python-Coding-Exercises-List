# Slides — Lecture 16: Abstract Base Classes (ABCs) & collections.abc — Validated Duck Typing

---

## Slide 1
**Title:** The Problem Duck Typing Does Not Solve

- Duck typing is flexible but silent on missing implementations
- A plugin missing `cleanup()` passes import, init, and unit tests
- The error surfaces at runtime — deep in production
- We need a mechanism for early, explicit, named failure

---

## Slide 2
**Title:** What Abstract Base Classes Enforce

- ABCs raise `TypeError` at instantiation if any abstract method is missing
- The error message names every unimplemented method
- You catch the bug at object creation, not at first invocation
- This is the difference between debugging at startup vs. 8 hours into a production run

---

## Slide 3
**Title:** Defining an ABC

- Import `ABC` and `abstractmethod` from the `abc` module
- Subclass `ABC` in your base class
- Decorate each required method with `@abstractmethod`
- Any concrete subclass must implement all abstract methods before instantiation

---

## Slide 4
**Title:** Concrete Methods in ABCs — The Template Method Pattern

- ABCs can define concrete (non-abstract) methods with full implementations
- Subclasses inherit these for free
- The ABC defines the algorithm structure; subclasses fill in the steps
- Used by: scikit-learn `BaseEstimator`, Django `View`, Apache Beam pipelines

---

## Slide 5
**Title:** Example 16.1 — DataProcessor ABC

- `DataProcessor` defines `load`, `process`, `save` as abstract methods
- `runPipeline` is a concrete method that calls all three in order
- `CsvProcessor` implements all three — instantiation succeeds
- Missing any method → `TypeError` raised at `CsvProcessor()` call

---

## Slide 6
**Title:** ABCs vs. Protocols — Choosing the Right Tool

- **ABCs**: inheritance-based, enforced at runtime, support mixin methods
- **Protocols**: structural subtyping, checked by mypy/static tools, no inheritance needed
- Use ABCs when: you control the class hierarchy and want runtime guarantees
- Use Protocols when: you cannot modify classes or want static type checker support

---

## Slide 7
**Title:** collections.abc — ABCs for Built-in Container Types

- Standard library ships ABCs for all container interfaces
- Key ABCs: `Sequence`, `Mapping`, `Set`, `Iterator`, `Callable`, `MutableMapping`
- Inheriting from these gives free mixin methods derived from your implementations
- `isinstance(obj, Mapping)` works correctly for any class that inherits from it

---

## Slide 8
**Title:** The Mixin Method Deal — Mapping Example

- Implement only 3 methods: `__getitem__`, `__iter__`, `__len__`
- Receive for free: `get()`, `keys()`, `values()`, `items()`, `__contains__`, `__eq__`
- Python derives all free methods from your three implementations
- You declare the contract; the ABC fulfills the rest

---

## Slide 9
**Title:** Example 16.2 — FrozenConfig with collections.abc.Mapping

- `FrozenConfig` inherits from `Mapping`
- Implements `__getitem__`, `__iter__`, `__len__` — nothing else
- Gets `keys()`, `values()`, `items()`, `get()`, `__contains__` for free
- `isinstance(config, Mapping)` returns `True`

---

## Slide 10
**Title:** ABC.register() — Virtual Subclassing

- Declare that an existing class satisfies an ABC without modifying it
- `Serializable.register(LegacyRecord)` — no inheritance change needed
- `isinstance()` and `issubclass()` return `True` after registration
- Trade-off: `register()` does NOT verify the class implements required methods

---

## Slide 11
**Title:** Example 16.3 — Registering a Legacy Class

- `Serializable` ABC requires `serialize` and `deserialize`
- `LegacyRecord` implements both but predates the ABC — no inheritance
- `Serializable.register(LegacyRecord)` declares compatibility
- `isinstance(record, Serializable)` → `True`; use when you have inspected the class

---

## Slide 12
**Title:** Where ABCs Excel — Summary

- **Plugin systems**: enforce interface at startup, not in production
- **Framework APIs**: template method pattern locks in algorithm structure
- **Custom containers**: inherit from `collections.abc`, implement the minimum, get the rest free
- **Key rule**: ABCs enforce at instantiation — early, loud, and with a clear error message

---
