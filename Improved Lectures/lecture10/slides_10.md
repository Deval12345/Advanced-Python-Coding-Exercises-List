# Slides — Lecture 10: Decorators — Behavior Transformation Without Code Modification

---

## Slide 1
**Title:** From Closures to Decorators

- L6 established functions as first-class objects
- L7 showed closures capturing and preserving state
- A decorator is a function that receives a function and returns a modified version of it
- The @ syntax is shorthand for: originalFunction = myDecorator(originalFunction)

---

## Slide 2
**Title:** The Cross-Cutting Concern Problem

- Large systems have concerns that appear across hundreds of functions: logging, timing, auth, retry
- Handling these inside each function duplicates code and violates single responsibility
- When retry logic changes, you update it in every function that uses it
- Decorators extract each concern into one reusable, independently testable layer

---

## Slide 3
**Title:** How a Decorator Works — Three Steps

- Step 1: Receive a function as an argument
- Step 2: Define a wrapper function that calls the original, adding behavior before or after
- Step 3: Return the wrapper function
- The original function is captured in the wrapper's closure and still exists unchanged

---

## Slide 4
**Title:** Example 10.1 — logCall: Basic Decorator

- logCall receives targetFunction
- wrapper prints a log message using targetFunction's name
- wrapper calls targetFunction with *args and **kwargs and returns the result
- logCall returns wrapper
- @logCall above processPayment: every call logs automatically, processPayment is untouched

---

## Slide 5
**Title:** The @ Syntax and Decorator Stacking

- @decoratorName above a def is syntactic sugar applied at module load time
- Python transforms the function once before any call happens
- Multiple decorators stack: applied bottom-to-top at definition time
- At call time, the outermost decorator runs first

---

## Slide 6
**Title:** functools.wraps — Preserving Function Identity

- Naive wrapping replaces the original function's name and docstring with "wrapper"
- Debugging, stack traces, and introspection tools all show incorrect names
- functools.wraps copies name, docstring, module, and annotations from original to wrapper
- Always use functools.wraps in production decorators

---

## Slide 7
**Title:** Example 10.2 — measureTime with functools.wraps

- measureTime receives targetFunction
- @functools.wraps(targetFunction) is applied to wrapper
- wrapper records start time, calls targetFunction, records end time, prints elapsed, returns result
- The wrapper's __name__ equals targetFunction's __name__
- Introspection and debuggers see the correct function name

---

## Slide 8
**Title:** Decorators with Arguments — One More Level

- Some decorators need configuration: maxAttempts, callLimit, timeToLive
- Outer function accepts configuration and returns a decorator
- Decorator accepts the function and returns a wrapper
- @ with parentheses calls the outer function first, receives a decorator, applies it

---

## Slide 9
**Title:** Example 10.3 — retryOnFailure with maxAttempts

- retryOnFailure(maxAttempts) returns decorator
- decorator(targetFunction) returns wrapper
- wrapper loops up to maxAttempts times: tries calling targetFunction, returns on success
- After all attempts exhausted, raises the last exception
- One implementation handles retry logic for every function in the system

---

## Slide 10
**Title:** Three Structural Patterns for Decorators

- Pre-condition: run setup before the function (auth check, input validation)
- Post-condition: run cleanup after the function (log result, commit transaction)
- Around: wrap the entire call (timing, cache lookup, retry)
- All three use the same wrapper structure — placement of the targetFunction call is the only difference

---

## Slide 11
**Title:** Decorators in Industry

- Django @login_required: authentication without touching view logic
- Flask @app.route: registers a function as a URL handler at import time
- Python @property: turns a method into an attribute
- Tenacity @retry: adds exponential backoff to any function
- Prometheus/StatsD @timer: performance monitoring across entire services

---

## Slide 12
**Title:** Summary

- Decorators transform function behavior without modifying function code
- @ syntax applies the transformation at definition time, making intent visible
- functools.wraps preserves function identity for correct tooling behavior
- Configurable decorators add one outer wrapping level for parameters
- Next lecture: context managers — structural thinking applied to resource cleanup

