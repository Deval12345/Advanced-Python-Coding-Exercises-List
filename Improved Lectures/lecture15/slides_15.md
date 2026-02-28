# Slides — Lecture 15: Attribute Access Control — __getattr__ and __getattribute__

---

## Slide 1
**Title:** Attribute Access Control — __getattr__ and __getattribute__
- Descriptors control specific named attributes at the class level
- __getattr__ and __getattribute__ control attribute lookup itself
- These hooks are how proxy objects, ORMs, and mock libraries work
- Understanding them turns Python "magic" into readable, debuggable code
- Today: the full attribute lookup chain and both interception hooks

---

## Slide 2
**Title:** Python's Attribute Lookup Order
- Step 1: Check data descriptors on the class and its MRO (define both __get__ and __set__)
- Step 2: Check the instance's __dict__ directly
- Step 3: Check non-data descriptors on the class (e.g., methods)
- Step 4: If all three fail, call __getattr__ as a last resort
- __getattribute__ intercepts everything — it runs before this chain begins

---

## Slide 3
**Title:** Why the Lookup Order Matters
- Descriptor takes priority over instance __dict__ — this is intentional for validation
- Instance __dict__ beats non-data descriptors — instance data wins over class-level methods
- __getattr__ only fires when lookup fails — safe to implement without interfering with real attributes
- Confusing AttributeErrors in complex hierarchies often trace back to this chain
- Knowing the order cold eliminates a whole class of hard-to-reproduce bugs

---

## Slide 4
**Title:** __getattr__ — The Fallback Handler
- Called only when normal lookup fails — it is the last resort
- If the attribute exists in __dict__ or as a descriptor, __getattr__ is never invoked
- Safe for dynamic attribute generation: only activates when genuinely needed
- Common uses: dynamic API clients, config dotted access, forwarding to wrapped objects
- Risk: can silently catch typos — always raise AttributeError for unknown names

---

## Slide 5
**Title:** Example 15.1 — DynamicConfig
- Stores configuration data in a private dict (_configData)
- __getattr__ looks up the key in _configData when normal lookup fails
- config.host, config.port, config.database all resolve through __getattr__
- _configData itself resolves through __dict__ — no recursion risk
- Raises AttributeError with a clear message for unknown keys

---

## Slide 6
**Title:** __getattribute__ — Total Interception
- Called for every attribute access, successful or not
- Completely replaces Python's default lookup chain
- Critical danger: self.anything inside __getattribute__ calls __getattribute__ again — infinite recursion
- Correct pattern: object.__getattribute__(self, 'name') to access own instance data
- Used for: access logging, read-only enforcement, proxy objects, lazy initialization

---

## Slide 7
**Title:** Example 15.2 — ReadOnlyProxy
- Wraps any object; blocks all attribute writes through the proxy
- __init__ uses object.__setattr__ to store _target — avoids triggering own __setattr__
- __getattribute__ checks for _target specially, forwards everything else to the wrapped object
- __setattr__ raises AttributeError unconditionally — proxy is fully read-only
- Underlying object is still mutable directly; only the proxy interface is locked

---

## Slide 8
**Title:** The Recursion Problem and the Solution
- Inside __getattribute__, self.x recursively calls __getattribute__('x') — stack overflow
- Solution: object.__getattribute__(self, 'x') bypasses the override entirely
- Same applies to __setattr__: use object.__setattr__(self, 'x', val) in __init__
- Convention: check if name starts with '_' to fast-path internal state
- This is not a workaround — it is the intended design pattern

---

## Slide 9
**Title:** Proxy Pattern and Object Virtualization
- Combining __getattr__, __getattribute__, __setattr__ gives full object virtualization
- The proxy behaves like the wrapped object without holding its data directly
- Can log every access, enforce permissions, delay instantiation, redirect to remote objects
- Django lazy QuerySet: no DB query until you iterate — implemented via these hooks
- SQLAlchemy dirty checking: tracks attribute changes for efficient database writes

---

## Slide 10
**Title:** Example 15.3 — LoggingProxy
- Records every public attribute access with a timestamp
- Private names (starting with '_') route to object.__getattribute__ to avoid recursion
- Public names: retrieve target, log the access, return the value
- getAccessLog() returns the full timeline of what was accessed
- Pattern used in profiling, security auditing, and debugging complex object graphs

---

## Slide 11
**Title:** Industry Applications
- unittest.mock: generates responses for any attribute access or method call dynamically
- SQLAlchemy: instrumented attributes track dirty fields before database flush
- Django QuerySet: lazy evaluation delays DB query until iteration
- API client libraries: client.users.list() generated from attribute chain, not hard-coded
- All of these rely on __getattr__ and/or __getattribute__ at their core

---

## Slide 12
**Title:** Lecture 15 Takeaways
- Attribute lookup order: data descriptors → instance __dict__ → non-data descriptors → __getattr__
- __getattr__: safe fallback for missing attributes; only fires when lookup fails
- __getattribute__: total interception; requires object.__getattribute__ to avoid recursion
- Proxy pattern combines both hooks with __setattr__ for full object virtualization
- These hooks are how Django, SQLAlchemy, and mock libraries produce their "magic" behavior
