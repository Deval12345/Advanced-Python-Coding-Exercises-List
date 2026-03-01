# Key Points — Lecture 14: Descriptors Part 2 — Lazy Loading, Computed Properties, and Framework Patterns

---

- A **non-data descriptor** defines only `__get__` — no `__set__`. The instance's `__dict__` takes priority over it.
- The priority reversal between data and non-data descriptors is the mechanism behind **lazy loading**: the descriptor computes a value once, writes it to `instance.__dict__`, and on subsequent accesses the `__dict__` answers first.
- **Lazy loading** avoids both the upfront cost of computing in `__init__` and the repeated cost of computing in `@property`.
- Use **`@property`** (a data descriptor) when the value depends on mutable state and must always reflect current values.
- Use **`functools.cached_property`** (a non-data descriptor) when the value is derived from state that does not change after construction.
- The **instance-is-None check** in `__get__` is the load-bearing pattern for framework class-level APIs: returning `self` when `instance is None` gives frameworks access to the descriptor object itself for query building and introspection.
- SQLAlchemy, Django ORM, dataclasses, and attrs all use this pattern — the same attribute access returns a query-building descriptor through the class and returns the actual value through an instance.
- The **full attribute lookup priority chain**: (1) data descriptors, (2) instance `__dict__`, (3) non-data descriptors, (4) `__getattr__` or `AttributeError`.
- This chain explains all Python attribute behavior: `@property` beats `instance.__dict__`, `cached_property` loses to it after the first access, and functions are non-data descriptors explaining how method binding works.
- Python's own `@classmethod` and `@staticmethod` are descriptors — understanding the descriptor protocol means understanding how Python itself is implemented.

---
