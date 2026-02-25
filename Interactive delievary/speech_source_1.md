# Python Data Model Magic Dunder Methods

## Lesson Opening

Alright everyone, let us get started.

Today we are going to talk about something that quietly powers everything you do in Python, the Python Data Model, also called magic methods or dunder methods.

Python is actually a framework, and the API of that framework is objects.

By the end of this lesson, you should be able to mentally translate normal Python syntax into method calls.

## Learning Goals

Understand Python as an object centric framework.

See how syntax like len of x maps to special methods.

Observe duck typing in action.

Design objects that behave like built ins.

Understand responsible operator overloading.

# 1. Python as a Framework

### CONCEPT 1.1

Python enables syntax participation through special methods.

If your object implements specific methods, Python allows it to participate in built in syntax.

### EXAMPLE 1.1

Here we define a class named LengthExample.

Inside it, we implement the special dunder method len.

The class stores values inside a variable named valueStore.

The special dunder method len returns the length of valueStore.

Because the class defines the special dunder method len, calling len on an instance triggers that method and returns the computed length.

### CONCEPT 1.2

len of x maps to x dot special dunder method len.

### EXAMPLE 1.2

Here we define a class named IndexContainer.

Inside it, we implement the special dunder method getitem.

The class stores values inside a variable named dataStore.

The special dunder method getitem returns the value at a given index.

Because the class defines the special dunder method getitem, accessing an instance using square bracket syntax translates into a method call that retrieves the indexed value.

### CONCEPT 1.3

A for loop over an object maps to the special dunder method iter.

### EXAMPLE 1.3

Here we define a class named IterableExample.

Inside it, we implement the special dunder method iter.

The class stores values inside a variable named dataStore.

The special dunder method iter returns an iterator over dataStore.

Because the class defines the special dunder method iter, a loop can obtain an iterator from the instance and iterate over its values.

### CONCEPT 1.4

The in keyword maps to the special dunder method contains.

### EXAMPLE 1.4

Here we define a class named ContainmentExample.

Inside it, we implement the special dunder method contains.

The class stores values inside a variable named dataStore.

The special dunder method contains returns a Boolean indicating membership.

Because the class defines the special dunder method contains, using the in keyword on an instance triggers that method to determine membership.

# 2. Custom Container

### CONCEPT 2.1

If a class implements the special dunder method len, the special dunder method getitem, the special dunder method contains, and the special dunder method iter, the object can behave like a native container.

# 3. Special Methods as Syntax Glue

### CONCEPT 3.1

Operators map to special dunder methods.

For example, addition maps to the special dunder method add.

### EXAMPLE 3.1

Here we define a class named PriceValue.

Inside it, we implement the special dunder method add.

The class stores a numeric value inside a variable named amount.

The special dunder method add returns a new instance of PriceValue with the combined amount.

Because the class defines the special dunder method add, using the plus operator between two instances invokes that method and produces a new PriceValue.

# 4. Duck Typing

### CONCEPT 4.1

Behavior matters more than type.

Objects are accepted based on the methods they implement, not based on inheritance.

### EXAMPLE 4.1

Here we define a class named DataStream.

Inside it, we implement the special dunder method iter.

The class stores values inside a variable named dataStore.

The special dunder method iter returns an iterator over dataStore.

Because the class defines the special dunder method iter, it behaves like an iterable and can be used anywhere an iterable is expected.

# 5. Lazy Indexable Objects

### CONCEPT 5.1

Indexing is enabled through the special dunder method getitem, allowing objects to compute values lazily when accessed by index.

# 6. Strategy Pattern

### CONCEPT 6.1

Objects that share the same method signature can act as interchangeable strategies.

### EXAMPLE 6.1

Here we define a class named DiscountStrategy.

Inside it, we define a method named apply.

The class stores a value inside a variable named rate.

The apply method returns a modified value based on the rate.

Because the class defines a method named apply, any object that implements apply with the same structure can be injected and used as a strategy.

# Final Takeaway

### CONCEPT 7.1

Special methods act as the interface between your objects and Python syntax.
