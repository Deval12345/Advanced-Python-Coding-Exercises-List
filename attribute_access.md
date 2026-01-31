# Attribute Access Control in Python  
(__getattr__ and __getattribute__)

This file explains **how Python resolves attribute access**, why Python allows
interception of attribute lookup, and how this mechanism solves **real-world
problems involving dynamic and lazy attributes**.

This topic comes:
- **after Descriptors** (behavior of known attributes)
- **before ABCs** (formalizing expected behavior)

(Reference: *Fluent Python*, Part IV – Chapter 19)

---

## 1. Why Attribute Access Control Exists (Problem First)

In many real systems, **not all attributes are known when a class is written**.

Examples:
- configuration keys loaded from files or environment variables
- feature flags added without code changes
- proxy objects wrapping remote services
- adapters over third-party or legacy APIs

### Core problem

> How can an object respond meaningfully  
> when an attribute is accessed **even if that attribute does not yet exist**?

Python’s answer is **attribute access interception**.

---

## 2. Python Attribute Lookup Order (Precise and Critical)

When Python evaluates:

```python
obj.name
