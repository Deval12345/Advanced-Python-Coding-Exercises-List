
# Memory Layout for Different Data Models

## Overview

Python objects are:

- reference-based
- dynamically typed
- dictionary-backed

This design enables flexibility but introduces aliasing, mutability,
and memory overhead.

This module explains consequences, not premature optimization.

You will learn:

- Reference vs value semantics
- Identity vs equality
- Mutability and shared state
- Copying semantics
- Truth-value testing
- Object memory overhead
- RAM-efficient design trade-offs

---

## Section 1 — Identity vs Equality

Python variables reference objects.

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)
print(a is b)
print(a is c)
```

Rule: check None with `is`, not `==`.

---

## Exercise 1

```python
x = 1000
y = 1000
z = x

print(x == y)
print(x is y)
print(x is z)
```

Explain identity vs equality.

---

## Section 2 — Mutability & Aliasing

Assignment binds names, it does not copy objects.

```python
list1 = [1, 2, 3]
list2 = list1
list2.append(4)

print(list1)
```

---

## Exercise 2

```python
data = {"scores": [10, 20]}
alias = data
alias["scores"].append(30)
print(data)
```

Why did both change?

---

## Section 3 — Shallow vs Deep Copy

```python
import copy

original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
deep = copy.deepcopy(original)

shallow[0].append(99)

print(original)
print(deep)
```

---

## Exercise 3

Experiment with nested copies.

---

## Section 4 — Truth-Value Testing

Python uses __bool__ and __len__.

```python
print(bool([]))
print(bool(0))
print(bool([1]))
```

Custom example:

```python
class AlwaysTrue:
    def __bool__(self):
        return True
```
---

## Exercise 4

When should explicit comparisons replace implicit truthiness?

---

## Integrated Task — Payment Builder

```python
import copy
import uuid

DEFAULT_CONFIG = {
    "currency": "INR",
    "notes": {},
    "metadata": {"source": "web"},
    "discount_code": None
}

MISSING = object()

def build_payment(amount, currency=MISSING, notes=MISSING, discount_code=MISSING):
    config = copy.deepcopy(DEFAULT_CONFIG)
    config["amount"] = amount
    config["metadata"]["request_id"] = str(uuid.uuid4())

    if currency is not MISSING:
        config["currency"] = currency

    if notes is not MISSING:
        config["notes"] = notes

    if discount_code is not MISSING:
        config["discount_code"] = discount_code

    return config
```

---

## Section 5 — Memory Overhead

Python objects carry significant RAM overhead.

Lists of objects grow quickly.

Arrays store raw numbers compactly.

---

## Integrated Memory Optimization Task

```python
import sys
import array
import random

seen_ids = set()
amounts = array.array("f")

for i in range(1_000_000):
    tx_id = random.randint(1, 800_000)
    amount = random.random() * 100

    if tx_id not in seen_ids:
        seen_ids.add(tx_id)
        amounts.append(amount)

print(len(seen_ids))
print(sys.getsizeof(seen_ids))
print(sys.getsizeof(amounts))
```

---

## Final Takeaways

Understanding Python’s object model prevents real bugs.

Focus on:

- identity
- mutability
- copying
- truth protocols
- memory trade-offs

