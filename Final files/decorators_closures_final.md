
# Decorators and Closures

## Overview

Decorators exist to handle cross-cutting concerns without modifying core logic.

They transform behavior while keeping business code untouched.

In Fluent Python terms:
decorators are behavior transformation tools orthogonal to application logic.

This module focuses on system design, not syntax tricks.

---

## Section 1 — What a Decorator Really Is

A decorator is a function that takes a function and returns a new function.

It wraps behavior without editing original code.

```python
def logger(func):
    def wrapper(*args, **kwargs):
        print("Calling:", func.__name__)
        return func(*args, **kwargs)
    return wrapper


@logger
def add(a, b):
    return a + b

print(add(2, 3))
```

Core logic is unchanged.
Behavior is injected externally.

---

## Exercise 1 — Add Logging Everywhere

Decorate multiple functions without touching their implementation.

```python
@logger
def multiply(a, b):
    return a * b

@logger
def subtract(a, b):
    return a - b
```

This demonstrates non-invasive behavior injection.

---

## Section 2 — Closures and Retained State

Closures allow functions to retain private state.

This avoids globals and classes.

```python
def call_counter(limit):
    def decorator(func):
        count = 0

        def wrapper(*args, **kwargs):
            nonlocal count
            count += 1
            if count > limit:
                raise RuntimeError("Limit exceeded")
            return func(*args, **kwargs)

        return wrapper
    return decorator


@call_counter(3)
def api():
    print("API call")

for _ in range(5):
    try:
        api()
    except RuntimeError as e:
        print(e)
```

State persists safely via closure.

---

## Section 3 — Behavior Control via Decorators

Decorators enable runtime feature control.

```python
def feature_enabled(flag):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if not flag():
                print("Feature disabled")
                return None
            return func(*args, **kwargs)
        return wrapper
    return decorator


enabled = True

def flag():
    return enabled


@feature_enabled(flag)
def checkout():
    print("Checkout executed")

checkout()
enabled = False
checkout()
```

Behavior changes without editing checkout logic.

---

## Section 4 — Timing & Instrumentation

Decorators isolate instrumentation from business code.

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print("Time:", time.time() - start)
        return result
    return wrapper
```

This pattern scales to profiling, tracing, and monitoring.

---

## Real-World Uses

Decorators power:

- logging middleware
- authentication layers
- caching systems
- retry logic
- rate limiting
- framework routing
- instrumentation

They replace invasive conditionals with composable behavior.

---

## Final Takeaways

Decorators:

- inject behavior cleanly
- preserve core logic
- capture state via closures
- enable extensibility
- isolate cross-cutting concerns

They are architectural tools, not syntactic sugar.

