# Python Data Model (Magic / Dunder Methods)

## Introduction

In this lecture, we explore something very fundamental about Python that often hides in plain sight: **Python’s data model**.
Unlike many languages where syntax feels separate from objects, Python is designed as an **object-first framework**. Almost everything you do—indexing, looping, checking membership, even arithmetic—works because objects agree to a common protocol defined by special methods, often called *dunder methods*.

Understanding this unlocks a deeper level of Python mastery. You stop memorizing syntax and start *designing behavior*.

## Python as a Framework

Python does not introduce new syntax for new behaviors. Instead, it exposes its core behavior through objects.

When you write:
- `len(x)`
- `x[i]`
- `for item in obj`
- `value in obj`

Python is simply calling special methods implemented by the object.

## Syntax to Method Mapping

- `len(x)` → `x.__len__()`
- `x[i]` → `x.__getitem__(i)`
- `for item in obj` → `obj.__iter__()`
- `x in obj` → `obj.__contains__(x)`

## Building a Native-Like Container

```python
class CustomList:
    def __init__(self, data):
        self._data = list(data)

    def __len__(self):
        return len(self._data)

    def __getitem__(self, index):
        return self._data[index]

    def __contains__(self, item):
        return item in self._data

    def __iter__(self):
        return iter(self._data)
```

## Duck Typing

```python
def analyze(source):
    lines = list(source)
    return {
        "line_count": len(lines),
        "word_count": sum(len(line.split()) for line in lines),
        "first_line": lines[0] if lines else None
    }
```

## Lazy Indexable Objects

```python
class LazyFileLoader:
    def __init__(self, filename):
        self.filename = filename

    def __getitem__(self, index):
        with open(self.filename) as f:
            for i, line in enumerate(f):
                if i == index:
                    return line.strip()
        raise IndexError
```

## Strategy Pattern via Duck Typing

```python
class Discount10:
    def apply(self, price):
        return price * 0.9

class Discount20:
    def apply(self, price):
        return price * 0.8
```

## Final Takeaways

Special methods are the API of Python itself.
