
# Abstract Base Classes (ABCs) — Validated Duck Typing

## Overview

Duck typing is flexible but fragile at scale.

In small systems:
"If it quacks like a duck…" is enough.

In large systems:
You need explicit behavioral contracts.

Abstract Base Classes (ABCs) formalize duck typing by validating expected behavior at runtime.

They are not about inheritance for code reuse.
They are about **interface validation**.

You will learn:

✔ Why pure duck typing fails late  
✔ collections.abc behavioral contracts  
✔ Abstract methods  
✔ Custom ABC design  
✔ Runtime interface validation  
✔ ABCs vs Protocols  
✔ Framework extension safety  

---

## Section 1 — The Problem with Pure Duck Typing

Duck typing delays failure until runtime use.

Example failure:

```python
def process(payment_gateway):
    payment_gateway.pay(100)
```

If `pay()` is missing, failure happens deep in production.

This is late failure.

Large systems need early rejection.

---

## Section 2 — ABCs as Validated Duck Typing

ABCs enforce interface requirements at instantiation.

They validate behavior before execution.

### Exercise — Plugin Interface

```python
from abc import ABC, abstractmethod

class Plugin(ABC):
    @abstractmethod
    def run(self):
        pass


class GoodPlugin(Plugin):
    def run(self):
        print("Running")


class BadPlugin(Plugin):
    pass


p = GoodPlugin()   # works
b = BadPlugin()    # raises TypeError
```

Invalid implementations fail immediately.

No runtime surprises.

---

## Section 3 — Payment Gateway Exercise

Validated duck typing in a real system.

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def pay(self, amount):
        pass


class StripeGateway(PaymentGateway):
    def pay(self, amount):
        print("Stripe processed", amount)


class BrokenGateway(PaymentGateway):
    pass


gateway = StripeGateway()
gateway.pay(100)

broken = BrokenGateway()  # TypeError
```

Outcome:

✔ Duck typing preserved  
✔ Early failure  
✔ Explicit behavioral contract  
✔ Safer extensibility  

---

## Section 4 — collections.abc

Python ships built-in behavioral contracts:

- Iterable
- Mapping
- Sequence
- Sized
- Callable

Example:

```python
from collections.abc import Iterable

def requires_iterable(x):
    if not isinstance(x, Iterable):
        raise TypeError("Not iterable")
```

These validate capabilities, not concrete types.

---

## Section 5 — ABCs vs Protocols

Modern rule:

Protocols → static correctness (type checkers)  
ABCs → runtime correctness

Python runs without a type checker in production.

ABCs guarantee behavior at runtime.

Protocols guarantee behavior during development.

Use both when systems grow.

---

## Framework Mapping

Django uses ABC-style runtime contracts:

- BaseUserManager
- Storage
- CacheBase
- MiddlewareMixin

FastAPI emphasizes typing + runtime validation:

- Pydantic BaseModel
- middleware classes
- typed dependencies

Core Python exposes both:

- collections.abc → runtime ABCs
- typing.Protocol → static structural typing

---

## Exercise — Reject Invalid Plugin Early

Build a plugin loader that refuses invalid implementations
before execution begins.

Explain why early failure prevents production outages.

---

## Final Takeaways

ABCs provide:

✔ Runtime contracts  
✔ Early validation  
✔ Safer extension points  
✔ Explicit behavior guarantees  
✔ Scalable architecture  

They formalize duck typing without sacrificing flexibility.

---

## Suggested Extensions

1. Build plugin registry with ABC validation
2. Combine ABC + Protocol
3. Add runtime capability checker
4. Design extensible payment framework
5. Create middleware contract system

---

End of module.
