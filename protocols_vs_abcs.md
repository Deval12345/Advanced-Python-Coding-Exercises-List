# Protocols vs Abstract Base Classes (ABCs)
Static Guarantees vs Runtime Guarantees

This file explains **why Protocols exist even though ABCs already exist**,  
and why **ABCs are still not obsolete in a statically typed Python world**.

This topic builds directly on:
- Abstract Base Classes (runtime contracts)
- Duck typing and behavior-based design

(Reference: *Fluent Python*, typing module docs, PEP 544)

---

## 1. The Core Confusion (Problem First)

A common question in modern Python is:

> “If we have `typing.Protocol`, why do we still need ABCs?”

At first glance:
- Both describe *behavior*
- Both avoid concrete inheritance
- Both support duck typing

But they solve **different problems at different times**.

---

## 2. Two Different Worlds: Runtime vs Static Analysis

Python code lives in **two worlds**:

1. **Runtime execution**
2. **Static analysis (IDE, type checker, linters)**

The key insight:

> You cannot rely on static guarantees in a language that runs without a type checker.

---

## 3. Coding Problem #1 — Static Validation for Developers

### Problem statement

You are building a library:
- developers consume your API
- you want IDE autocomplete and type safety
- you do *not* want runtime overhead
- users may pass any compatible object

---

## 4. Solution — Protocols

```python
from typing import Protocol

class PaymentGateway(Protocol):
    def pay(self, amount: int) -> None:
        ...
