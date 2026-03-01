
Welcome back.

In the previous session, we explored the Python Data Model and understood how Python syntax connects to special methods such as the special dunder method len, get item, iter, and add.

We observed that objects participate in Python syntax by implementing specific methods, and that behavior matters more than inheritance.

Today, we build directly on that idea.

If special methods allow objects to participate in syntax, protocols explain how Python decides whether an object is compatible with a system based purely on behavior.

Python uses informal interfaces.

Instead of forcing you to explicitly inherit from an interface, Python simply checks whether the required methods exist at runtime.

It does not ask what class you are.

It asks whether you have the behavior needed right now.

This is why Python feels flexible and natural.

Implementing the special dunder method len and the special dunder method getitem makes an object behave like a sequence.

When these methods exist, calling len on the object works.

Accessing elements by index works.

Iteration works.

You never declared that the object is a sequence.

You demonstrated the required behavior.

Duck typing means compatibility is determined by behavior, not by type.

If an object supports iteration and addition, it can be used in a summation function.

Python does not verify ancestry.

It verifies capability.

Python systems are designed around behavior.

Objects become interchangeable when they share method signatures, even without shared inheritance.

This enables loosely coupled systems.

Protocols formalize behavior expectations without enforcing inheritance.

If an object implements the required methods, it satisfies the protocol.

Python favors behavior over inheritance.

It favors informal interfaces.

It encourages duck typing.

It supports protocol driven extensibility.

Compatibility comes from capability.
