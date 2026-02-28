# Speech Source — Lecture 15: Attribute Access Control — __getattr__ and __getattribute__

---

## CONCEPT 0.1 — Transition from Previous Lecture

Descriptors gave us programmable attribute access at the class level — we could intercept reads and writes for specific, named attributes defined on the class. Now we go one step deeper into Python's object system. __getattr__ and __getattribute__ are Python's hooks for what happens when attribute lookup either fails entirely or is intercepted before it even begins. These are the tools behind proxy objects, dynamic APIs, ORM lazy loading, and full object virtualization.

---

## CONCEPT 1.1 — The Attribute Lookup Order

Python's attribute lookup follows a precise chain every time you write obj.something. First, Python checks data descriptors defined on the class and its MRO — things like properties and __slots__. Then it checks the instance's __dict__. Then it checks non-data descriptors (like regular methods). If all of these fail to produce a result, Python calls __getattr__ as a last resort. Understanding this order is essential because it determines which tool you reach for: __getattr__ is only called when normal lookup fails, while __getattribute__ intercepts every single access, including successful ones. Using __getattribute__ incorrectly is how you write a class that crashes with a RecursionError — because accessing self.anything inside __getattribute__ calls __getattribute__ again.

Industry impact: the first time most developers encounter this is while debugging a confusing AttributeError in a complex class hierarchy. Understanding the lookup order eliminates an entire class of confusing, hard-to-reproduce bugs. It also explains behavior you've seen in ORMs, mock libraries, and proxy objects that seemed almost magical.

---

## CONCEPT 1.2 — __getattr__: The Fallback Handler

__getattr__ is called when normal attribute lookup fails — it is the last resort in the chain. If an attribute exists in __dict__ or as a descriptor, __getattr__ is never called for it. This property makes __getattr__ safe for implementing dynamic attributes: the method only activates when it is actually needed. Common uses include generating API endpoint attributes dynamically (so client.users maps to "/api/users"), forwarding attribute access to a wrapped object, and implementing configuration objects that allow dotted access to nested dictionary keys.

The risk with __getattr__ is that it can silently swallow NameError-like conditions. If you misspell an attribute name, __getattr__ might return a default value or raise a custom error rather than a plain AttributeError that would make the bug obvious. Production systems should be explicit about which attributes are dynamic and which are fixed.

---

## EXAMPLE 1.1 — DynamicConfig

Narration: Here we have a configuration class that lets you access dictionary keys using dot notation. When you write config.host, Python first checks the class for a descriptor named host — there isn't one. It checks the instance __dict__ — host isn't in there either. Normal lookup fails, so Python calls __getattr__ with the string "host". Our __getattr__ looks up "host" in the internal _configData dictionary and returns it. Notice that _configData itself is stored directly in __dict__, so accessing self._configData inside __getattr__ does not trigger another __getattr__ call — it resolves normally. If the key doesn't exist, we raise AttributeError, which is the correct contract for __getattr__.

```python
# Example 15.1
class DynamicConfig:                                   # line 1
    def __init__(self, configData):                    # line 2
        self._configData = configData                  # line 3

    def __getattr__(self, attributeName):              # line 4
        if attributeName in self._configData:          # line 5
            return self._configData[attributeName]     # line 6
        raise AttributeError(                          # line 7
            f"Config has no setting: '{attributeName}'"  # line 8
        )

config = DynamicConfig({                               # line 9
    "host": "localhost",                               # line 10
    "port": 5432,                                      # line 11
    "database": "production_db"                        # line 12
})

print(config.host)                                     # line 13
print(config.port)                                     # line 14
print(config.database)                                 # line 15
```

---

## CONCEPT 2.1 — __getattribute__: Total Interception

__getattribute__ is called for every attribute access, not just missing ones. It completely overrides the lookup chain — Python calls it before checking __dict__, before checking descriptors, before anything else. The critical danger: calling self.anything inside __getattribute__ immediately calls __getattribute__ again, causing infinite recursion. To access the instance's own data, you must bypass self and call object.__getattribute__(self, '__dict__') or object.__getattribute__(self, 'attributeName') directly. This is not a workaround — it is the intended pattern. __getattribute__ is used in production for access logging, read-only enforcement, lazy initialization, and proxy objects that intercept all attribute access on a wrapped object.

---

## EXAMPLE 2.1 — ReadOnlyProxy

Narration: ReadOnlyProxy wraps any object and prevents any attribute from being set on the proxy itself. Look at __init__: we use object.__setattr__ rather than self._target = targetObject, because self._target = would trigger our own __setattr__ and raise an error. In __getattribute__, we check if the requested attribute is _target itself — if so, we retrieve it via object.__getattribute__ to avoid recursion. For all other attributes, we retrieve the wrapped target first, then use getattr to fetch the attribute from it. __setattr__ simply raises AttributeError unconditionally, making the proxy read-only. The target object can still be mutated directly; it's only the proxy interface that is locked.

```python
# Example 15.2
class ReadOnlyProxy:                                   # line 1
    def __init__(self, targetObject):                  # line 2
        object.__setattr__(self, '_target', targetObject)  # line 3

    def __getattribute__(self, attributeName):         # line 4
        if attributeName == '_target':                 # line 5
            return object.__getattribute__(self, '_target')  # line 6
        target = object.__getattribute__(self, '_target')  # line 7
        return getattr(target, attributeName)          # line 8

    def __setattr__(self, attributeName, value):       # line 9
        raise AttributeError("This object is read-only")  # line 10

class Configuration:                                   # line 11
    def __init__(self):                                # line 12
        self.host = "localhost"                        # line 13
        self.port = 8080                               # line 14

config = Configuration()                               # line 15
proxy = ReadOnlyProxy(config)                          # line 16
print(proxy.host)                                      # line 17
print(proxy.port)                                      # line 18
```

---

## CONCEPT 3.1 — Proxy Pattern and Object Virtualization

The combination of __getattr__, __getattribute__, and __setattr__ enables full object virtualization: you can construct an object that behaves like any other object without directly holding the data. The proxy layer can log every access, enforce permissions, delay instantiation until first use, or redirect operations to a remote object over a network. This is not theoretical — Django's lazy QuerySet uses exactly this pattern to avoid database queries until you actually iterate over results. SQLAlchemy's instrumented attributes use it to track field changes for dirty checking before a database flush. Python's unittest.mock uses it to dynamically generate responses to any attribute access or method call. When you understand these hooks, all of that behavior becomes readable and debuggable rather than magic.

---

## EXAMPLE 3.1 — LoggingProxy

Narration: LoggingProxy records every attribute access made through it, with a timestamp. We store _target, _name, and _accessLog using object.__setattr__ during __init__ to avoid triggering our own __getattribute__. In __getattribute__, we check if the name starts with an underscore — private names are forwarded to object.__getattribute__ to access our own internal state. For all other attributes, we retrieve the target and log, fetch the attribute value from the target, append a log entry, and return the value. The getAccessLog method is public-facing but uses object.__getattribute__ explicitly to retrieve _accessLog directly. After three accesses, the log shows each one with its timestamp.

```python
# Example 15.3
import time                                            # line 1

class LoggingProxy:                                    # line 2
    def __init__(self, targetObject, objectName):      # line 3
        object.__setattr__(self, '_target', targetObject)  # line 4
        object.__setattr__(self, '_name', objectName)  # line 5
        object.__setattr__(self, '_accessLog', [])     # line 6

    def __getattribute__(self, attributeName):         # line 7
        if attributeName.startswith('_'):              # line 8
            return object.__getattribute__(self, attributeName)  # line 9
        target = object.__getattribute__(self, '_target')  # line 10
        name = object.__getattribute__(self, '_name')  # line 11
        log = object.__getattribute__(self, '_accessLog')  # line 12
        value = getattr(target, attributeName)         # line 13
        log.append(f"{time.time():.3f}: accessed {name}.{attributeName}")  # line 14
        return value                                   # line 15

    def getAccessLog(self):                            # line 16
        return object.__getattribute__(self, '_accessLog')  # line 17

class UserProfile:                                     # line 18
    def __init__(self, name, email):                   # line 19
        self.name = name                               # line 20
        self.email = email                             # line 21

profile = UserProfile("alice", "alice@example.com")   # line 22
proxy = LoggingProxy(profile, "UserProfile")           # line 23
_ = proxy.name                                         # line 24
_ = proxy.email                                        # line 25
_ = proxy.name                                         # line 26
print(proxy.getAccessLog())                            # line 27
```

---

## CONCEPT 4.1 — Final Takeaway Lecture 15

__getattr__ is the fallback handler for missing attributes — it only fires when the normal lookup chain finds nothing, making it safe for implementing dynamic attribute generation without interfering with real attributes. __getattribute__ intercepts every attribute access and requires calling object.__getattribute__ explicitly to access the instance's own state without triggering infinite recursion. Together, these two hooks — along with __setattr__ and __delattr__ — enable full proxy objects, access logging, read-only enforcement, and object virtualization. Django, SQLAlchemy, and Python's testing infrastructure rely on these exact mechanisms. The attribute lookup order — data descriptors, instance dict, non-data descriptors, then __getattr__ — is the foundation of Python's entire object system, and knowing it cold will make you a significantly better Python developer.
