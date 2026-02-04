
# Iterators, Iterables & Generators

## Overview

Python needed a way to:

- process large or infinite data
- decouple production from consumption
- avoid loading everything into memory

Iterators and generators solve these problems.

Generators turn explicit state machines into resumable computations.

This module focuses on iteration as a system design tool.

---

## Section 1 — Iterable vs Iterator

Important distinction:

- Iterable → produces a new iterator each time
- Iterator → single-use and exhaustible

Iterator protocol:

- __iter__()
- __next__()

### Example

```python
class CountDown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.start <= 0:
            raise StopIteration
        value = self.start
        self.start -= 1
        return value

for n in CountDown(5):
    print(n)
```

---

## Exercise 1 — Custom Iterator

Build an iterator that yields squares until N.

Explain why iterators get exhausted.

---

## Section 2 — Fresh Iterators vs Broken Design

Returning the same iterator causes silent bugs.

### Broken

```python
class Batch:
    def __init__(self, data):
        self.data = iter(data)

    def __iter__(self):
        return self.data
```

First consumer exhausts it.

Second consumer sees nothing.

### Correct

```python
class Batch:
    def __init__(self, data):
        self.data = data

    def __iter__(self):
        return iter(self.data)
```

Fresh iterator every time.

---

## Section 3 — Generator Functions

Generators automatically implement the iterator protocol.

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1
```

No manual state management required.

Generators pause and resume execution.

---

## Exercise 2 — Fibonacci Generator

Create an infinite Fibonacci stream.

---

## Section 4 — Lazy Evaluation

Generators compute values only when needed.

```python
def infinite_stream():
    n = 0
    while True:
        yield n
        n += 1
```

Memory stays constant regardless of stream length.

---

## Section 5 — Producer–Consumer Decoupling

Generators separate production from consumption.

```python
def read_lines(filename):
    with open(filename) as f:
        for line in f:
            yield line.strip()

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line:
            yield line

for error in filter_errors(read_lines("log.txt")):
    print(error)
```

No full file is loaded into memory.

---

## Section 6 — Generator Pipelines

Generators compose into streaming pipelines.

```python
def square(numbers):
    for n in numbers:
        yield n * n

pipeline = square(range(5))

for x in pipeline:
    print(x)
```

Each stage processes lazily.

---

## Final Takeaways

Iterators and generators enable:

- streaming computation
- infinite data processing
- constant memory pipelines
- decoupled architecture
- early termination
- scalable data systems

Generators hide complex state machines behind simple syntax.

