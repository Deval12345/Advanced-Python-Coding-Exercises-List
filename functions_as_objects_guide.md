
# Functions as Objects in Python

## Overview

Python treats functions as first-class objects. This allows behavior to be:

- Passed as arguments
- Stored in variables
- Returned from other functions
- Stored inside data structures
- Composed into pipelines

This is about system design and flexibility, not functional purity.

You will learn:

✔ Behavior as data  
✔ Runtime strategy selection  
✔ Closures and captured state  
✔ Returning functions  
✔ Functions inside data structures  
✔ Designing extensible systems  

---

## Section 1 — Functions as First-Class Objects

Functions are objects. They can be assigned, passed, and stored.

### Example

```python
def greet(name):
    return f"Hello {name}"

func = greet
print(func("Alice"))
```

No special syntax — functions behave like values.

This removes conditionals and replaces them with interchangeable behavior.

---

## Exercise 1 — Behavior as Data

### Goal

Process numbers using a dynamic rule.

### Implementation

```python
def process(numbers, rule):
    return [rule(x) for x in numbers]


def tax(x):
    return x * 1.18


def discount(x):
    return x * 0.9


def round_value(x):
    return round(x)


nums = [100, 200, 300]

print(process(nums, tax))
print(process(nums, discount))
print(process(nums, round_value))
```

### Lesson

Behavior is injected instead of hardcoded.

---

## Section 2 — Runtime Strategy Selection

Instead of if–elif chains, select functions dynamically.

### Exercise 2 — Pricing Engine

```python
def flat_fee(amount):
    return amount + 50

def percent_fee(amount):
    return amount * 1.05

def tiered_fee(amount):
    return amount + (20 if amount < 500 else 10)


strategies = {
    "flat": flat_fee,
    "percent": percent_fee,
    "tiered": tiered_fee
}

def checkout(amount, strategy_name):
    strategy = strategies[strategy_name]
    return strategy(amount)


print(checkout(400, "flat"))
print(checkout(400, "percent"))
print(checkout(400, "tiered"))
```

### Lesson

Logic is selected at runtime without conditionals.

---

## Section 3 — Returning Behavior (Closures)

Functions can create customized behavior.

### Exercise 3 — Validator Factory

```python
def make_validator(min_amount, currency):
    def validator(tx):
        return tx["amount"] >= min_amount and tx["currency"] == currency
    return validator


validate_inr = make_validator(100, "INR")

tx = {"amount": 150, "currency": "INR"}
print(validate_inr(tx))
```

The inner function remembers captured values.

This is a closure.

---

## Section 4 — Functions in Data Structures

Functions can live inside containers.

### Exercise 4 — Transformation Pipeline

```python
def add_tax(x): return x * 1.18
def apply_discount(x): return x * 0.9
def round_price(x): return round(x)

pipeline = [add_tax, apply_discount, round_price]

value = 100

for step in pipeline:
    value = step(value)

print(value)
```

Pipelines enable composable systems.

---

## Concrete Scenario — Shipping Rules

Hardcoded behavior:

```python
def shipping_cost(weight, city, premium):
    if weight < 1:
        return 50
    elif city == "Delhi":
        return 30
    elif premium:
        return 0
```

When business rules change, this becomes fragile.

### Better Design

```python
def checkout(weight, city, premium, rule):
    return rule(weight, city, premium)


def festival_rule(weight, city, premium):
    return 0


print(checkout(2, "Delhi", False, festival_rule))
```

Behavior is replaceable without editing checkout logic.

This is how real systems stay maintainable.

---

## Final Takeaways

First-class functions enable:

✔ Strategy pattern  
✔ Callbacks  
✔ Middleware  
✔ Event-driven systems  
✔ Pipeline architectures  

They reduce conditionals and improve extensibility.

---

## Suggested Extensions

1. Build a plugin registry using functions
2. Create a middleware pipeline for request processing
3. Implement event listeners with callback lists
4. Compose data-processing pipelines dynamically

---

End of module.
