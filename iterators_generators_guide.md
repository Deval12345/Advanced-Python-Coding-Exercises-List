
# Iterators, Generators & Lazy Evaluation in Python

## Overview

Python’s iterator and generator model allows programs to:

- Process large or infinite datasets
- Decouple data production from consumption
- Avoid loading everything into memory
- Stream computations safely
- Build scalable data pipelines

Generators turn explicit state machines into simple code.

This module explains iteration as a design tool — not just syntax.

You will learn:

✔ Iterator protocol  
✔ Iterable vs iterator  
✔ Generator functions  
✔ Lazy evaluation  
✔ Streaming architecture  
✔ Memory-safe processing  

---

## Section 1 — Iterable vs Iterator

Important distinction:

- Iterable → produces new iterator each time
- Iterator → single-use, gets exhausted

Built-ins like `list()`, `set()`, `tuple()` expect objects implementing `__iter__`.

### Bug-Driven Example

If an object returns the same iterator every time:

- First consumer exhausts it
- Second consumer sees nothing

### Broken Design

```python
class TransactionBatch:
    def __init__(self, data):
        self.data = iter(data)  # single iterator

    def __iter__(self):
        return self.data
```

```python
batch = TransactionBatch([10, 20, 30])

print(sum(batch))   # consumes iterator
print(list(batch))  # empty!
```

### Correct Design

```python
class TransactionBatch:
    def __init__(self, data):
        self.data = data

    def __iter__(self):
        return iter(self.data)  # fresh iterator
```

Now iteration works everywhere.

---

## Exercise 1 — Fix Exhaustion Bug

Make a batch reusable across:

- sum()
- list()
- loops
- comprehensions

Explain why iterators must be fresh.

---

## Section 2 — Why __iter__ Matters

Without iteration support, consumers depend on internal APIs.

### Bad API

```python
class Batch:
    def get_records(self):
        return [1, 2, 3]
```

Every caller must use `get_records()`.

If internal storage changes → all callers break.

### Iterable Design

```python
class Batch:
    def __init__(self, data):
        self.data = data

    def __iter__(self):
        return iter(self.data)
```

Now works with:

- loops
- sum()
- list()
- any()
- comprehensions

No caller changes required.

---

## Exercise 2 — Make a Custom Object Iterable

Add `__iter__` so existing code works unchanged.

---

## Section 3 — Why yield Matters

Eager iteration loads everything.

Lazy iteration streams data.

### Naïve Design (memory explosion)

```python
class TransactionLog:
    def __init__(self, filename):
        self.filename = filename

    def __iter__(self):
        with open(self.filename) as f:
            return [line.strip() for line in f]  # loads entire file
```

### Streaming Design (correct)

```python
class TransactionLog:
    def __init__(self, filename):
        self.filename = filename

    def __iter__(self):
        with open(self.filename) as f:
            for line in f:
                yield line.strip()
```

Now it processes 5GB safely.

---

## Exercise 3 — Stream Large File

Use `yield` to process transactions lazily.

Test with:

```python
any(int(x) > 1_000_000 for x in TransactionLog("transactions.log"))
```

No RAM blow-up.

---

## Section 4 — Generators as Pipelines

Generators support composable streaming systems.

### Integrated Task — Time-Series Analyzer

Tools used:

- itertools.islice
- itertools.chain
- itertools.takewhile
- itertools.cycle

### Implementation

```python
import itertools

def read_stream(filename):
    with open(filename) as f:
        for line in f:
            yield float(line.strip())

def anomaly_detector(stream):
    for value in stream:
        if value > 1000:
            print("Anomaly detected:", value)
            break
        yield value


stream1 = read_stream("series1.txt")
stream2 = read_stream("series2.txt")

combined = itertools.chain(stream1, stream2)
window = itertools.islice(combined, 1000)
processed = anomaly_detector(window)

for v in processed:
    pass
```

This pipeline:

✔ Streams data  
✔ Uses constant memory  
✔ Stops early  
✔ Chains sources  

---

## Final Takeaways

Generators enable:

✔ Lazy evaluation  
✔ Constant memory processing  
✔ Infinite streams  
✔ Scalable pipelines  
✔ Early termination  
✔ Clean data architecture  

They are essential for large systems.

---

## Suggested Extensions

1. Build a CSV streaming reader
2. Create infinite random number generator
3. Implement rolling averages with generators
4. Write log analyzer pipeline
5. Benchmark eager vs lazy loading

---

End of module.
