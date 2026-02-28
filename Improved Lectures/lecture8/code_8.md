# Code — Lecture 8: Iterators and Iterables — The Iteration Protocol

---

# Example 8.1

```python
# Example 8.1: SequenceIterator — a custom iterator built from scratch

class SequenceIterator:  # line 1

    def __init__(self, values):  # line 2
        self.dataStore = values   # line 3 — store the list of values
        self.currentIndex = 0    # line 4 — start position at zero

    def __iter__(self):          # line 5
        return self              # line 6 — an iterator is its own iterator

    def __next__(self):          # line 7
        if self.currentIndex >= len(self.dataStore):  # line 8 — check if exhausted
            raise StopIteration                        # line 9 — signal end of sequence
        item = self.dataStore[self.currentIndex]       # line 10 — retrieve current item
        self.currentIndex += 1                         # line 11 — advance position
        return item                                    # line 12 — return the item


# Demonstration
iterator = SequenceIterator(["alpha", "beta", "gamma"])  # line 13

for value in iterator:          # line 14 — for loop calls __iter__ then __next__
    print(value)                # line 15

# Output:
# alpha
# beta
# gamma

print(list(iterator))  # line 16 — iterator is exhausted; second pass produces nothing
# Output: []
```

---

# Example 8.2

```python
# Example 8.2: Broken TransactionBatch — the silent iterator exhaustion bug

class TransactionBatchBroken:   # line 1

    def __init__(self, data):   # line 2
        self.data = data        # line 3 — store raw data
        self.iteratorState = iter(self.data)  # line 4 — BUG: immediately convert to iterator

    def __iter__(self):         # line 5
        return self             # line 6 — BUG: returns self, making this an iterator not an iterable


# Demonstration of the silent bug
transactions = [100.0, 250.0, 75.0, 430.0, 90.0]  # line 7
batch = TransactionBatchBroken(transactions)         # line 8

# First consumer: computes totals
total = 0                       # line 9
for amount in batch:            # line 10 — exhausts the iterator
    total += amount             # line 11
print("Total:", total)          # line 12
# Output: Total: 945.0

# Second consumer: tries to generate a report
report = []                     # line 13
for amount in batch:            # line 14 — iterator already exhausted; loop body never runs
    report.append(amount)       # line 15
print("Report entries:", len(report))  # line 16
# Output: Report entries: 0
# No error raised. Data silently disappeared.
```

---

# Example 8.3

```python
# Example 8.3: Fixed TransactionBatch — fresh iterator design

class TransactionBatch:                  # line 1

    def __init__(self, data):            # line 2
        self.transactionList = data      # line 3 — store raw data, never convert to iterator

    def __iter__(self):                  # line 4
        return iter(self.transactionList)  # line 5 — return a fresh iterator on every call


# Demonstration: both consumers see complete data
transactions = [100.0, 250.0, 75.0, 430.0, 90.0]  # line 6
batch = TransactionBatch(transactions)              # line 7

# First consumer: computes totals
total = 0                                # line 8
for amount in batch:                     # line 9 — gets a fresh iterator
    total += amount                      # line 10
print("Total:", total)                   # line 11
# Output: Total: 945.0

# Second consumer: generates a report
report = []                              # line 12
for amount in batch:                     # line 13 — gets another fresh iterator
    report.append(amount)               # line 14
print("Report entries:", len(report))   # line 15
# Output: Report entries: 5
# Both consumers see the complete data independently.

# Demonstrating simultaneous independent iterators
iteratorOne = iter(batch)  # line 16 — fresh iterator, position 0
iteratorTwo = iter(batch)  # line 17 — another fresh iterator, position 0
print(next(iteratorOne))   # line 18 — 100.0
print(next(iteratorTwo))   # line 19 — 100.0  (independent of iteratorOne)
print(next(iteratorOne))   # line 20 — 250.0  (iteratorOne advances independently)
```

---

# Example 8.4

```python
# Example 8.4: DataPage — a well-designed custom iterable

class DataPage:                              # line 1

    def __init__(self, records):             # line 2
        self.recordList = records            # line 3 — store raw records as a list

    def __len__(self):                       # line 4
        return len(self.recordList)          # line 5 — number of records in this page

    def __iter__(self):                      # line 6
        return iter(self.recordList)         # line 7 — fresh iterator every call


# Demonstration
pageOne = DataPage(["row_1", "row_2", "row_3", "row_4"])  # line 8

# Multiple independent iterations over the same DataPage
firstPass = list(pageOne)   # line 9 — iterates fully; produces complete list
secondPass = list(pageOne)  # line 10 — iterates fully again; same complete list

print("First pass:", firstPass)    # line 11
# Output: First pass: ['row_1', 'row_2', 'row_3', 'row_4']

print("Second pass:", secondPass)  # line 12
# Output: Second pass: ['row_1', 'row_2', 'row_3', 'row_4']

print("Page size:", len(pageOne))  # line 13
# Output: Page size: 4

# Nested loop — two independent iterators over the same object
for outerRecord in pageOne:                    # line 14 — outer iterator
    for innerRecord in pageOne:                # line 15 — completely separate inner iterator
        pass                                   # line 16 — no collision, no corruption
print("Nested iteration completed safely.")   # line 17
```

---
