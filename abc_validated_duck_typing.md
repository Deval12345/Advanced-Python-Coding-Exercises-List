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
```

---

# Abstract Base Classes (ABCs) and Validated Duck Typing

Abstract Base Classes (ABCs) formalize **expected behavior** in Python without forcing concrete inheritance hierarchies.
They are Python’s answer to the question:

> “How do we *validate* duck typing without abandoning flexibility?”

This topic builds on:
- attribute access & descriptors (how objects behave)
- before memory layout and performance (cost of abstractions)

(Reference: *Fluent Python*, Part IV – Chapters 11 & 12)

---

## 1. The Problem ABCs Exist to Solve

Python encourages **duck typing**:

> “If it quacks like a duck, treat it as a duck.”

This works well — until it doesn’t.

### Failure mode of pure duck typing

- Errors appear **deep at runtime**
- Failures happen **far from the source**
- Debugging becomes expensive
- Frameworks cannot validate extensions early

### Core problem

> How can we accept *any object* that behaves correctly,  
> but **reject incorrect ones early and explicitly**?

Python’s answer: **Abstract Base Classes (ABCs)**.

---

## 2. What an ABC Really Is (Important Clarification)

An ABC is:
- a **runtime contract**
- enforced when objects are instantiated or checked
- focused on **behavior**, not shared implementation

ABCs are **still inheritance**, but used *only for validation*, not reuse.

This is why Ramalho calls them:

> **“Validated duck typing.”**

---

## 3. Coding Problem #1 — Plugin / Extension Validation

### Problem statement

You are designing a system that accepts **pluggable components**:
- payment gateways
- storage backends
- notification providers

Requirements:
- each plugin must implement a specific set of methods
- incorrect plugins should fail **immediately**
- no shared implementation is required

Example usage:

```python
gateway.pay(100)
```
