# Key Points — Lecture 17: Object Memory Layout & __slots__ — Lightweight Objects at Scale

---

- Every Python object has a `__dict__` by default — an actual Python dictionary allocated inside every instance that costs at minimum 232 bytes before any attribute is stored.
- At ten million instances, the default `__dict__` storage alone costs over 2 GB of memory, before any attribute values are counted.
- **`__slots__`** tells the Python interpreter to skip `__dict__` and allocate a fixed-layout array of named storage cells instead — one cell per declared attribute.
- Memory savings with `__slots__` are typically **3–4x**: a three-field class shrinks from ~260 bytes to ~56 bytes per instance.
- Attribute access on slotted classes is **faster** — Python goes directly to a fixed memory offset, not through a dictionary hash lookup.
- The trade-off: slotted instances cannot have arbitrary new attributes added at runtime. Any undeclared attribute raises `AttributeError`.
- **Inheritance rule**: every class in the inheritance chain must define `__slots__`. One class that omits `__slots__` re-introduces `__dict__` for that class and all subclasses, negating the savings.
- Including `'__dict__'` as a string inside `__slots__` is an explicit escape hatch: named slots get fast fixed-offset access, and `__dict__` catches any additional dynamic attributes.
- Use `__slots__` when: you have thousands to millions of instances, attributes are known at design time, and profiling confirms memory or access-speed is a bottleneck.
- Do not use `__slots__` for: configuration objects that accumulate dynamic attributes, test mocks, prototyping, or classes instantiated only a handful of times.
- **`tracemalloc`** is the standard library tool for measuring total memory allocation across a block of code — essential for confirming `__slots__` savings in real workloads.
- Industries that use `__slots__` heavily: data processing pipelines, high-frequency trading (order/quote objects), physics simulations (particles), ML pipelines (feature records).

---
