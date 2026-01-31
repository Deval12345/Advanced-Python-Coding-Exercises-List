

---

## 📄 `descriptors.md`

```markdown
# Descriptors

## Topic
Descriptor protocol (`__get__`, `__set__`, `__delete__`) for controlled attribute access.  
(Fluent Python – Chapter 20)

---

## Motivation
Some attributes must look like normal fields but execute logic on access (computed values, validation, ORM fields).

---

## Problem Scenario
Create a computed attribute that:
- Is not stored
- Is always correct
- Is read-only
- Uses attribute syntax

---

## Code Example

```python
class ExpiryDate:
    def __get__(self, obj, objtype=None):
        return obj.start + obj.duration

class Subscription:
    expiry_date = ExpiryDate()
    def __init__(self, start, duration):
        self.start = start
        self.duration = duration
