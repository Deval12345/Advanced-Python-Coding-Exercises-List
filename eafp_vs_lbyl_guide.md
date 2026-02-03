
# EAFP vs LBYL — Errors as Control Flow in Python

## Overview

Python separates normal execution from failure paths.

Errors are expected outcomes — not rare edge cases.

This philosophy improves:

- Readability
- Concurrency safety
- Composability
- Robustness in unreliable systems

Python encourages **EAFP**:

> Easier to Ask Forgiveness than Permission

Instead of **LBYL**:

> Look Before You Leap

This module explains why.

You will learn:

✔ Errors as alternate control flow  
✔ EAFP vs defensive programming  
✔ Readability advantages  
✔ Race condition safety  
✔ Production API handling  
✔ Concurrent system correctness  

---

## Section 1 — Why Python Assumes Success

Python’s data model is built around optimistic execution.

The main logic should remain linear and readable.

Failure handling should live in exception blocks.

This keeps the “happy path” uncluttered.

---

## Exercise 1 — Process External API Responses

Goal: Continue processing valid responses even if some fail.

```python
responses = [
    {"user": 1, "amount": 100},
    {"bad": "data"},
    {"user": 2, "amount": 200},
]

for r in responses:
    try:
        print("Processing:", r["user"], r["amount"])
    except KeyError:
        print("Skipping malformed response:", r)
```

Valid data continues. Bad data is isolated.

No nested conditionals.

---

## Section 2 — Defensive (LBYL) Approach

Naïve version:

```python
def process_request(data):
    if "user_id" in data:
        uid = data["user_id"]
        if uid is not None:
            if isinstance(uid, int):
                return uid
    return None
```

This grows unreadable quickly.

---

## Section 3 — EAFP Approach

Cleaner version:

```python
def process_request(data):
    try:
        uid = int(data["user_id"])
        discount = data.get("discount_code")
        return uid, discount
    except (KeyError, TypeError, ValueError):
        return None
```

Main logic stays linear.

Errors are handled separately.

---

## Exercise 2 — Rewrite Defensive Code Using EAFP

Take nested checks and convert to exception-driven flow.

Observe readability improvement.

---

## Section 4 — Why LBYL Fails in Concurrent Systems

Pre-checks are unsafe under concurrency.

Example race condition:

```python
if key in cache:
    value = cache[key]  # may fail if another worker deletes it
```

Correct approach:

```python
try:
    value = cache[key]
except KeyError:
    value = None
```

State is checked at the moment of use.

No race window.

---

## Concurrent-System Example

Simulated shared dictionary:

```python
import threading

cache = {"x": 1}

def worker():
    try:
        print(cache["x"])
    except KeyError:
        print("Missing")

threading.Thread(target=worker).start()
del cache["x"]
```

Pre-checks would not prevent failure.

Only exception handling reflects reality.

---

## Real-World Applications

EAFP is critical in:

✔ Retry/backoff systems  
✔ Fault-tolerant pipelines  
✔ API integrations  
✔ Async/concurrent services  
✔ Parsing unreliable data  

Production Python expects exceptions.

It does not fear them.

---

## Final Takeaways

EAFP:

✔ Keeps main logic clean  
✔ Avoids defensive nesting  
✔ Handles concurrency correctly  
✔ Separates success from failure paths  
✔ Matches real-world uncertainty  

Exceptions are part of normal control flow.

---

## Suggested Extensions

1. Build retry decorator with exponential backoff
2. Implement resilient file reader
3. Create fault-tolerant API pipeline
4. Add logging wrapper for exception tracking
5. Simulate concurrent race conditions

---

End of module.
