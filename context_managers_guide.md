
# Context Managers & Structural Cleanup in Python

## Overview

Context managers exist to guarantee cleanup across:

- Exceptions
- Early returns
- Complex control flow
- Partial failures

They are not utilities — they are part of Python’s execution protocol.

The `with` statement ensures structural cleanup, not optional cleanup.

You will learn:

✔ How `with` actually works  
✔ __enter__ / __exit__ protocol  
✔ Exception handling behavior  
✔ Guaranteed cleanup  
✔ Custom context managers  
✔ Real-world resource safety  

---

## Section 1 — Why Context Managers Exist

Manual cleanup is fragile.

Example without context manager:

```python
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()
```

Every exit path must remember cleanup.

Missing one → leaks resources.

Context managers make cleanup structural.

---

## Section 2 — How `with` Works Internally

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

`__exit__` always runs.

It receives exception details and can choose to swallow them.

---

## Exercise 1 — Timing Context Manager

Goal: Measure execution time and log even on failure.

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc, tb):
        end = time.time()
        print("Elapsed:", end - self.start)
        return False  # do not swallow exceptions


with Timer():
    time.sleep(1)
```

If an error happens, timing still prints.

Cleanup is guaranteed.

---

## Section 3 — Resource Leak Problem

Opening many files without closing them leaks descriptors.

### Bad Design

```python
for name in files:
    f = open(name)
    process(f)
    # forgot f.close()
```

Memory and file handles grow.

### Correct Design

```python
for name in files:
    with open(name) as f:
        process(f)
```

Cleanup happens automatically.

---

## Exercise 2 — Bank Import Context Manager

Goal:

Skip minor row errors  
Fail fast on system errors  
Always clean resources

```python
class ImportSession:
    def __enter__(self):
        print("Starting import")
        return self

    def __exit__(self, exc_type, exc, tb):
        print("Cleaning up session")
        if exc_type is ValueError:
            print("Skipping bad row")
            return True  # swallow minor errors
        return False  # propagate system errors


def process_row(row):
    if "BAD" in row:
        raise ValueError("Formatting issue")
    print("Processed:", row)


with ImportSession():
    rows = ["OK1", "BAD_ROW", "OK2"]
    for r in rows:
        process_row(r)
```

Centralized cleanup and policy.

No duplicated try/except everywhere.

---

## Real-World Uses

Context managers power:

✔ Database transactions (commit/rollback)  
✔ Locks and concurrency control  
✔ File and socket handling  
✔ Cloud SDK sessions  
✔ Temporary environment switches  
✔ Test isolation fixtures  

---

## Final Takeaways

Context managers:

✔ Guarantee cleanup  
✔ Centralize error policy  
✔ Protect resources  
✔ Simplify control flow  
✔ Prevent hidden leaks  

They make cleanup structural, not optional.

---

## Suggested Extensions

1. Build a transaction context manager
2. Create a temporary config override manager
3. Implement retry context manager
4. Add logging-on-exit manager
5. Combine with decorators

---

End of module.
