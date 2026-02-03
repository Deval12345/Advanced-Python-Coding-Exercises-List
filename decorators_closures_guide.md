
# Decorators & Closures in Python

## Overview

Decorators exist to handle cross-cutting concerns without modifying core business logic.

They allow behavior transformation that is orthogonal to the function’s main purpose.

You will learn:

✔ What decorators really are (functions wrapping functions)  
✔ Closures and retained state  
✔ Non-invasive behavior injection  
✔ Logging and instrumentation  
✔ Feature flags and runtime control  
✔ Persistent state using closures  

---

## Section 1 — What is a Decorator?

A decorator is a function that takes a function and returns a new function.

### Example

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

The original function is unchanged — behavior is injected externally.

---

## Exercise 1 — Add Logging Without Modifying Code

Goal: Add logging to multiple functions without editing them.

```python
@logger
def multiply(a, b):
    return a * b

@logger
def subtract(a, b):
    return a - b

print(multiply(3, 4))
print(subtract(10, 5))
```

Lesson:

Cross-cutting behavior should not pollute business logic.

---

## Section 2 — Closures and Persistent State

Closures allow inner functions to retain variables from outer scope.

This enables private state.

### Problem 1 — Count API Calls

```python
def call_counter(limit):
    def decorator(func):
        count = 0

        def wrapper(*args, **kwargs):
            nonlocal count
            count += 1
            if count > limit:
                raise RuntimeError("API limit exceeded")
            return func(*args, **kwargs)

        return wrapper
    return decorator


@call_counter(limit=3)
def api_handler():
    print("API called")

for _ in range(5):
    try:
        api_handler()
    except RuntimeError as e:
        print(e)
```

The counter persists across calls without global variables.

This is closure state.

---

## Section 3 — Feature Flags

Enable/disable behavior at runtime.

### Problem 2 — Feature Flag Decorator

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

Behavior changes without touching checkout logic.

---

## Real-World Uses

Decorators power:

✔ Authentication layers  
✔ Logging middleware  
✔ Caching systems  
✔ Rate limiting  
✔ Framework internals (Flask/FastAPI style routing)  

---

## Final Takeaways

Decorators:

✔ Inject behavior cleanly  
✔ Preserve core logic  
✔ Capture private state via closures  
✔ Enable runtime extensibility  

They replace invasive conditionals with composable behavior.

---

## Suggested Extensions

1. Build a caching decorator
2. Implement timing/profiling decorator
3. Create authorization decorator
4. Add retry logic decorator
5. Combine multiple decorators

---

End of module.
