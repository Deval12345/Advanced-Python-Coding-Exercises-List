
# Stage 2 — Resource Lifecycle & Context Manager Architecture

In Stage 1 we built a pluggable pipeline engine.

Now we add a new constraint:

The system runs continuously.

It opens files.
Maintains network connections.
Allocates buffers.
Spawns workers.

If resources are not cleaned correctly:

- memory leaks grow
- file handles exhaust
- sockets hang
- workers become zombies
- the system slowly dies

Stage 2 introduces:

structured resource management.

We use:

- context managers
- lifecycle boundaries
- safe acquisition & release
- deterministic cleanup

This mirrors real production services.

---

## Real-World Scenario

Our analytics engine now runs as a live backend service.

Each stage may:

- open a database connection
- write to log files
- maintain model handles
- allocate temporary buffers

If a stage crashes mid-processing:

resources must still be released.

Otherwise the system degrades over hours.

We need architectural cleanup guarantees.

---

# Part 1 — The Resource Problem

Naive resource handling:

```python
f = open("events.log", "w")
f.write("event processed")
# crash happens here
```

File never closes.

Repeat this thousands of times → system failure.

Production systems cannot rely on “best effort cleanup”.

They need guaranteed cleanup.

---

# Part 2 — Context Managers as Architecture

Context managers define:

```
acquire → use → release
```

They make lifecycle explicit.

---

## Simple Context Manager

```python
class FileWriter:
    def __enter__(self):
        self.f = open("events.log", "w")
        return self.f

    def __exit__(self, exc_type, exc, tb):
        self.f.close()
```

Usage:

```python
with FileWriter() as f:
    f.write("processed")
```

Even if an exception occurs:

cleanup runs.

This is deterministic resource safety.

---

# Part 3 — Integrating Context Managers into Pipeline Stages

We extend Stage architecture to support lifecycle hooks.

Each stage can manage its own resources safely.

---

## Stage With Lifecycle

```python
class ManagedStage:
    def __enter__(self):
        print("acquire resources")
        return self

    def __exit__(self, exc_type, exc, tb):
        print("release resources")

    def process(self, event):
        event["managed"] = True
        return event
```

---

## Pipeline With Structured Execution

```python
class ManagedPipeline:
    def __init__(self, stages):
        self.stages = stages

    def run(self, event):
        for stage in self.stages:
            with stage as s:
                event = s.process(event)
        return event
```

Each stage gets a safe lifecycle boundary.

No leaks.

No dangling resources.

---

# Part 4 — Nested Resource Scopes

Real systems manage multiple layers:

```
pipeline lifecycle
stage lifecycle
worker lifecycle
task lifecycle
```

Context managers compose naturally.

They define hierarchical ownership.

This is how large systems avoid chaos.

---

## Nested Context Example

```python
with ManagedStage() as stage:
    for i in range(3):
        print(stage.process({"id": i}))
```

Resources open once.
Used many times.
Closed once.

Efficient and safe.

---

# Part 5 — Context Managers for External Systems

This pattern scales to:

- database sessions
- GPU handles
- network clients
- thread pools
- multiprocessing pools

Example: safe DB session

```python
class DatabaseSession:
    def __enter__(self):
        print("connect db")
        return self

    def __exit__(self, exc_type, exc, tb):
        print("close db")
```

Production systems wrap everything critical in context managers.

---

# Exercises

1. Create a stage that opens a file and logs every event safely.
2. Implement a mock database stage with context cleanup.
3. Add timing measurement inside a context manager.
4. Create a nested pipeline lifecycle manager.

---

# Final Insight

Resource safety is architecture.

Context managers are not syntax sugar.

They define ownership.

Ownership defines system stability.

Without lifecycle discipline,
long-running systems rot.
