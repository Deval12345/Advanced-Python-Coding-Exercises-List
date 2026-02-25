Slide 1
Title: Python Data Model as Interface

Point 1: Python Data Model consists of dunder methods like "__len__" and "__getitem__"
Point 2: Objects plug into Python syntax through these methods
Point 3: Syntax can be mentally translated into method calls

Slide 2
Title: Length via "__len__"

Point 1: Calling "len(numbers)" invokes "__len__"
Point 2: Example: numbers = [1, 2, 3]
Point 3: Example: len(numbers)

Slide 3
Title: Indexing via "__getitem__"

Point 1: Indexing at position i invokes "__getitem__"
Point 2: Example: numbers = [10, 20, 30]
Point 3: Example: value = numbers[1]

Slide 4
Title: Iteration and Membership

Point 1: For loop invokes "__iter__"
Point 2: Membership check invokes "__contains__"
Point 3: Behavior enables natural syntax

Slide 5
Title: Custom Objects with Dunder Methods

Point 1: Implement "__len__" and "__getitem__" to mimic list behavior
Point 2: Example: custom = CustomList([5, 6, 7])
Point 3: Example: len(custom), custom[1]

Slide 6
Title: Operators as Methods

Point 1: Addition invokes "__add__"
Point 2: Example: result = a + b
Point 3: Operators are syntax over method calls

Slide 7
Title: Duck Typing

Point 1: Functions rely on behavior, not explicit type
Point 2: Iteration and addition are sufficient
Point 3: Compatibility comes from capability

Slide 8
Title: Lazy Indexing

Point 1: Implement "__getitem__" for on demand loading
Point 2: Example: loader = LazyFileLoader("log.txt")
Point 3: Example: line = loader[5]

Slide 9
Title: Interchangeable Behavior

Point 1: Shared method like "apply" defines contract
Point 2: Example: discount.apply(100)
Point 3: Polymorphism through behavior

Slide 10
Title: Why It Matters

Point 1: Dunder methods power Python syntax
Point 2: They connect objects to language features
Point 3: Understanding them improves design clarity
