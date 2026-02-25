# EXAMPLE 1.1

```python
class LengthContainer:                                # line 1
    def __init__(self, dataStore):                    # line 2
        self.dataStore = dataStore                    # line 3
    def __len__(self):                                # line 4
        return len(self.dataStore)                    # line 5

lengthContainer = LengthContainer([10, 20, 30])       # line 6
print(len(lengthContainer))                           # line 7
```

# EXAMPLE 2.1

```python
class NumberStream:                                   # line 1
    def __iter__(self):                               # line 2
        return iter([1, 2, 3])                        # line 3

numberStream = NumberStream()                         # line 4
print(sum(numberStream))                              # line 5
```

# EXAMPLE 3.1

```python
class DiscountCalculator:                             # line 1
    def apply(self, value):                           # line 2
        return value * 0.9                            # line 3

discountCalculator = DiscountCalculator()             # line 4
print(discountCalculator.apply(100))                  # line 5
```

# EXAMPLE 4.1

```python
class SimpleWriter:                                   # line 1
    def write(self, text):                            # line 2
        print(text)                                   # line 3

simpleWriter = SimpleWriter()                         # line 4
simpleWriter.write("Hello")                           # line 5
```
