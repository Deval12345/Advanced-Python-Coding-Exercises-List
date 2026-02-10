
# Attribute Access Control (__getattr__ and __getattribute__)

## Overview

Python allows interception of attribute access to support objects whose state is:

- incomplete
- dynamic
- remote
- lazily loaded
- virtualized

Attribute access becomes programmable.

This enables object virtualization and proxy design.

This module focuses on why attribute interception exists, not just how.

---

## Section 1 — Attribute Lookup Order

When Python evaluates:

obj.name

Lookup order:

1. Instance dictionary
2. Class attributes
3. Descriptors
4. Base classes
5. __getattr__ fallback

__getattr__ only runs when normal lookup fails.

Descriptors control known attributes.
__getattr__ handles unknown ones.

This separation is intentional.

---

## Section 2 — __getattr__ (Missing Attribute Hook)

Called only when attribute does not exist.

Used to materialize attributes lazily.

```python
class Config:
    def __init__(self, source):
        self._source = source
        self._cache = {}

    def __getattr__(self, name):
        if name in self._cache:
            return self._cache[name]

        print("Loading", name)
        value = self._source.get(name)

        if value is None:
            raise AttributeError(name)

        self._cache[name] = value
        return value


source = {"DB_HOST": "localhost", "PORT": 5432}
config = Config(source)

print(config.DB_HOST)
print(config.DB_HOST)
```

Lazy loading + caching + clean API.

---

## Section 3 — Remote API Proxy Example

Attribute access triggers remote fetch.

```python
class RemoteAPI:
    def __getattr__(self, name):
        print("Fetching remote:", name)
        return f"<remote:{name}>"


api = RemoteAPI()
print(api.user_profile)
```

Attributes behave like access portals.

---

## Section 4 — __getattribute__ (Full Interception)

Intercepts all attribute access.

Runs before normal lookup.

```python
class Tracer:
    def __getattribute__(self, name):
        print("Access:", name)
        return super().__getattribute__(name)


t = Tracer()
t.x = 10
print(t.x)
```

Used for tracing, monitoring, security, virtualization.

Must call super() to avoid infinite recursion.

---

## Key Insight — Descriptors vs __getattr__

Descriptors → behavior of known attributes  
__getattr__ → materialization of unknown attributes

Different problems.
Different tools.

---

## Exercise — Virtual API Object

Create an object where any attribute returns computed data
without being predefined.

---

## Final Takeaways

Intercepting attribute access enables:

- object virtualization
- lazy loading
- proxy design
- dynamic APIs
- framework-level control

Attributes become programmable entry points.

