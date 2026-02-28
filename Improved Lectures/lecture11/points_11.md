Slide 0
Title: Welcome back

Point 1: Previous lecture: decorators wrap function calls non-invasively, injecting behavior before and after without modifying the function itself.

Point 2: Today: context managers wrap code blocks the same way, guaranteeing setup and teardown around any block of code regardless of whether it raises an exception.

Slide 1
Title: The Resource Leak Problem

Point 1: Without the with statement, an exception before a close call causes a resource leak; in high-traffic systems this exhausts connection pools and brings down services.

Point 2: try/finally solves leaks but is verbose and only a convention — developers under pressure skip it; the with statement makes cleanup structural and impossible to forget.

Slide 2
Title: The with Statement and __enter__ / __exit__

Point 1: Python calls __enter__ to set up the resource and __exit__ to tear it down; there is no execution path through a with block that bypasses __exit__.

Point 2: The value returned by __enter__ is bound to the "as" variable; __exit__ receives exceptionType, exceptionValue, and tracebackInfo describing any exception that escaped the block.

Slide 3
Title: __exit__ Return Value and Exception Suppression

Point 1: Returning True from __exit__ suppresses the exception; execution continues after the with block as if nothing happened.

Point 2: Returning False or None lets the exception propagate normally; this allows surgical suppression of specific exception types while letting unexpected errors through.

Slide 4
Title: Timer — Structural Instrumentation

Point 1: Recording start time in __enter__ and elapsed time in __exit__ guarantees measurement even when the timed block raises an exception.

Point 2: Instrumentation belongs structurally in a context manager rather than as manual start/end bookkeeping scattered throughout production code.

Slide 5
Title: @contextmanager — Generator Shortcut

Point 1: Decorating a generator function with @contextmanager turns it into a context manager; code before yield is setup, code after yield is teardown, and try/finally around the yield guarantees the finally block always runs.

Point 2: The generator form is more concise for simple cases; the class-based form is preferred when full control over __exit__ exception inspection or selective suppression is required.

Slide 6
Title: -

Point 1: Context managers make resource cleanup structural: __enter__ sets up, __exit__ tears down, and the guarantee is absolute; every file, connection, lock, and timing instrument in Python benefits from this design.

