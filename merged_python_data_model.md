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
```

---

# Descriptors — Programmable Attribute Access

Descriptors explain **how attributes themselves can have behavior**.
They are the mechanism behind `@property`, methods, ORM fields, and many
framework-level abstractions in Python.

This topic comes **after exception handling (EAFP)** and **before full attribute interception**,
because descriptors operate on *known attributes*, not unknown ones.

(Reference: *Fluent Python*, Part IV – Chapter 20)

---

## 1. Why Descriptors Exist (Problem First)

Many attributes in real programs are not simple stored values.

Common requirements:
- computed attributes (derived from other fields)
- validation on assignment
- read-only fields
- lazy loading from external systems (DB, cache, API)
- tracking when a field changes

### The core problem

> How can an attribute look like a normal field  
> **but execute logic every time it is accessed or assigned**?

Python’s answer is **descriptors**.

---

## 2. The Descriptor Protocol (Minimal and Precise)

A descriptor is any object that implements one or more of:

```python
__get__(self, obj, objtype=None)
__set__(self, obj, value)
__delete__(self, obj)
```
