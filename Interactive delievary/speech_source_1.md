## Lesson Opening

### CONCEPT 1.1
The Python Data Model, also called magic methods or dunder methods, defines how objects interact with Python syntax. It is the underlying protocol that connects your objects to built in operations like len, indexing, iteration, and operators.

### CONCEPT 1.2
Python is best understood as an object centric framework. Its real API is not standalone functions. It is objects that implement specific methods. Syntax works only because objects implement expected behaviors.

---

## Learning Goals Spoken

### CONCEPT 1.3
By the end of this session, you should be able to mentally translate common Python syntax like len of x, x at index i, or x plus y into the special method calls that Python performs internally.

---

## 1. Python as a Framework

### CONCEPT 2.1
Python does not introduce new syntax for new features. Instead, it enables syntax participation through method implementation. If an object implements specific special methods, Python allows it to integrate into built in syntax.

### CONCEPT 2.2
The expression len of x works because Python checks whether x implements the special dunder method len and calls it if available.

### EXAMPLE 2.2
When you call len on an object, Python internally calls the special dunder method len on that object.

### CONCEPT 2.3
Indexing syntax like x at index i works because Python translates it into a call to the special dunder method getitem on the object.

### EXAMPLE 2.3
When you access an object at position two, Python internally calls the special dunder method getitem with the index value.

### CONCEPT 2.4
A for loop over an object requires the object to implement the special dunder method iter, which provides an iterator.

### EXAMPLE 2.4
When Python executes a for loop over an object, it first calls the special dunder method iter on that object.

### CONCEPT 2.5
The membership test using the in keyword works when the object implements the special dunder method contains.

### EXAMPLE 2.5
When you check whether a value exists inside an object using in, Python calls the special dunder method contains internally.

---

## 2. Exercise — Build a Native Like Container

### CONCEPT 3.1
A custom class can behave like a built in container if it implements the special methods that unlock container behavior. Python does not require inheritance from list to enable list like functionality.

### CONCEPT 3.2
The special dunder method init initializes object state. Here, an internal attribute named data stores the actual list. The leading underscore indicates intended internal use.

### EXAMPLE 3.2
An internal attribute like a variable named data separates implementation details from the external interface of the object.

### CONCEPT 3.3
The special dunder method len enables len syntax on a custom object by returning the container size.

### EXAMPLE 3.3
If the special dunder method len returns five, then calling len on the object evaluates to five.

### CONCEPT 3.4
The special dunder method getitem enables square bracket indexing, allowing access using an index value.

### EXAMPLE 3.4
If the special dunder method getitem returns the value A for index two, then accessing the object at index two produces A.

### CONCEPT 3.5
The special dunder method contains enables membership testing using the in keyword.

### EXAMPLE 3.5
If the special dunder method contains returns true for a value, then using the in keyword with that value evaluates to true.

### CONCEPT 3.6
The special dunder method iter enables iteration by returning an iterator over the internal data.

### EXAMPLE 3.6
If the special dunder method iter returns an iterator over the internal data, the object works naturally inside a for loop.

### CONCEPT 3.7
Python evaluates objects based on behavior, not type. If an object implements required methods, it is accepted. The constructor argument can be any type that supports expected operations.

### EXAMPLE 3.7
A file object, a generator, or an API stream can be stored internally if it supports iteration.

---

## 3. Special Methods as Syntax Glue

### CONCEPT 4.1
Operators in Python are translated into method calls. An expression like x plus y becomes a call to the special dunder method add on x with y as the argument.

### EXAMPLE 4.1
If the special dunder method add returns a new object, then the expression x plus y evaluates to that new object.

### CONCEPT 4.2
Indexing and built in functions are also method based transformations. Accessing an index maps to the special dunder method getitem, and calling len maps to the special dunder method len.

### CONCEPT 4.3
Operator overloading allows redefining behavior of operators, but it must align with user expectations to maintain clarity and correctness.

---

## 4. Duck Typing

### CONCEPT 5.1
Duck typing means behavior determines compatibility. If an object implements required methods, it can be used in that context without explicit type checks.

### CONCEPT 5.2
A function like analyze depends only on behaviors such as iterability, length support, and string like elements.

### EXAMPLE 5.2
If a source object implements the special dunder method iter, then converting that source into a list works naturally.

### CONCEPT 5.3
Calling len on a collection depends on the special dunder method len, and calling split on each line assumes that each element behaves like a string.

### EXAMPLE 5.3
A generator that yields strings works because it satisfies both iteration and string behavior expectations.

### CONCEPT 5.4
Duck typing avoids inheritance hierarchies and type checks. It relies purely on method presence and consistent behavior.

---

## 5. Lazy Indexable Objects

### CONCEPT 6.1
An object can provide indexing behavior without storing all data in memory by implementing the special dunder method getitem dynamically.

### CONCEPT 6.2
A lazy file loader can store only a filename and load data on demand during indexing.

### EXAMPLE 6.2
When you access the loader at a specific index, it opens the file, reads until that line number, and returns the corresponding line.

### CONCEPT 6.3
Raising an index error is required for invalid indices because Python expects this exception to signal failed indexing.

### EXAMPLE 6.3
If no matching line is found, raising an index error ensures consistent container behavior.

### CONCEPT 6.4
This design enables lazy evaluation and constant memory usage while maintaining familiar list like syntax.

---

## 6. Strategy Plug in Pattern

### CONCEPT 7.1
The strategy plug in pattern uses interchangeable objects that expose the same method, allowing flexible behavior without conditionals.

### CONCEPT 7.2
A method named apply acts as a behavioral contract. Any object that defines a method named apply taking self and price can function as a discount strategy.

### EXAMPLE 7.2
An object whose apply method returns price multiplied by zero point nine can be used wherever a discount strategy is expected.

### CONCEPT 7.3
This design relies on duck typing. No inheritance or base class is required. Only consistent method naming and behavior are needed.

---

## Final Takeaways

### CONCEPT 8.1
Special methods form the interface between user defined objects and Python syntax. They are foundational, not advanced.

### CONCEPT 8.2
Understanding the data model makes Python predictable. Syntax becomes readable as method calls, improving design clarity and expressiveness.
