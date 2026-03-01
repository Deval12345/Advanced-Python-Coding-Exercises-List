Slide 0
Title: Previous Lecture Takeaway
Point 1: Descriptors attach reusable behavior to attributes; today we treat behavior itself as data by using functions as first-class objects.

Slide 1
Title: Functions Are Values
Point 1: In Python a function is an object; you can assign it, store it in structures, and pass it; that is first-class functions.
Point 2: Functions behave like any other value; no special syntax; this enables strategies, callbacks, and plugins.

Slide 2
Title: Passing Functions as Arguments
Point 1: Passing a function as an argument lets the caller inject behavior without changing the receiver; the receiver just calls what it was given.

Slide 3
Title: Runtime Strategy Selection
Point 1: Store functions in a dictionary keyed by strategy name; look up and call at runtime; new strategies are one new function and one dictionary entry.

Slide 4
Title: Callables
Point 1: Callables include functions, methods, classes, and objects implementing *call*; pass any callable when the receiver only needs to call it.

Slide 5
Title: Final Takeaways
Point 1: First-class functions enable assignment, storage, and passing; strategy selection via dictionaries; callables (including *call*) unify functions and callable objects for extensible code.
