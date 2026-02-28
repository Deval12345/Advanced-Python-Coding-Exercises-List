# Speech Source — Lecture 10: Decorators — Behavior Transformation Without Code Modification

---

## CONCEPT 0.1 — Transition from Previous Lectures

In the previous two lectures, we saw that functions can be stored in variables, passed as arguments, and returned from other functions. We saw that closures capture context from their enclosing scope and keep it alive. Now we put these two ideas together. A decorator is a function that takes another function as input and returns a modified version of it. That is the whole thing. The at-sign syntax is just a shorthand for what you already know how to do.

---

## CONCEPT 1.1 — The Cross-Cutting Concern Problem

Every production system has concerns that cut across many functions. Logging, timing, authentication, validation, and retry logic all appear in dozens or hundreds of places. If you handle these inside each function, you violate the single responsibility principle. Your payment function both processes payments and logs and retries. Your search function both searches and authenticates and times itself. When the retry logic changes, you modify every function that uses it.

Decorators solve this by making cross-cutting concerns into composable layers that wrap around functions. The payment function handles payments. A retry decorator handles retrying. A logging decorator handles logging. Each layer is independent, testable, and reusable across the entire codebase.

---

## CONCEPT 1.2 — Decorators as Closures Applied to Functions

A decorator is a function that follows three steps. Step one: it receives a function as its argument. Step two: it defines a new wrapper function that calls the original function, adding behavior before or after. Step three: it returns the wrapper function.

When you write the at-sign followed by a decorator name above a function definition, Python executes the following: the original function's name is rebound to the result of calling the decorator with the original function. The decorated function's name now points to the wrapper. The original function still exists, captured inside the closure.

---

## EXAMPLE 1.1 — Basic logCall Decorator

Here we define a function named logCall that receives a parameter named targetFunction. Inside, we define a function named wrapper that accepts star-args and double-star-kwargs. The wrapper prints a message saying targetFunction's name is being called. Then it calls targetFunction with the provided args and kwargs and returns the result. The logCall function returns wrapper.

When we apply logCall above a function named processPayment using the at-sign syntax, every call to processPayment now automatically logs a message. The processPayment function itself is unchanged. The logging concern lives entirely inside logCall.

---

## CONCEPT 2.1 — The @ Syntax and Decorator Stacking

The at-sign before a function definition is syntactic sugar. When Python sees the at-sign followed by a decorator name above a def statement, it automatically applies that decorator to the function being defined. This happens at module load time, not at call time. The function definition is transformed once, before any code calls it.

Stacking multiple decorators is possible. When you write two at-sign decorators above a function, they are applied from bottom to top. The decorator closest to the function definition is applied first, producing a wrapped function. The next decorator above it wraps that result. The outermost decorator runs first when the function is called.

---

## CONCEPT 2.2 — functools.wraps: Preserving Function Identity

When we wrap a function, the wrapper's name and docstring replace the original function's metadata. Debugging becomes confusing because every decorated function appears to have the name wrapper. Introspection tools, logging frameworks, and stack traces all show the wrong name.

The functools.wraps decorator solves this. It copies the original function's name, docstring, module, and annotations onto the wrapper function. The wrapper behaves like the original as far as all inspection tools are concerned. You should always use functools.wraps in any decorator you write for production use.

---

## EXAMPLE 2.1 — measureTime with functools.wraps

Here we define a function named measureTime that receives a parameter named targetFunction. Inside, we apply functools.wraps to wrapper using targetFunction as the argument. The wrapper records the start time using time.perf_counter, calls targetFunction with all args and kwargs, records the end time, computes the elapsed time, prints it with targetFunction's name, and returns the result.

Because we used functools.wraps, the wrapper's name attribute matches targetFunction's name. Introspection tools and debuggers see the correct function name. Stack traces reference the decorated function correctly. The timing concern is fully isolated in measureTime.

---

## CONCEPT 3.1 — Decorators with Arguments: One More Level of Wrapping

Sometimes you need to configure a decorator. A retry decorator needs to know how many retries to allow. A rate limiter needs to know the maximum calls per minute. A cache decorator might need a time-to-live parameter. This requires one additional level of wrapping: a function that accepts the configuration and returns a decorator.

The outer function receives the configuration parameters and returns a decorator. The decorator receives the function and returns a wrapper. The wrapper does the actual work. When you use the at-sign syntax with parentheses, Python calls the outer function first, gets a decorator back, and applies that decorator to the function below.

---

## EXAMPLE 3.1 — retryOnFailure with Configurable maxAttempts

Here we define a function named retryOnFailure that receives a parameter named maxAttempts. Inside, we define a function named decorator that receives a parameter named targetFunction. Inside decorator, we apply functools.wraps and define a wrapper that loops from zero up to maxAttempts. In each iteration, it tries to call targetFunction with all args and kwargs. If the call succeeds, it returns the result immediately. If it raises an exception, it stores the exception. After all attempts are exhausted, it raises the last exception.

The retryOnFailure function returns decorator. When applied with the at-sign syntax and parentheses specifying maxAttempts, the result is a decorator that retries the wrapped function up to that many times before giving up. The retry logic lives in one place. Every function that needs retry behavior uses the same implementation.

---

## CONCEPT 4.1 — Practical Decorator Patterns in Industry

Three major structural patterns cover nearly all decorator use cases.

The pre-condition pattern: run setup logic before calling the function. Authentication checks, input validation, and connection establishment are examples. The wrapper checks a condition first and either raises an error or calls the original function.

The post-condition pattern: run cleanup logic after the function returns. Logging the result, committing a database transaction, and releasing resources are examples. The wrapper calls the original function, then performs the cleanup regardless of whether the function succeeded.

The around pattern: wrap the function entirely in a try-except or time measurement. Timing decorators, caching decorators that return early on a cache hit, and retry decorators that call the function multiple times are all around-pattern decorators.

All three use the same wrapper structure. The difference is only where you place the call to targetFunction relative to the surrounding logic.

In industry, Django's login-required decorator adds authentication without touching view logic. Flask's app.route decorator registers a function as a URL handler. Python's built-in property decorator turns a method into an attribute. The Tenacity library's retry decorator adds exponential backoff to any function. Prometheus and StatsD use timer decorators for performance monitoring at scale.

---

## CONCEPT 5.1 — Final Takeaway

Decorators transform function behavior without modifying function code. The at-sign syntax applies the transformation at definition time, making the intent visible at the point where the function is defined. functools.wraps preserves function identity so debugging and introspection work correctly. Decorators with arguments add one more level of wrapping to accept configuration. In production systems, decorators enforce cross-cutting concerns — authentication, logging, retrying, timing — across many functions from a single implementation.

In the next lecture, we will explore context managers, which apply similar structural thinking to the problem of guaranteed resource cleanup.

