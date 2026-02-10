
# Descriptors

## Overview

Python needed a way to attach reusable behavior to attribute access,
not to methods.

Descriptors exist because validation, conversion, computed fields,
and lazy loading do not belong inside business logic.

They centralize attribute behavior so frameworks can control fields
consistently across many classes.

Descriptors are not syntax tricks — they are part of Python’s object model.

---

## Section 1 — Why Properties Are Not Enough

@property works per class.

Descriptors work across classes.

If validation is written inside setters:

- logic gets duplicated
- enforcement becomes inconsistent
- maintenance becomes fragile

Descriptors solve reuse at the attribute level.

---

## Section 2 — Descriptor Protocol

A descriptor is an object implementing:

__get__(self, obj, owner)  
__set__(self, obj, value)  
__delete__(self, obj)

Python calls these automatically during attribute access.

Descriptors live on the class but operate on instances.

This separation enables reusable field behavior.

---

## Exercise 1 — Reusable Validated Field

Reusable validation without duplication.

```python
class ValidatedField:
    def __init__(self, min_value, max_value):
        self.min = min_value
        self.max = max_value
        self.name = None

    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, owner):
        return getattr(obj, self.name)

    def __set__(self, obj, value):
        if not (self.min <= value <= self.max):
            raise ValueError("Out of range")
        setattr(obj, self.name, value)


class Product:
    price = ValidatedField(0, 10000)


p = Product()
p.price = 500
print(p.price)
```

Validation logic reused across classes.

No duplication.

---

## Section 3 — Lazy Loading Descriptor

Framework-style attribute control.

```python
class DBField:
    def __get__(self, obj, owner):
        if obj is None:
            return self

        if "value" not in obj._cache:
            print("Loading from DB...")
            obj._cache["value"] = f"Data for {obj.id}"

        return obj._cache["value"]


class User:
    data = DBField()

    def __init__(self, id):
        self.id = id
        self._cache = {}


u = User(10)
print(u.data)
print(u.data)
```

Lazy DB access with caching.

---

## Section 4 — Computed Field Descriptor

Derived attribute tied to instance state.

```python
from datetime import date, timedelta

class ExpiryDate:
    def __get__(self, obj, owner):
        if obj is None:
            return self
        return obj.start_date + timedelta(days=obj.duration)


class Subscription:
    expiry_date = ExpiryDate()

    def __init__(self, start_date, duration):
        self.start_date = start_date
        self.duration = duration


s = Subscription(date(2025, 1, 1), 30)
print(s.expiry_date)
```

Attribute syntax preserved.
No method calls required.

---

## Real-World Uses

Descriptors power:

- ORM fields
- validation frameworks
- @property internals
- dataclass field control
- configuration systems

Many Python frameworks depend on descriptors.

---

## Final Takeaways

Descriptors:

- attach behavior to attributes
- enable reusable validation
- separate class logic from instance data
- remove duplication
- preserve clean APIs
- enable framework architecture

They are one of Python’s most powerful abstractions.
