
# Caching & Memoization Architecture — Designing Systems That Avoid Repeated Work

Performance is not only about making code faster.

It is about avoiding unnecessary work.

Caching is one of the most powerful performance tools in real systems.

But naive caching causes:

- stale data bugs
- memory explosions
- inconsistency
- hidden correctness failures

This guide explains how to design safe caching architecture.

---

## Real-World Scenario: Recommendation Engine

A recommendation system computes:

- product rankings
- similarity scores
- personalization weights

These computations are expensive.

Users repeatedly request similar queries.

Without caching:

CPU usage explodes.

With naive caching:

data becomes incorrect.

We need structured caching.

---

# Part 1 — Basic Memoization

Memoization stores results of expensive functions.

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive(x):
    print("Computing", x)
    return x * x

print(expensive(5))
print(expensive(5))  # cached
```

Second call is instant.

No recomputation.

---

# Part 2 — Cache Invalidation Problem

Real systems change.

Cached data may become outdated.

Example:

- product price changes
- user profile updates
- model retrains

Cache must expire.

---

## Time-Based Expiration Example

```python
import time

cache = {}
TTL = 2

def cached_compute(x):
    now = time.time()
    if x in cache and now - cache[x][1] < TTL:
        return cache[x][0]

    value = x * x
    cache[x] = (value, now)
    return value
```

Expiration prevents stale results.

---

# Part 3 — Cache Architecture Patterns

Professional systems separate:

```
compute layer
cache layer
storage layer
```

Workers never mix responsibilities.

This improves:

- testability
- correctness
- scalability

---

# Part 4 — Distributed Cache Example

Real production systems use:

- Redis
- Memcached
- shared cache servers

Example with Redis:

```python
import redis

r = redis.Redis()

def cached(x):
    key = f"value:{x}"
    val = r.get(key)
    if val:
        return int(val)

    result = x * x
    r.set(key, result)
    return result
```

Now cache is shared across processes and machines.

---

# Part 5 — Cache Safety Rules

Caching must guarantee:

- deterministic outputs
- idempotent behavior
- expiration strategy
- bounded memory
- failure tolerance

Bad cache architecture causes subtle corruption.

---

# Final Insight

Caching is architecture, not optimization.

Safe caching:

reduces work  
improves scalability  
preserves correctness  

Bad caching:

creates invisible bugs.
