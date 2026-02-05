
# Stage 6 — Fault Tolerance & Resilience Layer

Our pipeline is now fast and memory stable.

But production systems do not fail occasionally.

They fail constantly.

Networks drop.
Databases restart.
Workers crash.
Dependencies timeout.

If the pipeline assumes perfect behavior,
it collapses under real conditions.

Stage 6 introduces resilience architecture:

- safe retry logic
- idempotent processing
- circuit breakers
- failure isolation
- graceful degradation

This stage transforms the pipeline from fragile
to production-ready.

---

## Real-World Scenario

Our analytics backend depends on:

- external scoring service
- payment API
- recommendation model server

During peak traffic:

the scoring service times out.

Without resilience:

❌ events are lost  
❌ queues back up  
❌ system stalls  
❌ data becomes inconsistent  

We must survive failure.

---

# Part 1 — Retry with Exponential Backoff

Naive retry overloads systems.

Professional retry spreads attempts.

---

## Retry Wrapper

```python
import time
import random

def retry(fn, attempts=5):
    delay = 0.1
    for _ in range(attempts):
        try:
            return fn()
        except Exception:
            time.sleep(delay)
            delay *= 2
    raise RuntimeError("permanent failure")
```

Backoff protects dependencies.

---

# Part 2 — Idempotent Stage Design

Retries must not corrupt state.

Same event processed twice must be safe.

---

## Idempotent Stage

```python
processed = set()

class SafeStage:
    def process(self, event):
        eid = event["id"]
        if eid in processed:
            return event
        processed.add(eid)
        event["safe"] = True
        return event
```

Duplicate processing becomes harmless.

Correctness preserved.

---

# Part 3 — Circuit Breaker Protection

If a dependency fails repeatedly,
stop calling it temporarily.

---

## Circuit Breaker Example

```python
failures = 0
threshold = 3

def guarded_call(fn):
    global failures
    if failures >= threshold:
        print("circuit open")
        return None
    try:
        return fn()
    except:
        failures += 1
```

This prevents cascading collapse.

---

# Part 4 — Dead Letter Isolation

Some events will fail permanently.

We isolate them instead of blocking the pipeline.

```python
dead_letter = []

def resilient_process(event):
    try:
        return pipeline.run(event)
    except Exception:
        dead_letter.append(event)
```

System continues.

Failures are inspectable later.

---

# Part 5 — Graceful Degradation

Under overload:

disable non-critical features.

Example:

skip analytics stage if queue grows too large.

Architecture chooses survival over perfection.

---

# Exercises

1. Add retry wrapper around an unstable stage.
2. Simulate dependency failure.
3. Implement dead letter queue.
4. Measure pipeline stability under artificial faults.

---

# Final Insight

Resilience is architecture.

Systems must assume failure.

Correct systems survive chaos.
