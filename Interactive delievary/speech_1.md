The Python Data Model, also called magic methods or dunder methods, defines how objects interact with Python syntax. It is the underlying protocol that connects your objects to built in operations like len, indexing, iteration, and operators.

Python is best understood as an object centric framework. Its real API is not standalone functions. It is objects that implement specific methods. Syntax works only because objects implement expected behaviors.

By the end of this session, you should be able to mentally translate common Python syntax like len of x, x at index i, or x plus y into the special method calls that Python performs internally.

Python does not introduce new syntax for new features. Instead, it enables syntax participation through method implementation. If an object implements specific special methods, Python allows it to integrate into built in syntax.

The expression len of x works because Python checks whether x implements the special dunder method len and calls it if available.

Indexing syntax like x at index i works because Python translates it into a call to the special dunder method getitem on the object.

A for loop over an object requires the object to implement the special dunder method iter, which provides an iterator.

The membership test using the in keyword works when the object implements the special dunder method contains.

A custom class can behave like a built in container if it implements the special methods that unlock container behavior. Python does not require inheritance from list to enable list like functionality.

The special dunder method init initializes object state. Here, an internal attribute named data stores the actual list. The leading underscore indicates intended internal use.

The special dunder method len enables len syntax on a custom object by returning the container size.

The special dunder method getitem enables square bracket indexing, allowing access using an index value.

The special dunder method contains enables membership testing using the in keyword.

The special dunder method iter enables iteration by returning an iterator over the internal data.

Python evaluates objects based on behavior, not type. If an object implements required methods, it is accepted. The constructor argument can be any type that supports expected operations.

Operators in Python are translated into method calls. An expression like x plus y becomes a call to the special dunder method add on x with y as the argument.

Indexing and built in functions are also method based transformations. Accessing an index maps to the special dunder method getitem, and calling len maps to the special dunder method len.

Operator overloading allows redefining behavior of operators, but it must align with user expectations to maintain clarity and correctness.

Duck typing means behavior determines compatibility. If an object implements required methods, it can be used in that context without explicit type checks.

A function like analyze depends only on behaviors such as iterability, length support, and string like elements.

Calling len on a collection depends on the special dunder method len, and calling split on each line assumes that each element behaves like a string.

Duck typing avoids inheritance hierarchies and type checks. It relies purely on method presence and consistent behavior.

An object can provide indexing behavior without storing all data in memory by implementing the special dunder method getitem dynamically.

A lazy file loader can store only a filename and load data on demand during indexing.

Raising an index error is required for invalid indices because Python expects this exception to signal failed indexing.

This design enables lazy evaluation and constant memory usage while maintaining familiar list like syntax.

The strategy plug in pattern uses interchangeable objects that expose the same method, allowing flexible behavior without conditionals.

A method named apply acts as a behavioral contract. Any object that defines a method named apply taking self and price can function as a discount strategy.

This design relies on duck typing. No inheritance or base class is required. Only consistent method naming and behavior are needed.

Special methods form the interface between user defined objects and Python syntax. They are foundational, not advanced.

Understanding the data model makes Python predictable. Syntax becomes readable as method calls, improving design clarity and expressiveness.
