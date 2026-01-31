# Design Trade-offs in Python
Duck Typing, Validated Duck Typing, and Static Typing

This file ties together the **design philosophy** behind Python’s flexibility,
showing how different mechanisms trade **freedom, safety, and performance**.

It is intended to be read **after**:
- attribute access & descriptors
- ABCs and Protocols
- memory layout & object overhead
- concurrency basics

(Reference: *Fluent Python*, typing PEPs, real-world framework design)

---

## 1. The Core Design Tension (Problem First)

Every non-trivial Python system must answer:

> How much flexibility can we allow  
> **without sacrificing correctness, maintainability, or performance**?

Python does not enforce one answer.
Instead, it provides **multiple mechanisms**, each with different trade-offs.

---

## 2. Duck Typing (Maximum Flexibility)

### Idea

“If it behaves correctly, accept it.”

### Example

```python
class FileLogger:
    def write(self, msg):
        print(msg)

def log(writer):
    writer.write("hello")

log(FileLogger())
