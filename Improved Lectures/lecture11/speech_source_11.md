# Speech Source — Lecture 11: Context Managers — Structural Cleanup and Resource Lifecycle

---

## CONCEPT 0.1 — Transition from Decorators

In the previous lecture, we used decorators to add behavior around functions non-invasively. Decorators inject code before and after function calls — they wrap a function and return a new one with augmented behavior. Now we need something similar but for code blocks, not function calls. We need a way to set up a resource before a block of code runs, and guarantee cleanup when the block finishes — whether it finishes normally or throws an exception. That is what context managers do. They are the structural equivalent of a decorator, but for code blocks rather than function definitions.

---

## CONCEPT 1.1 — The Resource Leak Problem

Consider opening a database connection. Without context managers, the pattern is straightforward: open the connection, run some code, then close it. But what if the code raises an exception? The close call never runs. The connection leaks. In a high-traffic system with thousands of requests, leaked connections exhaust the connection pool. The system becomes unresponsive. The only fix is a restart.

The defensive pattern is try/finally. You write try, put your code inside, then write finally, put cleanup inside. The finally block always runs, even if an exception occurs. But this is verbose, error-prone, and easy to forget. If you have five resources to clean up, you write five try/finally blocks. Nested try/finally becomes unreadable. Worse, it is a convention — there is nothing in the language forcing you to use it. Developers under deadline pressure skip it. The resource leaks.

Python developers working on file I/O frequently left files open due to missing f.close() calls in exception paths. This class of bug was so common and so damaging that Python introduced the with statement in version 2.5 via PEP 343 specifically to eliminate it. In enterprise systems, unclosed database connections were a major source of production outages. Connection pools would exhaust. Systems would hang. Memory would leak. The with statement made cleanup impossible to forget by making it structural.

---

## CONCEPT 1.2 — The with Statement Design

The with statement is Python's solution. It says: enter this context, do this work, and when the work is done, always clean up. Python guarantees the cleanup. You cannot forget it because it is part of the syntax.

When Python encounters a with statement, it calls the __enter__ method on the context object. This method sets up the resource and returns it. The "as" clause binds the returned value to a name. Then Python runs the indented block. When the block finishes — whether normally or with an exception — Python calls the __exit__ method. The __exit__ method performs the cleanup. The guarantee is absolute. There is no execution path through a with block that bypasses __exit__.

---

## EXAMPLE 1.1 — DatabaseConnection Context Manager

Here we define a class named DatabaseConnection.

Inside the special dunder method enter, we print a connection message and return self.

Inside the special dunder method exit, we accept three parameters: exceptionType, exceptionValue, and tracebackInfo. We print a disconnection message. If exceptionType is not None, a connection error occurred and we print a rollback message. We return False, which means exceptions are not suppressed — they propagate normally after cleanup.

When we use this class with the "with" keyword and the "as" keyword naming it connection, the enter method runs first. Our code inside the block runs next. Then exit runs automatically, regardless of whether an exception occurred inside the block. The connection is always released.

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

---

## CONCEPT 2.1 — The __exit__ Return Value and Exception Policy

The __exit__ method receives three pieces of information about any exception that occurred inside the with block: the exception type, the exception value, and the traceback. If no exception occurred, all three are None.

If __exit__ returns True, the exception is suppressed. Execution continues after the with block as if nothing happened. If __exit__ returns False or None, the exception propagates normally. This gives context managers precise control over exception handling.

This capability is powerful. A context manager can selectively suppress specific exception types while letting others propagate. It can log an error, clean up, and decide whether the calling code should see the exception or not. This is a clean separation of cleanup logic from business logic.

---

## EXAMPLE 2.1 — DataImporter that Skips Bad Rows

Here we define a class named DataImporter.

The special dunder method enter prints a session start message and returns self.

The special dunder method exit checks whether the exception type is ValueError. If it is, we print a message about skipping a bad row and return True, which suppresses the exception. Otherwise, we print a cleanup message and return False, which allows the exception to propagate.

This allows a data processing loop to continue after individual bad rows without crashing the entire import. Each row is processed inside its own with block. ValueError exceptions from malformed data are swallowed; unexpected exceptions like network errors or out-of-memory errors propagate and halt the import.

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

---

## CONCEPT 3.1 — Timer Context Manager

A very practical use of context managers is timing code blocks. Instead of manually recording start and end times around every block you want to measure, you wrap the block in a Timer context manager. The setup and teardown are handled structurally.

This is a perfect fit for context managers: the start time is recorded in __enter__, the elapsed time is computed in __exit__, and the measurement happens regardless of whether the timed block raised an exception. You get accurate timing even for failing code paths. This is exactly the kind of instrumentation that belongs in a context manager rather than scattered as manual bookkeeping throughout production code.

---

## EXAMPLE 3.1 — Timer with Guaranteed Timing Even on Failure

Here we define a class named Timer.

Inside the special dunder method enter, we record the start time in a variable named startTime using time.time() and return self.

Inside the special dunder method exit, we compute the elapsed time as the current time minus startTime and print it. We return False, so exceptions propagate normally.

Every time the block inside "with Timer()" runs, the elapsed time is measured and printed, whether the block succeeds or raises an exception. The measurement is guaranteed.

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

---

## CONCEPT 4.1 — The contextlib.contextmanager Decorator

Python provides a shortcut for writing context managers. Instead of defining a class with __enter__ and __exit__, you write a generator function and decorate it with @contextmanager from contextlib.

The mechanics map directly to the class-based approach. The code before yield corresponds to __enter__ — it runs when the with block is entered. The yield statement produces the value that the "as" variable receives, just as __enter__'s return value does. The code after yield corresponds to __exit__ — it runs when the with block exits. A try/finally around the yield ensures the cleanup code in the finally block runs even when an exception propagates through the yield.

This approach is significantly more concise than the class-based approach for simple cases. For context managers that do not need to inspect exception details or selectively suppress exceptions, the generator form is preferred. The class-based form is better when you need full control over the __exit__ logic.

---

## EXAMPLE 4.1 — managedTransaction Using @contextmanager

Here we import contextmanager from contextlib.

We define a generator function named managedTransaction that receives a parameter named connectionUrl.

Inside, we create a connection variable and print a transaction start message.

We use try/yield/finally: we yield the connection so the caller can use it inside the with block, and in the finally block we commit and close the connection.

When used with the "with" keyword, the caller gets the connection, uses it, and the transaction is committed and the connection closed automatically — whether the block succeeded or raised an exception.

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

---

## CONCEPT 5.1 — Final Takeaway

Context managers make resource cleanup structural. They guarantee that __exit__ runs regardless of what happens inside the block. The __enter__ method sets up; the __exit__ method tears down. Exception suppression is explicit and controlled — returning True swallows an exception, returning False lets it propagate.

The contextlib.contextmanager decorator turns a generator function into a context manager using yield as the boundary. The code before yield is setup; the code after yield is teardown; the finally clause guarantees the teardown always runs.

Every file, database connection, network socket, lock, or timing instrument in Python benefits from context manager design. When you see "with" in production code, you see a guarantee of cleanup. Django's database layer uses context managers for transactions. The threading module wraps locks with context managers so they are always released. The tempfile module uses context managers for temporary files that must be deleted. This pattern is ubiquitous because it is correct.

In the next lecture, we will look at how Python's exception model supports a complementary philosophy — EAFP, where you assume success and handle failure explicitly, rather than checking preconditions defensively.

---
