# Code — Lecture 9: Generators — Lazy Evaluation and Producer-Consumer Architecture

---

# Example 9.1 — Simple Countdown Generator

```python
# line 1  Define a generator function that counts down from a start value
def countdown(startValue):
    # line 2  Loop while startValue is still positive
    while startValue > 0:
        # line 3  Yield the current value — execution pauses here
        yield startValue
        # line 4  Decrement startValue before the next iteration resumes
        startValue -= 1

# line 5  Calling the generator function returns a generator object immediately
# line 6  The function body does NOT run yet at this point
generatorObject = countdown(10)

# line 7  The generator object has both __iter__ and __next__ automatically
# line 8  A for loop calls __next__ repeatedly until StopIteration is raised
for value in generatorObject:
    # line 9  Each iteration resumes the generator from the line after yield
    print(value)
# Output: 10, 9, 8, 7, 6, 5, 4, 3, 2, 1

# line 10  Demonstrate that a generator object is its own iterator
generatorTwo = countdown(3)
# line 11  iter() on a generator returns the same object — it is its own __iter__
print(iter(generatorTwo) is generatorTwo)  # True
# line 12  Manually calling next() to show pause-and-resume behavior
print(next(generatorTwo))  # 3
print(next(generatorTwo))  # 2
print(next(generatorTwo))  # 1
# line 13  Calling next() after exhaustion raises StopIteration automatically
```

---

# Example 9.2 — Generator Expression vs List Comprehension

```python
import sys

# line 1  A list comprehension computes ALL values immediately and stores them
squaresList = [x * x for x in range(1_000_000)]

# line 2  A generator expression computes NOTHING until you iterate
# line 3  The only difference in syntax: parentheses instead of brackets
squaresGenerator = (x * x for x in range(1_000_000))

# line 4  Compare memory usage — the difference is dramatic for large ranges
listMemory = sys.getsizeof(squaresList)
generatorMemory = sys.getsizeof(squaresGenerator)

# line 5  Print the sizes to show the contrast
print(f"List size:      {listMemory:,} bytes")       # ~8,000,056 bytes
print(f"Generator size: {generatorMemory} bytes")    # ~104 bytes (constant)

# line 6  Both produce identical results when iterated — behavior is the same
# line 7  The generator computes each square lazily as the loop requests it
sumFromList      = sum(squaresList)
sumFromGenerator = sum(x * x for x in range(1_000_000))  # no extra parens needed
print(sumFromList == sumFromGenerator)  # True

# line 8  A generator expression passed directly to a function is idiomatic Python
maxSquare = max(x * x for x in range(100))
print(maxSquare)  # 9801
```

---

# Example 9.3 — Lazy Transaction Log Reader

```python
# line 1  Simulate raw log lines as a list (in production this would be a file handle)
rawLogLines = [
    "150.00 USD",
    "  -20.00 USD  ",
    "300.50 EUR",
    "  0.00 USD  ",
    "75.25 EUR",
    "500.00 USD",
]

# line 2  Define a generator function that parses transaction log lines lazily
def readTransactions(logLines):
    # line 3  Iterate over each raw line one at a time — never holding all results
    for rawLine in logLines:
        # line 4  Strip surrounding whitespace from the raw line
        cleanLine = rawLine.strip()
        # line 5  Skip empty lines gracefully
        if not cleanLine:
            continue
        # line 6  Split the line into its two parts: amount string and currency code
        parts = cleanLine.split()
        amountStr = parts[0]
        currencyCode = parts[1]
        # line 7  Yield a dictionary representing one parsed transaction
        # line 8  Execution pauses here and resumes when next() is called again
        yield {"amount": float(amountStr), "currency": currencyCode}

# line 9  Create the generator — nothing has been parsed yet at this point
transactionGenerator = readTransactions(rawLogLines)

# line 10  Consume the generator one record at a time
for transaction in transactionGenerator:
    # line 11  Each iteration parses exactly one line and then pauses
    print(f"Amount: {transaction['amount']:>8.2f}  Currency: {transaction['currency']}")

# line 12  Demonstrate that the generator is exhausted after one full pass
# line 13  Calling next() on an exhausted generator raises StopIteration
exhaustedGenerator = readTransactions(rawLogLines)
firstRecord  = next(exhaustedGenerator)  # Parses only line 1
secondRecord = next(exhaustedGenerator)  # Parses only line 2
print(f"First:  {firstRecord}")
print(f"Second: {secondRecord}")
```

---

# Example 9.4 — Three-Stage Generator Pipeline

```python
# line 1  Stage 1: Producer — yields raw records from a data source one at a time
def readRecords(sourceData):
    # line 2  Iterate over the source — could be a file, database cursor, API stream
    for record in sourceData:
        # line 3  Yield each record without modification — pure pass-through producer
        yield record

# line 4  Stage 2: Filter — yields only records with a positive amount
def filterValidRecords(records):
    # line 5  Receive the upstream generator as input; iterate it lazily
    for record in records:
        # line 6  Drop records with zero or negative amounts
        if record["amount"] > 0:
            # line 7  Yield the valid record downstream to the next stage
            yield record

# line 8  Stage 3: Transform — yields computed totals for a specific currency
def computeTotals(validRecords, currency):
    # line 9  Receive the upstream generator as input; iterate it lazily
    for record in validRecords:
        # line 10  Only process records that match the requested currency
        if record["currency"] == currency:
            # line 11  Yield the computed total for this matching record
            yield record["amount"] * record["quantity"]

# line 12  Sample source data — in production this would be a file or database query
sampleData = [
    {"amount": 150.00, "currency": "USD", "quantity": 2},
    {"amount": -20.00, "currency": "USD", "quantity": 1},
    {"amount": 300.50, "currency": "EUR", "quantity": 3},
    {"amount":   0.00, "currency": "USD", "quantity": 1},
    {"amount":  75.25, "currency": "EUR", "quantity": 4},
    {"amount": 500.00, "currency": "USD", "quantity": 1},
]

# line 13  Assemble the pipeline by chaining each generator into the next
# line 14  Nothing runs yet — we have only described the computation
producerStage  = readRecords(sampleData)
filterStage    = filterValidRecords(producerStage)
transformStage = computeTotals(filterStage, currency="USD")

# line 15  Iteration triggers the entire pipeline lazily, one record at a time
# line 16  Each call to next() on transformStage pulls one value through all stages
print("USD totals:")
for total in transformStage:
    # line 17  One value travels through all three stages per loop iteration
    print(f"  {total:.2f}")

# line 18  Pipeline memory is constant — no stage buffers all records
# line 19  Each stage processes one record, yields one result, then pauses
```

---

# Example 9.5 — Lazy File Reader for Large Datasets

```python
import os

# line 1  Define a generator function that reads a large file one line at a time
def lazyFileReader(filePath):
    # line 2  Open the file — the with statement ensures it is closed after use
    with open(filePath, "r") as fileHandle:
        # line 3  Iterate over the file handle one line at a time
        for rawLine in fileHandle:
            # line 4  Strip newline characters from the end of each line
            cleanLine = rawLine.rstrip("\n")
            # line 5  Yield the clean line — file stays open between yields
            # line 6  Only one line is in memory at any moment
            yield cleanLine

# line 7  Define a generator that filters for lines containing a keyword
def filterByKeyword(lines, keyword):
    # line 8  Receive the upstream generator and iterate it lazily
    for line in lines:
        # line 9  Yield only lines that contain the keyword
        if keyword in line:
            yield line

# line 10  Define a generator that extracts a specific field from each matching line
def extractField(matchingLines, fieldIndex):
    # line 11  Receive the upstream generator and iterate it lazily
    for line in matchingLines:
        # line 12  Split the line by comma and yield the requested field
        parts = line.split(",")
        if fieldIndex < len(parts):
            yield parts[fieldIndex].strip()

# line 13  Demonstrate the pipeline with a small in-memory simulation
# line 14  In production, lazyFileReader would receive an actual file path
simulatedLines = [
    "2024-01-15,USD,150.00,BUY",
    "2024-01-15,EUR,300.50,SELL",
    "2024-01-16,USD,500.00,BUY",
    "2024-01-16,GBP,200.00,BUY",
    "2024-01-17,USD,75.25,SELL",
]

# line 15  Build the pipeline: filter for USD lines, then extract the amount field
usdLines     = (line for line in simulatedLines if "USD" in line)
amountFields = extractField(usdLines, fieldIndex=2)

# line 16  Consume the pipeline — only USD amounts are extracted
print("USD transaction amounts:")
for amount in amountFields:
    print(f"  {amount}")
# Output: 150.00, 500.00, 75.25

# line 17  Show memory behavior: the generator object itself is tiny
generatorSize = __import__("sys").getsizeof(extractField(iter(simulatedLines), 2))
print(f"\nGenerator object size: {generatorSize} bytes (constant regardless of file size)")
```
