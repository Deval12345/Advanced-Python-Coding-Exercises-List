
---

## 📄 `slots_memory.md`

```markdown
# Attribute Storage and __slots__

## Topic
Instance `__dict__`, attribute storage, and memory optimization.  
(Fluent Python – Object References & Mutability)

---

## Motivation
Every Python object carries memory overhead due to its attribute dictionary.

---

## Problem Scenario
Creating millions of small objects causes memory blow-up and performance degradation.

---

## Code Example

```python
class Normal:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class Slotted:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x = x
        self.y = y

