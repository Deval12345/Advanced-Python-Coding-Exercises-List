# Slides — Lecture 14: Descriptors Part 2 — Lazy Loading, Computed Properties, and Framework Patterns

---

## Slide 1
**Title:** From Data Descriptors to Non-Data Descriptors

- L13 covered data descriptors: __get__ + __set__ wins over instance __dict__
- A non-data descriptor defines only __get__ — no __set__
- Instance __dict__ takes priority over a non-data descriptor
- This priority reversal is the exact mechanism that enables lazy loading
- Data descriptors enforce; non-data descriptors yield to cached results

---

## Slide 2
**Title:** The Lazy Loading Problem

- Expensive attributes: parsing large files, database queries, loading ML models
- Computing in __init__: pays full cost every time an object is created, even if never used
- Computing in a property: re-runs the full computation on every single access
- Lazy loading: compute exactly once on first access, cache, return cached on all later accesses
- The interface stays unchanged — callers see a plain attribute, not a method call

---

## Slide 3
**Title:** How Non-Data Descriptors Enable Lazy Loading

- First access: instance __dict__ has no entry, Python falls to the non-data descriptor, calls __get__
- Inside __get__: compute the value, write it into instance.__dict__[attributeName]
- Second access: Python checks instance __dict__ first, finds the cached value, returns it
- __get__ is never called again — no flags, no if-already-computed checks required
- The lookup priority mechanism of Python handles all caching logic structurally

---

## Slide 4
**Title:** Example 14.1 — LazyProperty Descriptor

- LazyProperty stores computeFunction and attributeName (set via __set_name__)
- __get__ calls computeFunction(instance), writes result to instance.__dict__, returns value
- On second access, instance __dict__ answers before the descriptor is ever reached
- @LazyProperty on ReportData.processedData: "Computing..." prints once, never on second access
- This is the pattern behind functools.cached_property in the Python standard library

---

## Slide 5
**Title:** Computed Properties vs Cached Properties — When to Use Each

- Computed property (use @property): value depends on mutable state, must always be fresh
- Example: Rectangle.area — width and height can change, so area must recalculate every time
- Cached property (use functools.cached_property): value derived from state that does not change
- Example: Document.wordCount after loading from disk, DataSet.featureMatrix after parsing
- Choosing the wrong one causes either stale data bugs or unnecessary repeated computation

---

## Slide 6
**Title:** Example 14.2 — Computed vs Cached Property Side by Side

- recordCount uses @property: calls len(self.sourceData) every time, reflects live changes
- summary uses @functools.cached_property: "Computing summary..." prints only once
- After adding an item, recordCount updates immediately; summary stays cached
- functools.cached_property is a non-data descriptor — exactly LazyProperty from Example 14.1
- Standard library proof: Python's designers chose the non-data descriptor pattern intentionally

---

## Slide 7
**Title:** Descriptors as Class-Level APIs — The Framework Pattern

- MyModel.id through the class returns a column expression object for building SQL queries
- instance.id through an instance returns the actual stored integer value
- Same attribute name, completely different result depending on class vs instance access
- This is the instance-is-None check in __get__: return self for class access, return data for instances
- SQLAlchemy, Django ORM, dataclasses, and attrs all depend on exactly this pattern

---

## Slide 8
**Title:** Example 14.3 — ORM-Style TypedField Descriptor

- TypedField stores fieldType and default; __set_name__ stores the attribute name
- __get__ returns self when instance is None — enabling framework query building
- __get__ returns instance.__dict__ value (or default) when accessed through an instance
- __set__ enforces the fieldType with a precise TypeError if the type is wrong
- User class uses TypedField(str), TypedField(int), TypedField(float) — clean typed model fields

---

## Slide 9
**Title:** The Full Attribute Lookup Priority Chain

- Step 1: search class MRO for a data descriptor → call its __get__ if found
- Step 2: check instance.__dict__ → return value if found
- Step 3: search class MRO for a non-data descriptor → call its __get__ if found
- Step 4: call __getattr__ or raise AttributeError
- This chain explains all Python attribute behavior — no exceptions, no special cases

---

## Slide 10
**Title:** Why the Chain Explains Everything

- property (data descriptor) beats instance.__dict__ — that is why assigning to a property calls setter
- cached_property (non-data descriptor) loses to instance.__dict__ on the second access — by design
- Functions in a class are non-data descriptors — that is how method binding works
- The chain is not a rule to memorize; it is a mechanism to understand
- Once you understand it, every "surprising" attribute behavior becomes predictable

---

## Slide 11
**Title:** Descriptors in the Standard Library — What You Already Used

- @property: data descriptor — getter called on every read, setter on every write
- @functools.cached_property: non-data descriptor — computes once, stores in instance.__dict__
- @classmethod: descriptor — __get__ returns a bound method with the class as first argument
- @staticmethod: descriptor — __get__ returns the raw function with no binding
- Every Python developer uses descriptors — understanding them closes the gap between use and mastery

---

## Slide 12
**Title:** Summary — Lecture 14

- Non-data descriptors define only __get__ and lose to instance.__dict__
- This makes them the right tool for lazy loading: compute once, write to __dict__, step aside
- For mutable-state-derived values use @property; for stable-computed values use cached_property
- The instance-is-None pattern in __get__ enables framework class-level APIs
- Full priority chain: data descriptors → instance __dict__ → non-data descriptors → __getattr__
- Next lecture: attribute access control — __getattr__ and __getattribute__

---
