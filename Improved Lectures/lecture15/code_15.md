# Code — Lecture 15: Attribute Access Control — __getattr__ and __getattribute__

---

## Example 15.1 — DynamicConfig

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

**Output:**
```
localhost
5432
production_db
```

**Explanation:**
- Line 1: Define DynamicConfig, a class that will expose dictionary keys as dot-access attributes.
- Line 2: __init__ accepts a dictionary of configuration data.
- Line 3: Store the dict in _configData. Because this assignment happens in __init__ via normal assignment, _configData goes directly into the instance's __dict__. __getattr__ is not triggered here.
- Line 4: Define __getattr__, which Python calls only when normal attribute lookup fails — i.e., when the attribute is not found in __dict__ or in any class descriptor.
- Line 5: Check if the requested attribute name exists as a key in _configData. Accessing self._configData here resolves through __dict__ normally, not through __getattr__ again, because _configData is a real instance attribute.
- Line 6: If the key exists, return its value from the dictionary.
- Line 7-8: If the key does not exist, raise AttributeError with a clear, informative message. This is the correct contract for __getattr__ — it should raise AttributeError for unknown names, not return a default silently.
- Line 9-12: Instantiate DynamicConfig with a dictionary. The keys host, port, and database are not defined as class attributes or instance properties.
- Line 13: config.host triggers the lookup chain: no descriptor named host, no __dict__ entry named host. Normal lookup fails, so __getattr__("host") is called. It finds "host" in _configData and returns "localhost".
- Line 14: Same process for port — returns 5432.
- Line 15: Same process for database — returns "production_db".

---

## Example 15.2 — ReadOnlyProxy

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

**Output:**
```
localhost
8080
```

**Explanation:**
- Line 1: Define ReadOnlyProxy, which will wrap any object and block attribute writes.
- Line 2: __init__ receives the object to wrap.
- Line 3: Use object.__setattr__ to store _target on the proxy instance. We cannot use self._target = targetObject here, because that would call our own __setattr__ (line 9), which raises AttributeError unconditionally. Going directly to object.__setattr__ bypasses our override.
- Line 4: Override __getattribute__, which is called for every attribute access on the proxy — not just missing ones.
- Line 5: Special-case the name '_target'. If we returned object.__getattribute__(self, attributeName) for all names, the _target check would be safe, but we want to be explicit.
- Line 6: Retrieve _target directly via object.__getattribute__ to avoid infinite recursion. Calling self._target here would call __getattribute__ again.
- Line 7: For all other attribute names, first retrieve the wrapped target object using object.__getattribute__ to avoid recursion.
- Line 8: Use getattr (Python's standard attribute lookup) to get the attribute from the actual target object. This gives the caller transparent access to the target's attributes.
- Line 9: Override __setattr__ on the proxy.
- Line 10: Raise AttributeError unconditionally for any attribute set. Any attempt to write an attribute through the proxy fails immediately with a clear message.
- Lines 11-14: Define a regular Configuration class with two mutable attributes.
- Line 15: Create a normal Configuration instance.
- Line 16: Wrap it in ReadOnlyProxy.
- Lines 17-18: Read access works transparently — proxy.host and proxy.port are forwarded to config.host and config.port. Any attempt to write (e.g., proxy.host = "newvalue") would raise AttributeError: "This object is read-only".

---

## Example 15.3 — LoggingProxy

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

**Output:**
```
['1709298765.123: accessed UserProfile.name',
 '1709298765.124: accessed UserProfile.email',
 '1709298765.124: accessed UserProfile.name']
```
(Timestamps will vary; format is UNIX time to 3 decimal places.)

**Explanation:**
- Line 1: Import time for timestamping access log entries.
- Line 2: Define LoggingProxy, which records every public attribute access on the wrapped object.
- Line 3: __init__ receives the target object and a human-readable name for log messages.
- Lines 4-6: Store _target, _name, and _accessLog using object.__setattr__ to bypass our __getattribute__ override. _accessLog starts as an empty list.
- Line 7: Override __getattribute__ to intercept all attribute access on the proxy.
- Line 8: Check if the attribute name starts with '_'. Private attributes are our own internal state.
- Line 9: For private names, delegate directly to object.__getattribute__ so we access our own state without logging it. This also handles special methods Python calls internally.
- Lines 10-12: For public names, retrieve _target, _name, and _accessLog via object.__getattribute__ to avoid recursion while accessing proxy internals.
- Line 13: Retrieve the actual attribute value from the wrapped target using getattr.
- Line 14: Append a timestamped log entry. The format shows UNIX time to 3 decimal places, followed by the proxy name and attribute name. This is the audit trail.
- Line 15: Return the value to the caller — the proxy is transparent from the outside.
- Line 16: Define getAccessLog as a regular public method.
- Line 17: Return _accessLog using object.__getattribute__ — since _accessLog starts with '_', it would follow line 8-9's fast path anyway, but being explicit here makes intent clear.
- Lines 18-21: Define a simple UserProfile class.
- Line 22: Create a profile instance with name and email.
- Line 23: Wrap it in LoggingProxy labeled "UserProfile".
- Lines 24-26: Access proxy.name, proxy.email, proxy.name — each access goes through __getattribute__, logs the event, and returns the value. The underscore assignment discards the value since we only care about demonstrating logging.
- Line 27: Print the full access log. The output shows three entries, one for each attribute access, with timestamps that make the sequence clear.
