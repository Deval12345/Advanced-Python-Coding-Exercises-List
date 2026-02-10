
# Exception Model & EAFP (Easier to Ask Forgiveness than Permission)

## Overview

Python deliberately separates normal execution paths from failure paths
to preserve readability and composability.

Errors are expected outcomes — not rare edge cases.

EAFP means:

> Assume success, handle failure explicitly.

This module explains exceptions as control flow,
not just error reporting.

---

## Section 1 — Why Python Assumes Success

Python’s data model favors optimistic execution.

Main logic should remain linear.
Error handling should live separately.

This keeps the “happy path” readable.

---

## Exercise 1 — Process Unreliable API Responses

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

Valid data continues.
Bad data is isolated cleanly.

---

## Section 2 — Defensive (LBYL) Approach

Naïve defensive code:

```python
def process_request(data):
    if "user_id" in data:
        uid = data["user_id"]
        if uid is not None:
            if isinstance(uid, int):
                return uid
    return None
```

Readable? No.
Scalable? No.

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

Main logic is linear.
Errors are isolated.

---

## Exercise 2 — Rewrite Using EAFP

Convert nested checks into try/except.
Compare readability.

---

## Section 4 — Concurrency Safety

Pre-checks are unsafe in concurrent systems.

```python
if key in cache:
    value = cache[key]
```

Race condition possible.

Correct version:

```python
try:
    value = cache[key]
except KeyError:
    value = None
```

State is validated at moment of use.

No race window.

---

## Real-World Applications

EAFP is critical in:

- retry/backoff systems
- API integrations
- async services
- parsing unreliable data
- distributed pipelines

Production Python expects failure.

It plans for it structurally.

---

## Final Takeaways

EAFP:

- keeps main logic clean
- avoids defensive nesting
- handles concurrency correctly
- separates success from failure
- matches real-world uncertainty

Exceptions are part of normal flow.

