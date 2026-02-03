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
