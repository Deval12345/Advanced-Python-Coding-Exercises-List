# Example 1.1

```python
def greet(name):                                      # line 1
    return f"Hello {name}"                            # line 2

func = greet                                          # line 3
print(func("Alice"))                                  # line 4
```

# Example 2.1

```python
def process(numbers, rule):                          # line 1
    return [rule(n) for n in numbers]                 # line 2

def addTax(x):                                        # line 3
    return x * 1.18                                  # line 4

def applyDiscount(x):                                 # line 5
    return x * 0.9                                   # line 6

values = [100, 200, 300]                             # line 7
print(process(values, addTax))                        # line 8
print(process(values, applyDiscount))                 # line 9
```

# Example 3.1

```python
def flatFee(amount):                                  # line 1
    return amount + 50                               # line 2

def percentFee(amount):                               # line 3
    return amount * 1.05                             # line 4

def tieredFee(amount):                                # line 5
    return amount + (20 if amount < 500 else 10)     # line 6

strategies = {                                        # line 7
    "flat": flatFee,                                  # line 8
    "percent": percentFee,                            # line 9
    "tiered": tieredFee                               # line 10
}

def checkout(amount, strategyName):                   # line 11
    strategy = strategies[strategyName]               # line 12
    return strategy(amount)                           # line 13

print(checkout(400, "flat"))                          # line 14
print(checkout(400, "percent"))                       # line 15
```

# Example 4.1

```python
class Multiplier:                                     # line 1
    def __init__(self, factor):                       # line 2
        self.factor = factor                          # line 3
    def __call__(self, value):                        # line 4
        return value * self.factor                    # line 5

double = Multiplier(2)                                # line 6
print(double(10))                                     # line 7
```
