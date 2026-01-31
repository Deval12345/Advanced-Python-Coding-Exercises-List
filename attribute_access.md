# Attribute Access Control in Python  
(`__getattr__` and `__getattribute__`)

This file explains **how Python resolves attribute access**, why Python allows interception of attribute lookup, and how this mechanism is used to solve **real coding problems** involving dynamic and lazy attributes.

This topic builds directly on **descriptors** and is foundational for:
- configuration systems
- proxies and adapters
- ORMs and frameworks
- lazy-loading patterns
- understanding Python’s object model

(Reference: *Fluent Python*, Part IV – Chapter 19)

---

## 1. Why Attribute Access Control Exists (Problem First)

In real programs, **not all attributes are known when a class is written**.

Common real-world situations:
- configuration keys loaded from files or environment variables
- feature flags added without code changes
- proxy objects wrapping remote services
- objects adapting legacy or third-party APIs

### The core problem

> How can an object respond meaningfully when an attribute is accessed **even if that attribute does not yet exist**?

Python’s answer is **attribute access interception**.

---

## 2. Python Attribute Lookup Order (Exact and Important)

When Python evaluates:

```python
obj.name
