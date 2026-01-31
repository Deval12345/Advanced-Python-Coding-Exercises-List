# Object Memory Layout in Python  
(`__dict__`, `__slots__`, and Lightweight Objects)

This file explains **how Python stores object attributes**, why object memory overhead can dominate performance, and how `__slots__` allows you to trade flexibility for efficiency.

This topic comes **after ABCs** (contracts and behavior) and **before concurrency** (scaling work),
because memory layout determines how well objects scale.

(Reference: *Fluent Python*, Object References & Mutability; *High Performance Python*, Chapter on memory)

---

## 1. Why Memory Layout Matters (Problem First)

Python objects are **powerful but heavy**.

Every instance normally carries:
- an attribute dictionary (`__dict__`)
- bookkeeping overhead
- indirection costs

This flexibility is great — until you create **many objects**.

### Core problem

> How can we reduce memory usage and improve performance  
> when creating large numbers of similar objects?

---

## 2. Default Attribute Storage (`__dict__`)

By default, instance attributes are stored in a dictionary.

### Baseline example

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
