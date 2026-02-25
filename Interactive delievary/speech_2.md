Alright everyone, let us get started.

In the previous lesson, we explored how Python syntax maps to special methods behind the scenes [slide 1 point 1]. We saw how operators, iteration, and built in functions translate into method calls on objects.

Today, we move one level higher [slide 1 point 2].

Instead of asking how syntax works, we ask a different question. How does Python decide whether an object qualifies to participate in a system [slide 1 point 3].

This lesson is about structural compatibility.

We will study why Python does not require formal declarations to determine whether something behaves like a sequence, a serializer, or a writable destination.

By the end of this session, you will understand how Python evaluates capability at runtime [slide 2 point 1], and how modern type checkers can express those same expectations explicitly without changing runtime flexibility.

What we are really studying today is how Python decides whether something fits into a system [slide 2 point 1].

The answer is not by what it is, but by what it can do [slide 2 point 2].

Qualification is based on behavior, not class hierarchy [slide 2 point 3].

Now let us begin with informal interfaces.

In many languages, if you want an object to behave like a sequence, you
must explicitly declare that relationship. You inherit from an
interface, you implement required methods, and the compiler enforces it
\[slide 3 point 1\].

Python does not work like that \[slide 3 point 2\].

Instead, Python asks a much simpler question at runtime \[slide 3 point
3\].

Do you have the methods I need right now \[slide 3 point 4\].

Almost like a restaurant asking, can you cook, not where did you study.

Let us consider this code block.

In \[module 1 line 1\], the instructor defines a class named Sequence.

Notice something important. It does not inherit from anything special.
It is just a normal class.

In \[module 1 line 2\], there is a constructor. It takes a value called
data, and stores it inside a variable named data that belongs to the
object.

You can imagine this data coming from a database query, or sensor
readings from a device. We are simply wrapping it.

In \[module 1 line 5\], there is a function definition for length
behavior. This tells Python how long the object is.

Because of that, built in functionality that asks for the length will
now work with this object.

In \[module 1 line 8\], there is another function definition that allows
item access by index.

This means the object can return an element when an index is requested.

Now think about what we have created.

It can be indexed. It has a length. It can be iterated.

Python treats it like a sequence, even though we never declared it as
one \[slide 4 point 1\].

That is an informal interface \[slide 4 point 2\]. You demonstrate
compatibility instead of declaring it \[slide 4 point 3\].

It is like plugging a device into your laptop. If it follows the
expected behavior, it works. No one asks who manufactured it.

Now let us move to duck typing.

If it walks like a duck and quacks like a duck, Python concludes that it
is acceptable \[slide 5 point 1\].

Python checks capabilities, not ancestry \[slide 5 point 2\].

Let us consider this code block.

In \[module 2 line 1\], there is a function definition named total. It
takes a variable named values.

In \[module 2 line 2\], a variable named result is initialized to zero.

In \[module 2 line 3\], the function iterates over whatever was passed
in.

In \[module 2 line 4\], each element is added to the variable named
result.

In \[module 2 line 5\], the function returns the accumulated value.

In \[module 2 line 7\] and \[module 2 line 8\], the function is called
with different kinds of collections.

The function assumes only two things. The object can be iterated, and
its elements support addition \[slide 5 point 3\].

Lists work. Tuples work.

You could even pass a custom generator that reads numbers from a file,
and it would still work as long as it behaves correctly.

Python never checks the type explicitly. Only behavior matters \[slide 5
point 4\].

That is duck typing \[slide 5 point 5\].

In real world systems, this is powerful. Imagine writing a payment
processor. As long as the object provides a process behavior, it does
not matter whether it represents a credit card, a digital wallet, or
online banking.

Behavior defines compatibility \[slide 6 point 1\].

Now let us talk about behavior driven design.

Instead of building deep inheritance trees, Python encourages us to
focus on shared behavior \[slide 6 point 2\].

Let us consider this code block.

In \[module 3 line 1\], a class named JsonSerializer is defined.

In \[module 3 line 2\], it defines a function named serialize.

In \[module 3 line 6\], another class named XmlSerializer is defined.

In \[module 3 line 7\], it also defines a function named serialize.

In \[module 3 line 10\], there is a function definition named save. It
takes a serializer and some data.

In \[module 3 line 11\], it calls the serialize behavior on the provided
object.

In \[module 3 line 13\] and \[module 3 line 14\], the save function is
invoked with different serializer objects.

Both serializers share behavior, not ancestry \[slide 6 point 3\].

They both provide a serialize capability \[slide 6 point 4\].

The save function does not care whether the format is JSON, XML, or
something else. It only cares that the serialize behavior exists \[slide
6 point 5\].

This is a strategy style approach without unnecessary ceremony \[slide 6
point 6\].

Behavior driven systems are easier to extend, loosely coupled, and
simpler to maintain \[slide 6 point 7\].

If tomorrow you need another format, you simply create another object
that provides serialize. You do not modify existing classes \[slide 6
point 8\].

Now let us discuss protocol based extensibility.

Imagine you are building a logging system. It should be able to write to
a file, to memory, or even to a network connection.

What matters is that the object can write \[slide 7 point 1\].

Let us consider this code block.

In \[module 4 line 1\], a class named MemoryBuffer is defined.

In \[module 4 line 2\], there is a constructor that initializes a
variable named data as an empty value.

In \[module 4 line 5\], a function named write is defined. It takes some
text and appends it to the internal data.

In \[module 4 line 8\] and \[module 4 line 9\], an instance is created
and the write behavior is invoked.

Anything that provides write behaves like a file from the perspective of
a logging system \[slide 7 point 2\].

You could replace it with a real file object, and the rest of the system
would not notice \[slide 7 point 3\].

Now consider static structural protocols.

Let us consider this code block.

In \[module 5 line 1\], a protocol base is imported for type checking
purposes.

In \[module 5 line 3\], a class named Writable is defined as a protocol.

In \[module 5 line 4\], it specifies that any conforming object must
provide a write behavior that accepts text.

In \[module 5 line 7\], there is a function definition named log. It
expects something that satisfies the Writable protocol.

In \[module 5 line 8\], it invokes the write behavior.

In \[module 5 line 10\], the log function is called with a MemoryBuffer
object.

Here we are making expectations explicit for type checkers \[slide 8
point 1\].

Your development tools can now warn you if you pass something that does
not provide write \[slide 8 point 2\].

At runtime, nothing changes. Python still relies on behavior \[slide 8
point 3\].

It is like defining a job description that says must be able to write
reports. Anyone who satisfies that qualifies, regardless of their
background.

Type checkers enforce expectations. Runtime remains flexible \[slide 8
point 4\].

Let us conclude with key takeaways.

Python favors behavior over inheritance \[slide 9 point 1\].

It supports informal interfaces \[slide 9 point 2\].

It encourages trust based design \[slide 9 point 3\].

It enables protocol driven extensibility \[slide 9 point 4\].

Compatibility comes from capability, not from class hierarchy \[slide 9
point 5\].

When designing systems, ask yourself what behavior do I require \[slide
9 point 6\].

Do not begin with what class should this inherit from \[slide 9 point
7\].

For homework, refactor an older class design you created.

If you built a deep inheritance tree just to reuse a method, try
removing inheritance that exists only for compatibility \[slide 10 point
1\].

Replace it with behavior based design or protocols \[slide 10 point 2\].

Document the original structure, the removed inheritance, the behaviors
you relied on, and why the new design is cleaner \[slide 10 point 3\].
