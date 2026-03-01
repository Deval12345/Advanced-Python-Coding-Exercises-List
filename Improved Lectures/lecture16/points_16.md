# Key Points — Lecture 16: Abstract Base Classes (ABCs) & collections.abc — Validated Duck Typing

---

- **Early failure at instantiation**: An ABC raises `TypeError` the moment you try to instantiate a subclass that has not implemented all abstract methods — the error names every missing method, surfacing the bug at object creation rather than at the moment the missing method is first invoked in production.

- **Defining an ABC**: Subclass `abc.ABC` and decorate each required method with `@abstractmethod`; any concrete subclass that skips even one abstract method cannot be instantiated.

- **Concrete methods in ABCs**: An ABC can define fully implemented (non-abstract) methods that all subclasses inherit for free; this is the foundation of the template method pattern, where the algorithm structure lives in the base class and subclasses only fill in the steps.

- **Template method pattern**: Used by scikit-learn `BaseEstimator`, Django `View`, and Apache Beam; the base class defines the sequence of operations (e.g., `runPipeline` calling `load`, `process`, `save`) and subclasses supply the per-format implementations.

- **ABCs vs. Protocols**: ABCs use inheritance and enforce at runtime; Protocols use structural subtyping and are checked by static tools like mypy; choose ABCs for runtime guarantees and mixin methods, choose Protocols when you cannot modify classes or need static checker support.

- **collections.abc container ABCs**: The standard library ships ABCs for all built-in container interfaces — `Sequence`, `Mapping`, `MutableMapping`, `Set`, `Iterator`, `Callable` — each providing mixin methods derived from a small set of required implementations.

- **The mixin method deal**: Inheriting from `collections.abc.Mapping` and implementing only `__getitem__`, `__iter__`, and `__len__` gives you `get()`, `keys()`, `values()`, `items()`, `__contains__`, and `__eq__` for free; the ABC derives all of them from your three methods.

- **isinstance correctness**: Any class that inherits from a `collections.abc` ABC will pass `isinstance` checks against that ABC, enabling correct runtime type checking throughout the codebase without extra registration or monkey-patching.

- **ABC.register() — virtual subclassing**: Calling `MyABC.register(SomeClass)` makes `isinstance` and `issubclass` return `True` for that pair without requiring `SomeClass` to inherit from `MyABC`; essential for integrating third-party or legacy classes into an ABC hierarchy.

- **register() does not verify**: Virtual subclassing via `register()` declares compatibility but does not check that the registered class actually implements the required abstract methods; the responsibility for verification lies with the developer, not Python.

- **Primary use cases**: Plugin systems (enforce interface at startup), framework APIs (template method guarantees correct algorithm structure), and custom container types (inherit from `collections.abc`, implement the minimum, receive a complete interface).

---
