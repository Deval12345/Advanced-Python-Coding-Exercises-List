Slide 0
Title: Previous Lecture Takeaway
Point 1: Protocols and duck typing let objects participate based on behavior; today we extend that with attribute access interception when attributes are not yet defined.

Slide 1
Title: Why Attribute Interception Exists
Point 1: In real systems not all attributes are known when the class is written; configuration, feature flags, and proxies require responding to unknown attributes.
Point 2: Python follows a fixed lookup order; *getattr* is only called when normal lookup fails.

Slide 2
Title: The special dunder method getattr
Point 1: *getattr* is called only when the attribute does not exist after normal lookup; use it to materialize attributes lazily or provide virtual attributes.
Point 2: Using *getattr* for lazy loading keeps the public API clean; the object decides when to load and cache.

Slide 3
Title: Proxy and Virtual Attributes
Point 1: A proxy can forward attribute access to another system; *getattr* runs when the attribute is missing, enabling virtual attributes.

Slide 4
Title: getattribute
Point 1: *getattribute* intercepts every attribute access before normal lookup; you must call super to perform normal lookup to avoid infinite recursion.
Point 2: Use *getattr* for missing attributes; use *getattribute* when you need to observe or control every access.

Slide 5
Title: Final Takeaways
Point 1: Attribute lookup has a fixed order; *getattr* enables lazy loading and proxies; *getattribute* intercepts all access and must delegate to super; choosing the right hook keeps code maintainable.
