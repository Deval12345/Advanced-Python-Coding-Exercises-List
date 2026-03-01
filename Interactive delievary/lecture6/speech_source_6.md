# Decorators and Closures in Python

## Lesson Opening

In the previous session we saw that functions are first-class objects. We passed them as arguments and stored them in dictionaries to select behavior at runtime. The next step is to wrap a function in another function so that every call runs extra logic before or after the original. That is what decorators do.

Today we look at decorators and the closure mechanism that gives them private state. We will see how to add logging, timing, or checks without editing the original function.

By the end of this lesson you will understand what a decorator is, how closures capture variables, and how the at-sign syntax applies a decorator to a function.

------------------------------------------------------------------------

## Section 1 — What a Decorator Is

### CONCEPT 0.1

We passed functions into other functions. A decorator is a function that takes a function and returns a new function. The new function usually calls the original but adds behavior around it. The original function is not modified. Behavior is injected from the outside.

### CONCEPT 1.1

A decorator is a callable that accepts a function and returns a replacement. That replacement is often an inner function that runs some code, then calls the original, then runs more code. So the caller still calls one function; they do not see the wrapper. But the wrapper controls what happens every time.

### EXAMPLE 1.1

Here we define a function named logger that takes a parameter named func. Inside logger we define an inner function named wrapper that accepts arbitrary arguments. The wrapper prints a message with the function name, then calls func with those arguments and returns the result. Logger returns wrapper. So logger is a decorator. When we apply it to a function named add, the name add now refers to the wrapper. Calling add runs the print and then the original add. The original logic is unchanged; logging was added externally.

------------------------------------------------------------------------

## Section 2 — The at-sign Syntax

### CONCEPT 2.1

Writing add equals logger of add is correct but verbose. Python lets you write at-sign logger above the function definition. That means the function is passed to logger and the result replaces the original name. So at-sign logger def add is the same as defining add and then reassigning add to logger of add. It is syntactic sugar for applying a decorator at definition time.

------------------------------------------------------------------------

## Section 3 — Closures and Persistent State

### CONCEPT 3.1

The inner function in a decorator can use variables from the outer scope. When the outer function returns, those variables are kept alive because the inner function still references them. That is a closure. So a decorator can have private state that persists across calls, without global variables.

### EXAMPLE 3.1

Here we define a function named limitCalls that takes a parameter named maxCalls. Inside it we define a decorator that has a variable named count. The inner wrapper increments count each time it is called and raises an error if count exceeds maxCalls. Otherwise it calls the original function. Because count is in the closure, it persists across calls. So we can limit how many times a function is invoked without storing state in the function itself or in a global. The state is private to the decorator.

------------------------------------------------------------------------

## Section 4 — Decorators That Take Arguments

### CONCEPT 4.1

Sometimes you want a decorator that takes arguments, like at-sign limitCalls with maxCalls equals three. Then the decorator is not the outer function itself; the outer function takes the arguments and returns the real decorator. So you have a function that returns a decorator that returns a wrapper. The at-sign syntax then applies the result of calling the outer function with the arguments.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 5.1

A decorator is a function that takes a function and returns a new function, usually a wrapper that calls the original. The at-sign syntax applies the decorator at definition time. Closures let the wrapper keep private state across calls. Decorators are used for logging, timing, caching, and access control. They keep core logic clean and add behavior in one place.
