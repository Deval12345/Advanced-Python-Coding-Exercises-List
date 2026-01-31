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
