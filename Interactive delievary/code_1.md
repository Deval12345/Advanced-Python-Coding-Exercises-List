## Module 1

```python
class CustomList:                         # Line 1
    def __init__(self, data):              # Line 2
        self._data = list(data)            # Line 3


    def __len__(self):                     # Line 6
        return len(self._data)             # Line 7


    def __getitem__(self, index):          # Line 10
        return self._data[index]           # Line 11


    def __contains__(self, item):          # Line 14
        return item in self._data          # Line 15


    def __iter__(self):                    # Line 18
        return iter(self._data)            # Line 19
```

---

## Module 2

```python
def analyze(source):                                       # Line 1
    lines = list(source)                                   # Line 2
    return {                                               # Line 3
        "line_count": len(lines),                          # Line 4
        "word_count": sum(len(line.split()) for line in lines),  # Line 5
        "first_line": lines[0] if lines else None          # Line 6
    }
```

---

## Module 3

```python
class LazyFileLoader:                       # Line 1
    def __init__(self, filename):           # Line 2
        self.filename = filename            # Line 3


    def __getitem__(self, index):           # Line 6
        with open(self.filename) as f:      # Line 7
            for i, line in enumerate(f):    # Line 8
                if i == index:              # Line 9
                    return line.strip()     # Line 10
        raise IndexError                    # Line 11
```

---

## Module 4

```python
class Discount10:               # Line 1
    def apply(self, price):     # Line 2
        return price * 0.9      # Line 3


class Discount20:               # Line 6
    def apply(self, price):     # Line 7
        return price * 0.8      # Line 8
```
