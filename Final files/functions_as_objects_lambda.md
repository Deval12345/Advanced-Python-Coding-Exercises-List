
# Functions as Objects & Lambda Functions

## Overview

Python treats functions as first-class objects so behavior can be passed,
stored, returned, and composed.

This enables:

- runtime strategy injection
- extensible system design
- fewer conditionals
- behavior pipelines
- lightweight customization with lambdas

This module focuses on system design, not functional purity.

---

## Section 1 — Functions as First-Class Objects

Functions behave like values.

```python
def greet(name):
    return f"Hello {name}"

func = greet
print(func("Alice"))
```

Functions can be assigned, stored, and passed like any object.

---

## Exercise 1 — Behavior as Data

```python
def process(numbers, rule):
    return [rule(x) for x in numbers]

def tax(x):
    return x * 1.18

def discount(x):
    return x * 0.9

nums = [100, 200, 300]

print(process(nums, tax))
print(process(nums, discount))
```

Behavior is injected instead of hardcoded.

---

## Section 2 — Runtime Strategy Selection

```python
def flat_fee(amount):
    return amount + 50

def percent_fee(amount):
    return amount * 1.05

strategies = {
    "flat": flat_fee,
    "percent": percent_fee
}

def checkout(amount, strategy_name):
    return strategies[strategy_name](amount)
```

No if/elif chains — logic is selected dynamically.

---

## Section 3 — Closures and Captured State

Functions can remember data from their creation context.

```python
def make_validator(min_amount):
    def validator(value):
        return value >= min_amount
    return validator

validate_100 = make_validator(100)
print(validate_100(150))
```

Closures create customized behavior without classes.

---

## Section 4 — Functions in Data Structures

```python
def add_tax(x): return x * 1.18
def discount(x): return x * 0.9

pipeline = [add_tax, discount]

value = 100
for step in pipeline:
    value = step(value)

print(value)
```

Pipelines allow composable behavior.

---

## Section 5 — Lambda Functions

Lambdas provide lightweight inline behavior.

```python
nums = [1, 2, 3, 4]
squares = list(map(lambda x: x*x, nums))
print(squares)
```

They are anonymous, compact function objects.

Useful for:

- callbacks
- sorting rules
- small transformations

---

## Exercise 2 — Lambda Sorting

```python
items = [("apple", 3), ("banana", 1), ("cherry", 2)]
items.sort(key=lambda x: x[1])
print(items)
```

---

## Final Takeaways

First-class functions enable:

- strategy injection
- pipelines
- closures
- event-driven design
- replaceable behavior

Lambdas are compact behavior carriers.

Functions collapse design patterns and reduce conditionals.

