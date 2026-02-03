
# Intercepting Attribute Access — __getattr__ & __getattribute__

## Overview

Python allows objects to intercept attribute access because not all objects
have fully known state at class definition time.

Some objects are:

- dynamic
- lazy-loaded
- remote
- virtualized
- proxy-based

Attribute access becomes a programmable interface, not just storage.

This mechanism enables object virtualization.

You will learn:

✔ Attribute lookup order  
✔ __getattr__ vs __getattribute__  
✔ Lazy attributes  
✔ Dynamic attribute creation  
✔ Proxy-style objects  
✔ Framework-level interception  

---

## Section 1 — Attribute Lookup Order (Critical)

When Python evaluates:

```
obj.name
```

It searches in this order:

1. Instance dictionary (obj.__dict__)
2. Class attributes
3. Descriptors
4. Base classes
5. __getattr__ fallback

Only after normal lookup fails does __getattr__ run.

This is intentional design.

Descriptors handle known attributes.
__getattr__ handles missing attributes.

---

## Section 2 — __getattr__ (Missing Attribute Hook)

__getattr__ is only called when attribute lookup fails.

It materializes attributes dynamically.

### Exercise — Lazy Config Loader

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
print(config.DB_HOST)  # cached
```

Outcomes:

✔ Dynamic attributes  
✔ Lazy loading  
✔ Caching  
✔ Clean API  
✔ Central fallback logic  

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

Attributes behave like access points, not storage.

This is how SDK proxies work.

---

## Section 4 — __getattribute__ (Full Interception)

__getattribute__ intercepts *all* attribute access.

It runs before normal lookup.

Dangerous but powerful.

### Example

```python
class Tracer:
    def __getattribute__(self, name):
        print("Access:", name)
        return super().__getattribute__(name)


t = Tracer()
t.x = 10
print(t.x)
```

Used for:

✔ Debug tracing  
✔ Security wrappers  
✔ Monitoring  
✔ Virtual objects  

Must call super() to avoid recursion.

---

## Key Insight — Descriptors vs __getattr__

Descriptors control attributes that exist on the class.

__getattr__ handles attributes that do not exist yet.

They solve different problems:

Descriptors → behavior of known attributes  
__getattr__ → dynamic materialization of unknown attributes

This separation is deliberate.

---

## Real-World Uses

Attribute interception powers:

✔ Lazy ORM relationships  
✔ API SDK proxies  
✔ Mocking frameworks  
✔ Debug tracers  
✔ Config systems  
✔ Virtual file systems  

These systems treat attributes as access portals.

---

## Exercise — Virtual API Object

Build an object where any attribute access returns a computed value.

Demonstrate dynamic behavior without predefining attributes.

---

## Final Takeaways

Intercepting attribute access enables:

✔ Object virtualization  
✔ Lazy loading  
✔ Dynamic behavior  
✔ Proxy design  
✔ Clean APIs  

Attributes become programmable entry points.

---

## Suggested Extensions

1. Build caching proxy
2. Implement remote object mirror
3. Create dynamic config system
4. Add audit tracer
5. Combine with descriptors

---

End of module.
