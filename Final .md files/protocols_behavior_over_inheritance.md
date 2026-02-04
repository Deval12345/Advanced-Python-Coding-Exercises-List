
# Protocols in Python — Behavior over Inheritance

## Overview

Python emphasizes behavior over inheritance. Objects integrate into systems
based on what they can do, not what they inherit from.

This guide explores:

- Informal interfaces
- Duck typing
- Consenting adults philosophy
- Behavior-driven design
- Protocol-based extensibility

---

## Section 1 — Informal Interfaces

Python rarely requires explicit inheritance to define compatibility.

Instead of declaring interfaces formally, Python allows objects to participate
in systems by implementing expected behavior.

Example: anything iterable works in a for-loop.

```python
class Countdown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        n = self.start
        while n > 0:
            yield n
            n -= 1
```

This object behaves like a native iterable without inheriting from anything.

---

## Section 2 — Duck Typing

“If it walks like a duck and quacks like a duck…”

Python accepts objects based on behavior.

Functions depend on capabilities, not class ancestry.

```python
def total(values):
    result = 0
    for v in values:
        result += v
    return result

print(total([1, 2, 3]))
print(total((4, 5, 6]))
```

Both list and tuple work because they follow the iteration protocol.

---

## Section 3 — Consenting Adults Philosophy

Python assumes programmers know what they are doing.

Instead of strict enforcement, Python trusts developers to follow conventions.

This leads to:

- flexible APIs
- composable design
- minimal restrictions
- rapid experimentation

Private attributes are a convention, not a rule.

```python
class Account:
    def __init__(self):
        self._balance = 100  # protected by convention
```

Nothing prevents access — the design relies on discipline.

---

## Section 4 — Behavior-Driven Design

Systems should depend on behavior, not class trees.

Example: interchangeable strategies.

```python
class JsonSerializer:
    def serialize(self, data):
        import json
        return json.dumps(data)

class XmlSerializer:
    def serialize(self, data):
        return "<data>" + str(data) + "</data>"

def save(serializer, data):
    return serializer.serialize(data)

print(save(JsonSerializer(), {"x": 1}))
print(save(XmlSerializer(), {"x": 1}))
```

No inheritance required.
Only shared behavior matters.

---

## Section 5 — Protocol-Based Extensibility

Protocols describe expectations without enforcing inheritance.

Objects plug into frameworks by implementing methods.

Example: file-like protocol.

```python
class MemoryBuffer:
    def __init__(self):
        self.data = ""

    def write(self, text):
        self.data += text

buffer = MemoryBuffer()
buffer.write("Hello")
```

Anything with `.write()` works with code expecting a file-like object.

---

## Final Takeaways

Python favors:

- behavior over inheritance
- informal interfaces
- flexible composition
- trust in developers
- protocol-driven extensibility

Compatibility comes from capability, not class hierarchy.
