Welcome back.

### CONCEPT 0.1

In the previous session, we explored the Python Data Model and understood how Python syntax connects to special methods such as the special dunder method len, get item, iter, and add.

We observed that objects participate in Python syntax by implementing specific methods, and that behavior matters more than inheritance.

Today, we build directly on that idea.

If special methods allow objects to participate in syntax, protocols explain how Python decides whether an object is compatible with a system based purely on behavior.

------------------------------------------------------------------------

## Section 1 --- Informal Interfaces

### CONCEPT 1.1

Python uses informal interfaces.

Instead of forcing you to explicitly inherit from an interface, Python simply checks whether the required methods exist at runtime.

It does not ask what class you are.

It asks whether you have the behavior needed right now.

This is why Python feels flexible and natural.

### EXAMPLE 1.1

Here we define a class named LengthContainer.

Inside it, we implement the special dunder method len.

The class stores a value inside a variable named dataStore.

The special dunder method len returns the length of dataStore.

Because the class LengthContainer defines the special dunder method len, when we call len on lengthContainer, Python checks whether the object provides that method and then uses it to compute the length.

Python does not check inheritance.

It checks whether the required behavior exists.

That is an informal interface.

------------------------------------------------------------------------

### CONCEPT 1.2

Implementing the special dunder method len and the special dunder method getitem makes an object behave like a sequence.

When these methods exist, calling len on the object works.

Accessing elements by index works.

Iteration works.

You never declared that the object is a sequence.

You demonstrated the required behavior.

------------------------------------------------------------------------

## Section 2 --- Duck Typing

### CONCEPT 2.1

Duck typing means compatibility is determined by behavior, not by type.

If an object supports iteration and addition, it can be used in a summation function.

Python does not verify ancestry.

It verifies capability.

### EXAMPLE 2.1

Here we define a class named NumberStream.

Inside it, we implement the special dunder method iter.

The special dunder method iter returns an iterator over a list containing the values one, two, and three.

Because the class NumberStream defines the special dunder method iter, when we pass numberStream to sum, Python checks whether the object supports iteration and then consumes the values.

Functions like sum work automatically when the required behavior is present.

That is duck typing.

------------------------------------------------------------------------

## Section 4 --- Behavior-Driven Design

### CONCEPT 3.1

Python systems are designed around behavior.

Objects become interchangeable when they share method signatures, even without shared inheritance.

This enables loosely coupled systems.

### EXAMPLE 3.1

Here we define a class named DiscountCalculator.

Inside it, we define a method named apply.

The method named apply receives a parameter named value.

The method returns the value multiplied by zero point nine.

Because the class DiscountCalculator defines a method named apply, any other object that defines apply with the same structure can act in the same role.

Compatibility comes from shared behavior, not from sharing a base class.

------------------------------------------------------------------------

## Section 5 --- Protocol-Based Extensibility

### CONCEPT 4.1

Protocols formalize behavior expectations without enforcing inheritance.

If an object implements the required methods, it satisfies the protocol.

### EXAMPLE 4.1

Here we define a class named SimpleWriter.

Inside it, we implement a method named write.

The method named write receives a parameter named text.

The method prints the text.

Because the class SimpleWriter defines a method named write, any object that provides a write method with similar behavior can act as a writable object.

That is protocol based compatibility.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 5.1

Python favors behavior over inheritance.

It favors informal interfaces.

It encourages duck typing.

It supports protocol driven extensibility.

Compatibility comes from capability.
