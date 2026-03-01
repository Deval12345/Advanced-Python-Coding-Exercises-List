# Key Points — Lecture 15: Attribute Access Control — __getattr__ and __getattribute__

---

## Attribute Lookup Order
- Python resolves obj.name in this exact sequence: (1) data descriptors on the class/MRO, (2) instance __dict__, (3) non-data descriptors, (4) __getattr__ as a last resort
- __getattribute__ runs before this chain — it is the entry point for every attribute access
- Understanding this order explains confusing AttributeErrors in complex class hierarchies

## __getattr__
- Only called when normal attribute lookup fails — it is the fallback, not the first stop
- Safe for dynamic attribute generation: real attributes in __dict__ are never affected
- Must raise AttributeError for unknown names — do not return defaults silently
- Practical uses: dotted config access, dynamic API clients, delegation/forwarding patterns

## __getattribute__
- Called for every attribute access, successful or not — it intercepts the entire lookup chain
- The infinite recursion trap: self.x inside __getattribute__ calls __getattribute__ again
- The correct pattern: object.__getattribute__(self, 'x') to access your own instance data
- Use object.__setattr__(self, 'x', val) in __init__ to store data without triggering __setattr__ overrides

## Proxy Pattern
- Combining __getattr__/__getattribute__ + __setattr__ + __delattr__ enables full object virtualization
- ReadOnlyProxy: wraps any object, forwards all reads, blocks all writes
- LoggingProxy: records every attribute access with a timestamp for audit and debugging
- The proxy is transparent to callers — they interact with it exactly as they would the wrapped object

## Industry Relevance
- Django lazy QuerySet: no database query until you iterate — implemented with these hooks
- SQLAlchemy instrumented attributes: dirty-field tracking for efficient DB writes
- unittest.mock: generates responses for any attribute or method call dynamically
- API client libraries: client.users.list() generated via __getattr__ chaining

## Rules to Remember
- __getattr__: use it, it is safe; just always raise AttributeError for unknowns
- __getattribute__: use it carefully; always call object.__getattribute__ for internal state
- Never use self.x inside __getattribute__ — always use object.__getattribute__(self, 'x')
- Never use self.x = val inside __init__ when you have a custom __setattr__ — use object.__setattr__
