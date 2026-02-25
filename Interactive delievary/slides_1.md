Slide 1
Title: Python as a Framework

Point 1: Python enables syntax participation through special methods.

Point 2: "len" of "object_instance" maps to "object_instance.len"

Point 3: A for loop over an object maps to "iter"

Point 4: The "in" keyword maps to "contains"

Slide 2
Title: -
Point 1: If a class implements "len", "getitem", "contains", and "iter", the object can behave like a native container.

Slide 3
Title: -
Point 1: Operators map to special dunder methods. For example, addition maps to "add".

Slide 4
Title: -
Point 1: Behavior matters more than type. Objects are accepted based on the methods they implement, not based on inheritance.

Slide 5
Title: -
Point 1: Indexing is enabled through "getitem", allowing objects to compute values lazily when accessed by index.

Slide 6
Title: -
Point 1: Objects that share the same method signature can act as interchangeable strategies.

Slide 7
Title: -
Point 1: Special methods act as the interface between your objects and Python syntax.
