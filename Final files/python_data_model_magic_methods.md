
# Python Data Model (Magic / Dunder Methods)

## Overview

Python’s data model is not an add-on feature — it is the core API of the language.
This guide explains how user-defined objects plug directly into Python syntax
through special (dunder) methods.

Key themes:
- Python as a framework
- Special methods as syntax hooks
- Behavior-driven design
- Duck typing
- Operator overloading
- Attribute interception

---

## Section 1 — Python as a Framework

Python is designed as a framework for objects.

Built-ins like:
len(x), x[i], for x in obj, x in obj

work because objects implement:
__len__, __getitem__, __iter__, __contains__

Python avoids adding new syntax by exposing protocols any object can implement.

### Example

import numpy as np
a = np.array([1, 2, 3])
print(len(a))
print(a[1])

---

## Exercise 1 — Build a Native-Like Container

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

---

## Section 2 — Special Methods as Syntax Glue

x + y  →  x.__add__(y)
x[i]   →  x.__getitem__(i)
len(x) →  x.__len__()

Example:

class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

---

## Section 3 — Duck Typing

Behavior matters more than type.
Objects integrate through capability, not inheritance.

Mini Project:

def analyze(source):
    lines = list(source)
    return {
        "line_count": len(lines),
        "word_count": sum(len(line.split()) for line in lines),
        "first_line": lines[0] if lines else None
    }

---

## Section 4 — Lazy Protocol Objects

class LazyFileLoader:
    def __init__(self, filename):
        self.filename = filename

    def __getitem__(self, index):
        with open(self.filename) as f:
            for i, line in enumerate(f):
                if i == index:
                    return line.strip()
        raise IndexError

---

## Section 5 — Strategy Plug-in Pattern

class Discount10:
    def apply(self, price):
        return price * 0.9

class Discount20:
    def apply(self, price):
        return price * 0.8

def checkout(price, strategy):
    return strategy.apply(price)

---

## Section 6 — Attribute Access Control

__getattr__
__getattribute__

Used for:
- dynamic attributes
- proxies
- lazy loading

---

## Section 7 — Descriptors

Descriptor protocol:

__get__
__set__
__delete__

Descriptors power @property, ORM fields, validation systems.

---

## Final Takeaways

Special methods are the API of Python itself.
They allow objects to integrate directly into syntax.

