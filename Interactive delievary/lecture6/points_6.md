Slide 0
Title: Previous Lecture Takeaway
Point 1: Functions as first-class objects enable passing and storing behavior; decorators wrap functions to add behavior without changing the original.

Slide 1
Title: What a Decorator Is
Point 1: A decorator takes a function and returns a new function (often a wrapper that calls the original); behavior is injected from the outside.
Point 2: The replacement runs extra code before or after the original; the caller still calls one function.

Slide 2
Title: The at-sign Syntax
Point 1: *@decorator* above a function definition means the function is passed to the decorator and the result replaces the original name; syntactic sugar at definition time.

Slide 3
Title: Closures and Persistent State
Point 1: The inner function can use variables from the outer scope; that state is kept alive (closure) so the decorator can have private state across calls.

Slide 4
Title: Decorators That Take Arguments
Point 1: For *@decorator(args)* the outer function takes arguments and returns the real decorator; so function that returns decorator that returns wrapper.

Slide 5
Title: Final Takeaways
Point 1: Decorators take a function and return a wrapper; *@* syntax applies at definition time; closures give private state; used for logging, timing, caching, access control.
