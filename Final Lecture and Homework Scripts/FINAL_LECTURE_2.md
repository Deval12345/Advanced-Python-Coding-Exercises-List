# Protocols in Python --- Behavior over Inheritance

## Full Instructor Performance Script

### Lesson Opening

Alright everyone, let's get started.

Today we're going to talk about a very Pythonic idea --- protocols, and
more broadly, why Python prefers behavior over inheritance.

Now, this is one of those topics that quietly explains why Python code
feels flexible, even when no one explicitly taught you the rules. By the
end of this session, you'll understand why Python doesn't obsess over
class hierarchies, why things "just work," and how you can design
systems that feel natural instead of rigid.

What we're really studying today is how Python decides whether something
fits into a system.

And the answer is:\
👉 not by what it is, but by what it can do.

This lesson is based on the structured material you've seen:
**PROTOCOLS**.

------------------------------------------------------------------------

## Section 1 --- Informal Interfaces

Now let's start with the idea of informal interfaces.

In many languages, if you want an object to behave like a sequence,
you're forced to explicitly declare that relationship. You inherit from
an interface, you implement methods, and the compiler enforces it.

Python does not work like that.

Instead, Python asks a much simpler question at runtime:

> "Do you have the methods I need right now?"

### Instructor Explanation (Before Code)

I'm going to define a class called `Sequence`. Notice something
important --- it does not inherit from anything fancy. No abstract base
class. No interface. Just a plain class.

But we're going to give it a couple of very specific methods.

### Code

``` python
class Sequence:
    def __init__(self, data):
        self.data = data

    def __len__(self):
        return len(self.data)

    def __getitem__(self, index):
        return self.data[index]
```

### Instructor Walkthrough

We're defining a normal Python class. No inheritance here.

The constructor stores data internally.

`__len__` tells Python how long the object is.

`__getitem__` tells Python how to retrieve items by index.

Notice what we've created:

-   it can be indexed
-   it has a length
-   it can be iterated

Python treats it like a sequence --- without us ever declaring it.

That's an informal interface. You demonstrate compatibility instead of
declaring it.

------------------------------------------------------------------------

## Section 2 --- Duck Typing

Now let's move to duck typing.

"If it walks like a duck and quacks like a duck..."

Python finishes that sentence with:

"...then I don't care what class it is."

### Instructor Explanation (Before Code)

Python checks capabilities, not ancestry.

### Code

``` python
def total(values):
    result = 0
    for v in values:
        result += v
    return result

print(total([1, 2, 3]))
print(total((4, 5, 6)))
```

### Instructor Walkthrough

The function assumes:

-   the object is iterable
-   elements support addition

Lists work. Tuples work.

Python never checked their type. Only behavior mattered.

That's duck typing.

------------------------------------------------------------------------

## Section 3 --- Trust-Based Design Philosophy

Python follows the "trust based design" principle.

It trusts developers instead of enforcing strict walls.

### Code

``` python
class Account:
    def __init__(self):
        self._balance = 100  # protected by convention
```

The underscore signals intent --- not restriction.

Python encourages discipline instead of policing behavior.

This trust model aligns perfectly with protocols.

------------------------------------------------------------------------

## Section 4 --- Behavior-Driven Design

Instead of inheritance trees, Python focuses on behavior.

### Code

``` python
class JsonSerializer:
    def serialize(self, data):
        import json
        return json.dumps(data)

class XmlSerializer:
    def serialize(self, data):
        return "<data>" + str(data) + "</data>"

def save(serializer, data):
    return serializer.serialize(data)

print(save(JsonSerializer(), {"x": 1}))
print(save(XmlSerializer(), {"x": 1}))
```

Both serializers share behavior, not ancestry.

This is the Strategy pattern without ceremony.

Behavior-driven systems are:

-   easier to extend
-   loosely coupled
-   simpler to maintain

------------------------------------------------------------------------

## Section 5 --- Protocol-Based Extensibility

Protocols formalize expectations without inheritance.

### Code

``` python
class MemoryBuffer:
    def __init__(self):
        self.data = ""

    def write(self, text):
        self.data += text

buffer = MemoryBuffer()
buffer.write("Hello")
```

Anything with `.write()` behaves like a file.

### Static Structural Protocol

``` python
from typing import Protocol

class Writable(Protocol):
    def write(self, text: str) -> None:
        ...

def log(writer: Writable):
    writer.write("Log entry")

log(MemoryBuffer())
```

Type checkers enforce the protocol. Runtime remains flexible.

### Exercise 2 --- In-Class Guidance

Define a `Serializer` protocol. Update `save()` to use it.

Goal: improve clarity and tooling, not runtime behavior.

------------------------------------------------------------------------

## Final Takeaways

Python favors:

-   behavior over inheritance
-   informal interfaces
-   trust-based design
-   protocol-driven extensibility

**Compatibility comes from capability, not class hierarchy.**

------------------------------------------------------------------------

## Homework Guidance

Refactor an old class design.

Remove inheritance used only for compatibility.

Replace with behavior-based design or protocols.

Document:

-   original structure
-   removed inheritance
-   relied behaviors
-   why the new design is cleaner
