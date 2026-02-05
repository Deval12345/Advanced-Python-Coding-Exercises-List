
# Fault Tolerance & Retry Architecture — Designing Systems That Survive Failure

Real systems do not fail occasionally.

They fail constantly.

Networks drop packets.
Databases restart.
Workers crash.
Machines disappear.

High-performance architecture assumes failure is normal.

This guide explains how to design systems that continue operating
even when components fail.

---

## Real-World Scenario: Payment Processing Pipeline

A payment service processes thousands of transactions per second.

Steps:

1. validate payment
2. charge bank API
3. update ledger
4. notify client

If step 2 fails due to network error:

❌ transaction lost  
❌ money inconsistent  
❌ audit corruption  

A fragile system collapses.

A resilient system retries safely.

---

# Part 1 — Retry with Backoff

Naive retry floods the system.

Professional retry uses exponential backoff.

```python
import time
import random

def unreliable_call():
    if random.random() < 0.7:
        raise RuntimeError("Network failure")
    return "success"

def retry(fn, attempts=5):
    delay = 0.1
    for i in range(attempts):
        try:
            return fn()
        except Exception as e:
            print("Retrying:", e)
            time.sleep(delay)
            delay *= 2
    raise RuntimeError("Permanent failure")

print(retry(unreliable_call))
```

Backoff prevents overload cascades.

---

# Part 2 — Idempotent Operations

Retries must be safe.

If the same task runs twice:

result must remain correct.

Architecture rule:

> operations must be idempotent

Example:

charging a user twice is catastrophic.

Instead:

use transaction IDs.

---

## Idempotent Ledger Update

```python
processed = set()

def process_payment(tx_id, amount):
    if tx_id in processed:
        return "already processed"
    processed.add(tx_id)
    return f"charged {amount}"
```

Duplicate calls do not corrupt state.

---

# Part 3 — Dead Letter Queues

Some tasks fail permanently.

Do not block the pipeline.

Move them aside.

```
main queue → worker → dead letter queue
```

Failed tasks are inspected later.

System continues running.

---

## Dead Letter Pattern

```python
main = []
dead = []

def worker(task):
    try:
        if task < 0:
            raise ValueError("invalid")
        return task * 2
    except Exception:
        dead.append(task)

for t in [1, 2, -3, 4]:
    worker(t)

print("Dead:", dead)
```

Failure is isolated.

Pipeline survives.

---

# Part 4 — Circuit Breaker Pattern

When an external dependency fails repeatedly:

stop calling it temporarily.

This prevents cascading collapse.

```python
failures = 0
threshold = 3

def guarded_call():
    global failures
    if failures >= threshold:
        print("Circuit open")
        return None
    try:
        return unreliable_call()
    except:
        failures += 1
```

System protects itself.

---

# Final Insight

Resilient architecture assumes:

failure is inevitable.

Systems survive by:

retrying safely  
isolating damage  
avoiding duplication  
protecting dependencies  

Correctness under failure is the true performance metric.
