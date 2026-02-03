
# Python Data Model & Behavior-Driven Design

## Overview

This guide explains how Python’s data model enables user-defined objects to behave like native objects. 
You will learn:

- Special methods and operator overloading
- Python as a framework
- Behavior-driven design
- Duck typing and informal interfaces
- Protocol-based extensibility

Each section includes:

✔ Concept explanation  
✔ Real-life scenarios  
✔ Exercises  
✔ Full working code  

---

## Section 1 — Python as a Framework

Python is not just a language — it is a framework that lets objects plug into syntax itself.

Built-ins like:

- `len(x)`
- `x[i]`
- `for x in obj`
- `x in obj`

work because objects implement special methods such as:

- `__len__`
- `__getitem__`
- `__iter__`
- `__contains__`

This design keeps Python extensible without adding new syntax.

### Real-life impact

Libraries like NumPy and Pandas feel “native” because they integrate with Python’s data model.

Example:

```python
import numpy as np

a = np.array([1, 2, 3])
print(len(a))
print(a[1])
```

No special syntax was invented — they plug into Python’s framework.

---

## Exercise 1 — Build a Native-Like Container

### Goal

Create an object that behaves like a list using special methods.

### Requirements

Your object must support:

- len(x)
- indexing
- iteration
- membership testing

### Implementation

```python
class CustomList:
    def __init__(self, data):
        self._data = list(data)

    def __len__(self):
        return len(self._data)

    def __getitem__(self, index):
        return self._data[index]

    def __contains__(self, item):
        return item in self._data

    def __iter__(self):
        return iter(self._data)


nums = CustomList([1, 2, 3, 4])

print(len(nums))
print(nums[2])
print(3 in nums)

for n in nums:
    print(n)
```

### Observation

The object integrates seamlessly with Python syntax.

No custom API needed.

---

## Section 2 — Why Magic Methods Are Fast

Built-ins like `len(x)` are faster than `x.len()` because they use CPython’s internal C slots.

This avoids dynamic attribute lookup and enables direct protocol dispatch.

### Result

✔ Faster execution  
✔ Automatic interoperability  
✔ Cleaner syntax  

---

## Mini Project — Unified Text Analyzer

### Goal

Write one function that works with any text source.

### Requirements

- Works with files
- Works with in-memory buffers
- Works with custom generators
- No type checks
- No conditionals

### Implementation

```python
def analyze(source):
    lines = list(source)
    return {
        "line_count": len(lines),
        "word_count": sum(len(line.split()) for line in lines),
        "first_line": lines[0] if lines else None
    }


# File
with open("sample.txt") as f:
    print(analyze(f))

# In-memory buffer
from io import StringIO
buffer = StringIO("Hello world
Python is fun")
print(analyze(buffer))


# Custom generator object
class DynamicText:
    def __iter__(self):
        yield "Generated line one"
        yield "Generated line two"

print(analyze(DynamicText()))
```

### Key lesson

Behavior matters more than type.

Objects that follow iteration protocol integrate automatically.

---

## Section 3 — Duck Typing & Informal Interfaces

Python relies on **capabilities**, not ancestry.

If an object behaves correctly, it is accepted.

This is known as:

✔ Duck typing  
✔ Behavior-driven design  
✔ Informal interfaces  

> “If it walks like a duck and quacks like a duck…”

### Real-life uses

- Plugin systems
- Framework hooks
- Testing with mocks
- Dependency injection
- Strategy patterns

---

## Exercise 2 — Lazy Data Loader

### Problem

Access rows from a large file without loading everything into memory.

Must support:

```
data[i]
```

### Implementation

```python
class LazyFileLoader:
    def __init__(self, filename):
        self.filename = filename

    def __getitem__(self, index):
        with open(self.filename) as f:
            for i, line in enumerate(f):
                if i == index:
                    return line.strip()
        raise IndexError("Index out of range")


data = LazyFileLoader("transactions.txt")
print(data[5])
```

### Observation

The object behaves like a list without being a list.

This is protocol-driven design.

---

## Exercise 3 — Plug-in Strategy System

### Goal

Allow interchangeable strategies without inheritance.

### Implementation

```python
class Discount10:
    def apply(self, price):
        return price * 0.9


class Discount20:
    def apply(self, price):
        return price * 0.8


def checkout(price, strategy):
    return strategy.apply(price)


print(checkout(100, Discount10()))
print(checkout(100, Discount20()))
```

### Lesson

The function depends on behavior, not class hierarchy.

New strategies plug in without modifying existing code.

---

## Final Takeaways

Python’s power comes from:

✔ Protocols over inheritance  
✔ Behavior over declarations  
✔ Syntax extensibility  
✔ Minimal core language  
✔ Composable design  

Special methods are not magic.

They are the interface to Python itself.

---

## Suggested Practice Extensions

1. Build a custom matrix class supporting `+`, `*`, and indexing.
2. Create a file-like object that streams random data.
3. Implement a read-only container that forbids mutation.
4. Build a logging proxy object using `__getattr__`.

---

End of guide.
