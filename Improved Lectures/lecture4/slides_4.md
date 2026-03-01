Slide 0
Title: Protocols and Behavior-Driven Design

Point 1: Python checks for the presence of methods at runtime rather than verifying inheritance. An object that defines write behaves like a writable destination. Python never asks what class you are — it asks what you can do.

Slide 1
Title: What Is Behavior-Driven Design

Point 1: Behavior-driven design means you define compatibility through method signatures, not through inheritance trees. If two objects both have a method named process that accepts the same kind of input, they are interchangeable regardless of ancestry.

Slide 2
Title: Why Behavior-Driven Design Matters in Practice

Point 1: When building a data pipeline with components from different libraries and class hierarchies, forcing them into a shared inheritance tree becomes painful. With behavior-driven design, each component just needs a method named process — no shared ancestor required.

Slide 3
Title: Protocol-Based Extensibility

Point 1: Protocol-based extensibility means new components can join a system without modifying the system itself. The built-in print function can write to the console, a file, a network socket, or a custom logger — it only checks whether the destination has a method named write.

Slide 4
Title: Plugin Architectures and Protocols

Point 1: A web framework defines that middleware must have a method named handle. Any object that provides handle is a valid middleware — no class registration or framework inheritance required. New middleware can be added or swapped without touching the framework code.

Slide 5
Title: The Strategy Pattern Without Inheritance

Point 1: The strategy pattern lets you swap behavior at runtime. In Python, you do not need an abstract base class to achieve this. Any object or function with the expected method signature qualifies as a strategy.

Slide 6
Title: Strategy Pattern in a Pricing Engine

Point 1: A pricing engine can swap between RegularDiscount, HolidayDiscount, and LoyaltyDiscount at runtime simply because all three define a method named calculate. No DiscountStrategy base class or inheritance hierarchy is required.

Slide 7
Title: How Django Uses Protocols

Point 1: Django checks whether each field object provides methods like contribute_to_class and get_prep_value. Any object that implements those methods works as a field — you can write custom fields without inheriting from a Django base class.

Slide 8
Title: How Flask and Pandas Use Protocols

Point 1: Flask route handlers are any callable — a function, method, or object with __call__. Pandas can index into any object with __getitem__ and iterate over any object with __iter__. These frameworks define behavioral contracts, not rigid class hierarchies.

Slide 9
Title: Python as a Glue Language Through Protocols

Point 1: When connecting a database adapter to a web framework or a machine learning model to a data pipeline, you do not need adapter classes or complex wrappers. You need objects that speak the same behavioral language — and protocols are that language.

Slide 10
Title: What Breaks with Inheritance Hierarchies

Point 1: When a third-party WebhookNotifier already has its own class hierarchy, forcing it into a Notifier inheritance tree requires adapters, multiple inheritance, or a full refactor. With protocols, it only needs a method named send to join the system immediately.

Slide 11
Title: Tight Coupling Is the Cost of Inheritance

Point 1: When you change a base class signature or add a required method, every subclass must update or it breaks. Protocols avoid this entirely — each object is responsible for its own behavior and changes do not cascade through a class tree.

Slide 12
Title: Final Takeaways

Point 1: Protocols enable behavior-driven design where objects become interchangeable through shared method signatures. Protocol-based extensibility lets new components join a system without modifying existing code. The strategy pattern, Django, Flask, and pandas all rely on this philosophy — protocols are what make Python uniquely extensible.
