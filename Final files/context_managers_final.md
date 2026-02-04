
# Context Managers

## Overview

Python needed a language-level guarantee for resource cleanup that works across:

- exceptions
- early returns
- complex control flow

Context managers are part of Python’s execution protocol, not utilities.

They make cleanup structural, not conventional.

This module focuses on design motivation and system safety.

---

## Section 1 — Why Structural Cleanup Matters

Manual cleanup is fragile.

```python
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()
```

Every exit path must remember cleanup.
One missed branch → leak.

Context managers eliminate this class of bug.

---

## Section 2 — How `with` Works

This:

```python
with obj as x:
    block
```

is equivalent to:

```python
x = obj.__enter__()
try:
    block
except Exception as e:
    obj.__exit__(type(e), e, e.__traceback__)
    raise
else:
    obj.__exit__(None, None, None)
```

Key points:

- `__enter__` prepares the resource
- `__exit__` always runs
- exception details are passed in
- returning True swallows exceptions

---

## Exercise 1 — Timing Context Manager

Goal: measure time and log even on failure.

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc, tb):
        end = time.time()
        print("Elapsed:", end - self.start)
        return False


with Timer():
    time.sleep(1)
```

Timing prints even if an exception occurs.

Cleanup is guaranteed.

---

## Section 3 — Exception Policy Control

Context managers centralize error policy.

```python
class ImportSession:
    def __enter__(self):
        print("Starting import")
        return self

    def __exit__(self, exc_type, exc, tb):
        print("Cleaning session")
        if exc_type is ValueError:
            print("Skipping bad row")
            return True
        return False
```

Swallow minor errors.
Propagate critical ones.

No scattered try/except blocks.

---

## Exercise 2 — Structural Error Handling

Use ImportSession to process rows safely.

---

## Section 4 — Real Resource Safety

Bad design leaks:

```python
for name in files:
    f = open(name)
    process(f)
```

Correct design:

```python
for name in files:
    with open(name) as f:
        process(f)
```

No leak possible.

---

## Final Takeaways

Context managers:

- guarantee cleanup
- centralize policy
- protect resources
- simplify control flow
- prevent hidden leaks

They are execution protocol tools, not convenience helpers.

