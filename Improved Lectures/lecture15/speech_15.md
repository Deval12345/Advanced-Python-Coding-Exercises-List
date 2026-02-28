# Speech — Lecture 15: Attribute Access Control — __getattr__ and __getattribute__

---

In the last lecture we explored descriptors — the mechanism that lets you attach programmable behavior to specific, named attributes at the class level. Today we go one level deeper into Python's object system. We're going to look at two special methods that sit at the very core of how Python resolves every attribute access you ever write: __getattr__ and __getattribute__. These are not obscure corners of the language. They are the foundation that makes Django ORM queries work, that powers Python's mock testing library, that enables proxy objects and access logging in production systems. Understanding them turns a lot of previously mysterious Python behavior into something completely readable.

Let's start with the foundation: the attribute lookup order.

Every time you write obj.something in Python, the interpreter runs through a precise sequence of steps. First, it looks for a data descriptor on the class and its MRO — things like properties and __slots__ entries. Data descriptors define both __get__ and __set__, so they take priority over instance data. Second, if no data descriptor is found, Python checks the instance's __dict__ directly. Third, if the attribute isn't in __dict__, Python checks for non-data descriptors on the class — regular methods, for instance. If all three of those steps fail to produce an attribute, Python calls __getattr__ as a last resort.

Now here is the critical distinction: __getattr__ only fires when normal lookup fails. __getattribute__, on the other hand, is called for every single attribute access, successful or not. It sits before the entire chain — it is the entry point. Most of the time you never override __getattribute__ because Python's default implementation does the right thing. But when you do override it, you have to be extremely careful: calling self.anything inside __getattribute__ immediately triggers __getattribute__ again, and you have an infinite recursion. The correct pattern is to bypass self entirely and call object.__getattribute__(self, 'name') to access your own instance data.

So let's look at __getattr__ first, since it's the safer and more commonly used tool.

The most practical use case for __getattr__ is dynamic attribute generation. Imagine a configuration object where you want to access dictionary keys using dot notation — config.host instead of config["host"]. You could define every key as a property, but that defeats the purpose of a flexible config system. Instead, you implement __getattr__. When someone writes config.host, Python checks the class for a host descriptor — not found. It checks the instance __dict__ — not found there either. Normal lookup fails, so Python calls __getattr__("host"). Your implementation looks up "host" in the internal data dictionary and returns it. If the key doesn't exist, you raise AttributeError, which is the correct contract.

Notice something important: _configData itself is stored directly in __dict__ during __init__, so accessing self._configData inside __getattr__ does not trigger another __getattr__ call — that access resolves normally through __dict__. This is the property that makes __getattr__ safe: it only fires for attributes that genuinely don't exist.

The risk, though, is that __getattr__ can silently catch typos. If you misspell an attribute name on a class that has __getattr__, you might get a default value or a custom error instead of a plain AttributeError that would make the bug obvious immediately. In production code, __getattr__ implementations should be explicit about which attributes are dynamic and should always raise AttributeError for names they don't recognize.

Now let's look at __getattribute__, which is a more powerful and more dangerous tool.

__getattribute__ intercepts every attribute access. It completely replaces Python's default lookup chain — the entire sequence I described at the start. When you override it, you are responsible for implementing whatever behavior you want, including calling object.__getattribute__ to get to the actual data. This is why the pattern for accessing your own instance state inside __getattribute__ is always object.__getattribute__(self, 'attributeName') — not self.attributeName, which would recurse.

A canonical example is the ReadOnlyProxy. The proxy wraps any object and prevents any attribute from being set through the proxy interface. In __init__, we use object.__setattr__ rather than self._target = targetObject, because the latter would trigger our own __setattr__, which raises an error unconditionally. In __getattribute__, we check if the requested name is _target — if so, we retrieve it directly via object.__getattribute__ to avoid recursion. For everything else, we grab the wrapped target, then use getattr to fetch the attribute from it. This means every read goes through to the real object, but every write attempt is blocked.

Notice how clean the interface is from the outside. You create a Configuration, wrap it in ReadOnlyProxy, and then proxy.host works perfectly. proxy.host = "newvalue" raises AttributeError immediately with a clear message. The underlying config object is still mutable directly if you have a reference to it — the proxy just enforces read-only access through its own interface.

Now let's talk about why this matters at scale.

The LoggingProxy combines everything we've talked about. It wraps any object and records every attribute access with a timestamp. When you access proxy.name, the proxy logs the access, then retrieves and returns the actual value. After several accesses, you can call proxy.getAccessLog() and get a complete timeline of what was accessed and when. This is exactly the pattern used in profiling tools, in security audit systems that need to track which fields in a user record were read during a request, and in debugging tools that need to understand the sequence of attribute access in a complex object graph.

Notice the design in __getattribute__: we check if the name starts with an underscore first. Private names get forwarded directly to object.__getattribute__ so we can access our own internal state. Public names go through the logging path. This is a clean convention that avoids the recursion problem while keeping the interface simple.

The key insight here is that __getattr__ and __getattribute__ are not just low-level implementation details. They are the mechanism that makes entire categories of Python libraries possible. Django's QuerySet is lazy — it doesn't hit the database until you iterate. That laziness is implemented through exactly these hooks. SQLAlchemy tracks which attributes on your model have been modified, so it only writes changed columns to the database. unittest.mock generates a response for any attribute access or method call on a Mock object. All of that is __getattr__ and __getattribute__.

Let's bring it all together.

__getattr__ is the fallback for missing attributes. It's safe, it only fires when needed, and it's the right tool for dynamic attribute generation, forwarding, and flexible configuration objects. __getattribute__ is total interception — every access goes through it. It requires careful use of object.__getattribute__ to avoid infinite recursion, but it enables read-only proxies, access logging, and complete object virtualization. Together with __setattr__ and __delattr__, these hooks give you full control over the attribute interface of any Python object.

The attribute lookup order — data descriptors, instance __dict__, non-data descriptors, then __getattr__ — is the foundation of Python's object system. Knowing it precisely will help you debug confusing AttributeErrors, understand ORM behavior, and write clean proxy and delegation patterns. In the next lecture we'll look at Abstract Base Classes, which give us a different kind of control: enforcing that a class implements a complete interface, rather than controlling how attributes are accessed.
