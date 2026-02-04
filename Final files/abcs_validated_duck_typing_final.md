
# Abstract Base Classes (ABCs) & collections.abc

## Overview

Pure duck typing lacks explicit contracts in large systems.

ABCs exist to:

- formalize expectations
- enable early failure
- support tooling
- validate extensions safely

They are not about inheritance for code reuse.
They are about runtime interface validation.

Ramalho calls them:

validated duck typing.

---

## Section 1 — Why Pure Duck Typing Fails at Scale

Duck typing delays failure until deep runtime.

```python
def process(gateway):
    gateway.pay(100)
```

If pay() is missing, the crash happens late.

Large systems need early rejection.

ABCs fail fast.

---

## Section 2 — ABCs as Validated Duck Typing

ABCs enforce behavior at instantiation.

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


good = GoodPlugin()
bad = BadPlugin()  # TypeError
```

Invalid implementations fail immediately.

---

## Exercise — Plugin Interface Contract

Define an interface and reject invalid plugins before execution.

Explain why early failure is safer.

---

## Section 3 — Payment Gateway Example

```python
class PaymentGateway(ABC):
    @abstractmethod
    def pay(self, amount):
        pass


class Stripe(PaymentGateway):
    def pay(self, amount):
        print("Paid:", amount)


class Broken(PaymentGateway):
    pass


stripe = Stripe()
stripe.pay(100)

broken = Broken()  # TypeError
```

Duck typing preserved.
Validation enforced.

---

## Section 4 — collections.abc Behavioral Contracts

Python ships runtime capability contracts:

- Iterable
- Sequence
- Mapping
- Sized
- Callable

```python
from collections.abc import Iterable

def requires_iterable(x):
    if not isinstance(x, Iterable):
        raise TypeError("Not iterable")
```

Contracts validate behavior, not concrete types.

---

## Section 5 — ABCs vs Protocols

Protocols → static guarantees  
ABCs → runtime guarantees

Python executes without type checkers.

ABCs enforce behavior in production.

Protocols assist development.

Large systems use both.

---

## Final Takeaways

ABCs provide:

- runtime contracts
- early failure
- safe extension points
- validated duck typing
- scalable architecture

They formalize behavior without sacrificing flexibility.
