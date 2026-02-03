
# Descriptors — Reusable Attribute Behavior in Python

## Overview

Descriptors are one of Python’s deepest mechanisms.

They allow behavior to be attached to **attributes**, not methods.

This enables:

- validation
- conversion
- computed fields
- lazy loading
- framework-level control

Descriptors exist so attribute behavior can be reused centrally instead of
being duplicated inside business logic.

Many major frameworks rely on this mechanism.

You will learn:

✔ Descriptor protocol (__get__, __set__, __delete__)  
✔ Why properties alone are insufficient  
✔ Reusable attribute validation  
✔ Lazy loading via descriptors  
✔ Computed fields tied to instance state  
✔ Framework-style attribute control  

---

## Section 1 — Why Descriptors Exist

Properties solve per-class attribute behavior.

Descriptors solve **reusable attribute behavior across classes**.

If validation is written inside methods:

- logic gets duplicated
- enforcement becomes inconsistent
- maintenance becomes fragile

Descriptors centralize behavior.

---

## Section 2 — Descriptor Protocol

A descriptor is an object implementing:

```
__get__(self, obj, owner)
__set__(self, obj, value)
__delete__(self, obj)
```

Python calls these automatically during attribute access.

Descriptors live on the class but operate on instances.

This separation is the key design.

---

## Exercise 1 — Reusable Validated Field

Goal: reusable validation without duplicating code.

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

Reusable validation across classes.

No duplication.

---

## Section 3 — ORM Lazy Loading Descriptor

Class-level field drives per-instance behavior.

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
print(u.data)  # cached
```

Outcome:

✔ Lazy DB access  
✔ Per-instance caching  
✔ Clean API  
✔ No constructor logic  

Framework-style behavior.

---

## Section 4 — Computed Field Descriptor

Derived attribute computed from instance state.

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

Outcome:

✔ Derived field  
✔ Always correct  
✔ Read-only  
✔ Attribute syntax preserved  

Descriptor reads instance state transparently.

---

## Real-World Uses

Descriptors power:

✔ ORM fields (Django / SQLAlchemy style)  
✔ Validation frameworks  
✔ @property internals  
✔ Dataclass field behavior  
✔ Framework configuration systems  

Many frameworks would not exist without descriptors.

---

## Final Takeaways

Descriptors:

✔ Attach behavior to attributes  
✔ Reuse validation and logic  
✔ Separate class ownership from instance state  
✔ Enable framework-level design  
✔ Remove duplication  
✔ Preserve clean APIs  

They are one of Python’s most powerful abstractions.

---

## Suggested Extensions

1. Build type-checking descriptor
2. Create read-only descriptor
3. Implement caching descriptor
4. Add conversion descriptor
5. Simulate ORM model fields

---

End of module.
