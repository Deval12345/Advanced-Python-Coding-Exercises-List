# Slides — Lecture 11: Context Managers — Structural Cleanup and Resource Lifecycle

---

## Slide 1
**Title:** From Decorators to Context Managers

- L10 used decorators to wrap function calls with setup and teardown behavior
- Decorators operate at the function level — they transform a callable
- Context managers operate at the block level — they wrap an indented code block
- Both provide the same guarantee: setup runs before, teardown always runs after

---

## Slide 2
**Title:** The Resource Leak Problem

- Opening a connection and then throwing an exception means the close call never runs
- The connection leaks; in high-traffic systems, the connection pool exhausts
- try/finally solves it but is verbose, easy to forget, and only a convention
- Python introduced the with statement in version 2.5 via PEP 343 to make cleanup structural

---

## Slide 3
**Title:** The with Statement Design

- Python calls __enter__ on the context object to set up the resource
- The "as" clause binds the value returned by __enter__ to a name
- Python runs the indented block
- When the block finishes — normally or by exception — Python calls __exit__ to clean up
- The guarantee is absolute: no execution path through a with block bypasses __exit__

---

## Slide 4
**Title:** Example 11.1 — DatabaseConnection Context Manager

- __enter__ prints "Connection opened" and returns self
- __exit__ receives exceptionType, exceptionValue, and tracebackInfo
- If exceptionType is not None, a rollback message is printed
- __exit__ returns False so exceptions propagate normally after cleanup
- with DatabaseConnection() as connection: guarantees the connection is always released

---

## Slide 5
**Title:** The __exit__ Return Value and Exception Policy

- __exit__ receives three arguments: exceptionType, exceptionValue, tracebackInfo
- If no exception occurred inside the block, all three are None
- Return True: the exception is suppressed; execution continues after the with block
- Return False or None: the exception propagates normally
- Context managers can selectively suppress specific exception types and let others through

---

## Slide 6
**Title:** Example 11.2 — DataImporter that Skips Bad Rows

- __exit__ checks whether exceptionType is ValueError
- If it is, prints a skip message and returns True — the exception is suppressed
- Otherwise, prints a cleanup message and returns False — the exception propagates
- Each row is processed in its own with block; bad rows are skipped, unexpected errors halt
- Exception suppression is explicit and surgical, not a blanket catch-all

---

## Slide 7
**Title:** Timer Context Manager — Practical Instrumentation

- Context managers are ideal for timing code blocks without manual bookkeeping
- __enter__ records the start time; __exit__ computes and prints elapsed time
- The measurement is guaranteed even if the timed block raises an exception
- Timing logic lives in one place rather than scattered as manual start/end calls throughout code
- This is the instrumentation pattern that belongs structurally, not procedurally

---

## Slide 8
**Title:** Example 11.3 — Timer with Guaranteed Measurement

- time.time() is called in __enter__ and stored as self.startTime
- __exit__ computes elapsed time as time.time() minus self.startTime
- The elapsed time is printed formatted to four decimal places
- __exit__ returns False so exceptions are not suppressed
- Every block wrapped by Timer() reports its duration, success or failure

---

## Slide 9
**Title:** contextlib.contextmanager — The Generator Approach

- Python's contextlib provides a shortcut: decorate a generator function with @contextmanager
- Code before the yield corresponds to __enter__ — it runs on entry
- The yield value is what the "as" variable receives, just as __enter__'s return value is
- Code after the yield corresponds to __exit__ — it runs on exit
- A try/finally around the yield ensures the finally block runs even when an exception propagates

---

## Slide 10
**Title:** Example 11.4 — managedTransaction Using @contextmanager

- managedTransaction(connectionUrl) creates a connection and prints a start message
- try/yield/finally: yield hands the connection to the with block; finally commits and closes
- The caller gets the connection, uses it, and cleanup is automatic
- For simple cases, the generator form is more concise than the class-based form
- The class-based form is preferred when selective exception suppression is needed in __exit__

---

## Slide 11
**Title:** Context Managers in Industry

- Django's database layer uses context managers for transactions — commit or rollback is guaranteed
- threading.Lock implements __enter__ and __exit__ so locks are always released
- tempfile.NamedTemporaryFile uses a context manager so temporary files are always deleted
- Python's built-in open() is a context manager — file handles are always closed
- contextlib.suppress is a one-line context manager that suppresses specified exception types

---

## Slide 12
**Title:** Summary

- Context managers guarantee that __exit__ runs regardless of what happens inside the block
- __enter__ sets up the resource; __exit__ tears it down; cleanup is structural, not optional
- Return True from __exit__ to suppress an exception; return False to let it propagate
- @contextmanager turns a generator function into a context manager using yield as the boundary
- Next lecture: Python's EAFP philosophy — assume success and handle failure explicitly

