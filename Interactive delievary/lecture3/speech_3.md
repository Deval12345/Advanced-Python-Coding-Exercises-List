In the previous session we saw that protocols and duck typing let objects participate in Python based on behavior. Today we go one layer deeper. We look at how Python resolves attribute access and how you can intercept it when an attribute is not yet known or not yet loaded.

By the end of this lesson, you will understand Python’s attribute lookup order, when the special dunder method getattr runs, and when to use it for lazy loading and proxy objects.

In real systems, not all attributes are known when the class is written. Configuration keys may be loaded from files or environment variables. Feature flags may be added without code changes. Proxy objects may wrap remote services. Python allows attribute access interception so that objects can respond meaningfully even when an attribute has not been defined in advance.

Python follows a fixed lookup order when you access an attribute on an object. It checks the instance dictionary first, then class attributes and descriptors, then base classes. Only if normal lookup fails does Python call the special dunder method getattr. So getattr is a fallback for missing attributes, not for every access.

The special dunder method getattr is called only when the attribute does not exist after normal lookup. It is the right place to materialize attributes lazily or to provide virtual attributes.

Here we define a class named Config. Inside it, we store a source and a cache in variables named sourceStore and valueCache. We implement the special dunder method getattr with a parameter named name. Inside getattr we check if name is in the cache and return the cached value if present. Otherwise we load the value from the source, store it in the cache, and return it. Because the class defines the special dunder method getattr, when we access an attribute on a Config instance that is not found by normal lookup, Python calls this method so we can load and cache configuration on first access.

Using the special dunder method getattr for lazy loading keeps the public API clean. Callers use normal attribute access. The object decides when to load and when to cache. This pattern appears in configuration objects, feature flags, and adapters over external data.

A proxy object can forward attribute access to another system. When an attribute is requested, the proxy may fetch it from a remote API, a database, or another object. The special dunder method getattr is the hook that runs when the attribute is missing, so the proxy can implement virtual attributes.

Here we define a class named RemoteAPI. Inside it, we implement the special dunder method getattr with a parameter named name. The method returns a string that represents a remote value for that name. Because the class defines the special dunder method getattr, when we access an attribute on a RemoteAPI instance that is not on the instance, Python calls getattr and the proxy can return a virtual value or trigger a remote fetch.

The special dunder method getattribute is different from getattr. It intercepts every attribute access, before normal lookup. It runs first, so it can be used for tracing, monitoring, or full virtualization. You must call super and then getattribute to perform normal lookup, or you will get infinite recursion.

Here we define a class named Tracer. Inside it, we implement the special dunder method getattribute with a parameter named name. The method prints the name being accessed, then returns the result of calling super and then getattribute with name. Because getattribute runs for every access, we see a log line when we set or read any attribute. The call to super is required so that the actual attribute value is retrieved and the recursion stops.

Use the special dunder method getattr when you only need to handle missing attributes. Use the special dunder method getattribute when you need to observe or control every access. In most application code, getattr is enough for lazy loading and proxies.

Attribute lookup has a fixed order. The special dunder method getattr is the fallback for missing attributes and enables lazy loading, caching, and proxy objects. The special dunder method getattribute intercepts all access and must delegate to super to avoid infinite recursion. Choosing the right hook keeps your objects predictable and your code maintainable.
