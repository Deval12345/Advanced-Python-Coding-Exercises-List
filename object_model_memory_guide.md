
# Object References, Mutability & Memory in Python

## Overview

This module explains how Python’s object model affects correctness, bugs, and memory usage.

You will learn:

- Reference semantics vs value semantics
- Identity vs equality
- Mutability and aliasing
- Shallow vs deep copying
- Truth-value testing
- Shared state pitfalls
- Memory overhead of Python objects
- Designing RAM-efficient systems

Every section includes:

✔ Concepts  
✔ Real-life scenarios  
✔ Exercises  
✔ Full runnable Python code  

---

## Section 1 — Identity vs Equality

Python variables hold references to objects — not raw values.

Two objects can:

✔ Have the same value  
✔ Be different objects in memory  

### Example

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True (value equality)
print(a is b)  # False (different objects)
print(a is c)  # True (same object)
```

### Checking for None (critical rule)

Always use:

```python
if x is None:
    ...
```

Never:

```python
if x == None:  # wrong
```

Because `==` may be overridden.

---

## Exercise 1 — Identity Exploration

```python
x = 1000
y = 1000
z = x

print(x == y)
print(x is y)
print(x is z)
print(id(x), id(y), id(z))
```

Explain:

- Which variables share identity?
- Which only share value?
- Why None must use `is`?

---

## Section 2 — Mutability & Aliasing

Assignment does not copy objects.

It binds multiple names to the same object.

### Example

```python
list1 = [1, 2, 3]
list2 = list1

list2.append(4)

print(list1)
print(list2)
```

Both change because they reference the same list.

This is aliasing.

---

## Exercise 2 — Mutation Through Alias

```python
data = {"scores": [10, 20]}
alias = data

alias["scores"].append(30)

print(data)
```

Question:

Why did modifying `alias` change `data`?

---

## Section 3 — Shallow vs Deep Copy

Shallow copy duplicates outer container only.

Deep copy duplicates everything recursively.

### Example

```python
import copy

original = [[1, 2], [3, 4]]

shallow = copy.copy(original)
deep = copy.deepcopy(original)

shallow[0].append(99)

print("Original:", original)
print("Shallow:", shallow)
print("Deep:", deep)
```

Observation:

Inner objects are shared in shallow copy.

---

## Exercise 3 — Nested Copy Experiment

Modify inner lists and observe:

- Which structure changes?
- Why deep copy prevents leakage?

---

## Section 4 — Truth-Value Testing

Python uses:

- `__bool__`
- `__len__`

to determine truthiness.

### Example

```python
print(bool([]))
print(bool(0))
print(bool(""))
print(bool([1]))
```

Custom truth behavior:

```python
class AlwaysTrue:
    def __bool__(self):
        return True

print(bool(AlwaysTrue()))
```

Explicit comparisons can be clearer than relying on truthiness in critical systems.

---

## Exercise 4 — Truth Testing

Check behavior of:

- Empty containers
- Zero values
- Custom objects

When should explicit checks be used instead of implicit truthiness?

---

## Integrated Task — Payment Request Builder

### Scenario

Shared default configuration:

```python
DEFAULT_CONFIG = {
    "currency": "INR",
    "notes": {},
    "metadata": {"source": "web"},
    "discount_code": None
}
```

Goal:

Build requests without mutating shared defaults.

### Implementation

```python
import copy
import uuid

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


p1 = build_payment(500)
p2 = build_payment(700, notes={})

print(p1)
print(p2)
print(DEFAULT_CONFIG)  # unchanged
```

### Lessons

✔ Identity-based sentinel  
✔ Avoid mutating shared defaults  
✔ Deep copy for nested safety  
✔ Distinguish missing vs empty values  

---

## Section 5 — Saving Memory

Python objects carry overhead.

Lists of Python integers consume large RAM.

Arrays store raw numbers compactly.

---

## Integrated Memory Optimization Task

### Scenario

Process 1,000,000 transactions.

Requirements:

- Ignore duplicate IDs
- Store only unique transactions
- Measure memory usage
- Optimize storage

### Implementation

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

print("Unique transactions:", len(seen_ids))
print("Approx memory (IDs set):", sys.getsizeof(seen_ids))
print("Approx memory (amount array):", sys.getsizeof(amounts))
```

### Observation

- Sets eliminate duplicates
- Arrays reduce numeric memory
- Design is a trade-off between RAM and behavior

This mirrors real streaming/payment systems.

---

## Final Takeaways

Python’s object model is about:

✔ References, not values  
✔ Mutability awareness  
✔ Copy semantics  
✔ Truth protocols  
✔ Memory trade-offs  

Understanding this prevents real production bugs.

---

## Suggested Extensions

1. Implement copy-on-write container
2. Create immutable configuration object
3. Compare list vs array RAM usage experimentally
4. Build a memory profiler for custom objects

---

End of module.
