# Attribute Access Interception

## Topic
Intercepting attribute access using `__getattr__`, `__getattribute__`, attribute lookup order, and lazy attributes.  
(Fluent Python – Chapter 19)

---

## Motivation
Some attributes are not known at class-definition time (configuration keys, proxies, adapters). Python allows objects to dynamically respond when an attribute is requested.

---

## Attribute Lookup Order (Simplified)

When `obj.attr` is evaluated:

1. `obj.__getattribute__("attr")`
2. Instance `__dict__`
3. Class attributes and descriptors
4. Base classes
5. `__getattr__("attr")` (only if all above fail)

---

## Problem Scenario
Build a configuration object where keys come from a dictionary but should appear as attributes (`config.DB_HOST`) without being predefined.

---

## Code Example

```python
class Config:
    def __init__(self, source):
        self._source = source
        self._cache = {}

    def __getattr__(self, name):
        if name in self._source:
            value = self._source[name]
            self._cache[name] = value
            return value
        raise AttributeError(name)
