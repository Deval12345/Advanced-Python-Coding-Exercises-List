# Code — Lecture 11: Context Managers — Structural Cleanup and Resource Lifecycle

---

## Example 11.1 — DatabaseConnection: Class-Based Context Manager

```python
# Example 11.1
class DatabaseConnection:                                          # line 1
    def __enter__(self):                                           # line 2
        print("Connection opened")                                 # line 3
        return self                                                # line 4

    def __exit__(self, exceptionType, exceptionValue, tracebackInfo):  # line 5
        print("Connection closed")                                 # line 6
        if exceptionType is not None:                              # line 7
            print("Rolling back transaction")                      # line 8
        return False                                               # line 9

with DatabaseConnection() as connection:                           # line 10
    print("Executing query")                                       # line 11
```

**Output:**
```
Connection opened
Executing query
Connection closed
```

**Explanation:**
- Line 1: DatabaseConnection is an ordinary class; it becomes a context manager by implementing __enter__ and __exit__
- Line 2: __enter__ is called by Python when the with block is entered; it receives no arguments beyond self
- Line 3: setup behavior runs here — in a real system this would open a socket or acquire a connection from a pool
- Line 4: returning self makes the connection object available as the "as" variable in the with statement
- Line 5: __exit__ receives three arguments describing any exception; all three are None when no exception occurred
- Line 6: teardown behavior runs here — always, regardless of whether the block raised an exception
- Line 7: exceptionType is not None only when an exception escaped the with block
- Line 8: rollback logic runs only when an exception occurred; cleanup still completes regardless
- Line 9: returning False means exceptions are not suppressed; they propagate after __exit__ finishes
- Line 10: Python calls DatabaseConnection().__enter__() and binds the return value to connection
- Line 11: the indented block runs after __enter__ returns; __exit__ is called when this block exits

---

## Example 11.2 — DataImporter: Selective Exception Suppression

```python
# Example 11.2
class DataImporter:                                                # line 1
    def __enter__(self):                                           # line 2
        print("Import session started")                            # line 3
        return self                                                # line 4

    def __exit__(self, exceptionType, exceptionValue, tracebackInfo):  # line 5
        if exceptionType is ValueError:                            # line 6
            print(f"Skipping bad row: {exceptionValue}")           # line 7
            return True                                            # line 8
        print("Import session ended")                              # line 9
        return False                                               # line 10

rowData = ["42", "bad_value", "17", "also_bad", "99"]             # line 11
for rawValue in rowData:                                           # line 12
    with DataImporter() as importer:                               # line 13
        parsedValue = int(rawValue)                                # line 14
        print(f"Imported: {parsedValue}")                          # line 15
```

**Output:**
```
Import session started
Imported: 42
Import session ended
Import session started
Skipping bad row: invalid literal for int() with base 10: 'bad_value'
Import session started
Imported: 17
Import session ended
Import session started
Skipping bad row: invalid literal for int() with base 10: 'also_bad'
Import session started
Imported: 99
Import session ended
```

**Explanation:**
- Line 5: __exit__ is always called; its return value determines whether the exception propagates
- Line 6: the identity check uses "is" not "=="; exception types are compared by identity
- Line 7: exceptionValue is the actual exception instance; its message is printed to show which row failed
- Line 8: returning True suppresses the ValueError; Python does not raise it after __exit__ returns
- Line 9: the cleanup message only prints when no ValueError occurred — the two branches are mutually exclusive
- Line 10: returning False for all other exception types means unexpected errors (MemoryError, IOError) still propagate
- Line 11: rowData contains a mix of valid strings and unparseable strings
- Line 12: each iteration of the loop creates a fresh with block and a fresh DataImporter instance
- Line 13: a new context is entered for every row; __enter__ and __exit__ run once per iteration
- Line 14: int(rawValue) raises ValueError for non-numeric strings; __exit__ intercepts it
- Line 15: this line only runs when int() succeeds; bad rows never reach it

---

## Example 11.3 — Timer: Guaranteed Timing Instrumentation

```python
# Example 11.3
import time                                                        # line 1

class Timer:                                                       # line 2
    def __enter__(self):                                           # line 3
        self.startTime = time.time()                               # line 4
        return self                                                 # line 5

    def __exit__(self, exceptionType, exceptionValue, tracebackInfo):  # line 6
        elapsedTime = time.time() - self.startTime                 # line 7
        print(f"Elapsed: {elapsedTime:.4f} seconds")               # line 8
        return False                                               # line 9

with Timer():                                                      # line 10
    total = sum(range(1_000_000))                                  # line 11
    print(f"Sum: {total}")                                         # line 12
```

**Output:**
```
Sum: 499999500000
Elapsed: 0.0312 seconds
```

**Explanation:**
- Line 1: time module provides time.time() which returns the current time as a float in seconds
- Line 4: startTime is stored as an instance attribute so __exit__ can access it on the same object
- Line 5: returning self allows the caller to write "with Timer() as t:" and access t.startTime if needed
- Line 6: __exit__ is called immediately when the with block exits, so the elapsed measurement is accurate
- Line 7: elapsed time is computed as the difference between the current time and the stored start time
- Line 8: the format specifier :.4f prints four decimal places for readable sub-second precision
- Line 9: returning False ensures exceptions from the timed block still propagate; Timer never swallows errors
- Line 10: the "as" clause is omitted here since the Timer object is not needed inside the block
- Line 11: sum(range(1_000_000)) is a CPU-bound operation; the Timer measures its actual wall-clock duration
- Line 12: Sum prints before Elapsed because __exit__ runs after the block body completes

---

## Example 11.4 — managedTransaction: Generator-Based Context Manager

```python
# Example 11.4
from contextlib import contextmanager                              # line 1

@contextmanager                                                    # line 2
def managedTransaction(connectionUrl):                             # line 3
    connection = f"Connection({connectionUrl})"                    # line 4
    print(f"Transaction started on {connectionUrl}")               # line 5
    try:                                                           # line 6
        yield connection                                           # line 7
    finally:                                                       # line 8
        print("Committing and closing connection")                 # line 9

with managedTransaction("db://prod-server") as connection:         # line 10
    print(f"Using {connection}")                                   # line 11
    print("Writing data to database")                              # line 12
```

**Output:**
```
Transaction started on db://prod-server
Using Connection(db://prod-server)
Writing data to database
Committing and closing connection
```

**Explanation:**
- Line 1: contextmanager is a decorator from the standard library that turns a generator into a context manager
- Line 2: @contextmanager transforms managedTransaction into an object whose __enter__ and __exit__ are generated automatically
- Line 3: managedTransaction is a regular function that receives any configuration arguments needed for setup
- Line 4: setup work happens before the yield; this is the equivalent of __enter__ body
- Line 5: the transaction start message corresponds to the "Connection opened" print in the class-based approach
- Line 6: the try block wraps the yield so that finally always executes even when the with block raises
- Line 7: yield is the boundary between __enter__ and __exit__; the yielded value becomes the "as" variable
- Line 8: finally guarantees the cleanup code runs whether the with block returned normally or raised an exception
- Line 9: this line is the equivalent of __exit__ body — it always executes
- Line 10: managedTransaction("db://prod-server") returns a context manager object; "as connection" binds the yielded value
- Line 11: connection holds the string yielded on line 7; the with block runs between the yield and the finally
- Line 12: after this line executes, the with block ends and the finally block on line 8 runs

