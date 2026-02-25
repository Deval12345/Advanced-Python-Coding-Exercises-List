# Example 1.1

```python
class LengthExample:                                  # line 1
    def __init__(self, dataSource):                   # line 2
        self.valueStore = dataSource                  # line 3
    def __len__(self):                                # line 4
        return len(self.valueStore)                   # line 5
```

# Example 1.2

```python
class IndexContainer:                                 # line 1
    def __init__(self, dataSource):                   # line 2
        self.dataStore = dataSource                   # line 3
    def __getitem__(self, position):                  # line 4
        return self.dataStore[position]               # line 5
```

# Example 1.3

```python
class IterableExample:                                # line 1
    def __init__(self, dataSource):                   # line 2
        self.dataStore = dataSource                   # line 3
    def __iter__(self):                               # line 4
        return iter(self.dataStore)                   # line 5
```

# Example 1.4

```python
class ContainmentExample:                             # line 1
    def __init__(self, dataSource):                   # line 2
        self.dataStore = dataSource                   # line 3
    def __contains__(self, itemValue):                # line 4
        return itemValue in self.dataStore            # line 5
```

# Example 3.1

```python
class PriceValue:                                     # line 1
    def __init__(self, amount):                       # line 2
        self.amount = amount                          # line 3
    def __add__(self, otherValue):                    # line 4
        return PriceValue(self.amount + otherValue.amount)  # line 5
```

# Example 4.5

```python
class DataStream:                                     # line 1
    def __init__(self, dataSource):                   # line 2
        self.dataStore = dataSource                   # line 3
    def __iter__(self):                               # line 4
        return iter(self.dataStore)                   # line 5
```

# Example 6.3

```python
class DiscountStrategy:                               # line 1
    def __init__(self, rate):                         # line 2
        self.rate = rate                              # line 3
    def apply(self, priceValue):                      # line 4
        return priceValue * self.rate                 # line 5
```
