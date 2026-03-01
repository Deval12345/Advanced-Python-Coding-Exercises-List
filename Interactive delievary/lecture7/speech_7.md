In the previous session we used decorators to wrap functions and add behavior. Another common need is to run some setup before a block of code and guarantee cleanup after, whether the block finishes normally or raises an exception. That is what context managers and the with statement are for.

Today we look at how with works, and how to implement the special dunder method enter and the special dunder method exit so that your objects can be used after the word with. We will see why this matters for resources like files and locks.

By the end of this lesson you will understand the context manager protocol, when exit runs, and how to write a simple context manager for timing or cleanup.

Decorators wrap function calls. But sometimes the lifecycle is not a single call. You open a resource, do work, and you must close the resource on every path: when the work succeeds, when it fails, or when we return early. Manual try and finally works but is easy to forget. Context managers make cleanup structural so that exit code always runs.

When you open a file or acquire a lock, you must release it. If you forget to close or release on one path, you leak resources. Python’s with statement ensures that when you leave the block, whether by normal exit, return, or exception, a designated exit method runs. So cleanup is guaranteed, not optional. That is why with is used for files, database sessions, and locks.

When you write with an expression as a variable, Python evaluates the expression to get a context manager. It then calls the special dunder method enter on that object. The return value of enter is bound to the variable. The block runs. When the block ends or an exception occurs, Python calls the special dunder method exit. So enter is setup and exit is cleanup. Exit always runs, even if the block raised an exception.

The special dunder method exit receives information about any exception that occurred. It can return True to say that it handled the exception and the exception should not propagate. Returning False or None means the exception propagates. So you can decide inside exit whether to swallow a specific error or let it go up. Most of the time you return False so that real errors are not hidden.

A context manager does not have to manage a file or lock. It can measure time, switch configuration, or log the start and end of a block. The pattern is the same: enter runs first and can return something useful; exit runs when leaving the block.

Here we define a class named Timer. Inside it we implement the special dunder method enter. Enter records the start time in a variable and returns self so the block can use the timer if needed. We implement the special dunder method exit with parameters for exception type, value, and traceback. Exit computes the elapsed time and prints it. It returns False so that any exception in the block is not swallowed. Because Timer implements enter and exit, we can write with Timer and a block. When the block finishes, exit runs and we see the elapsed time. If an exception happens inside the block, exit still runs, so we still get the timing. Cleanup and logging are guaranteed.

The built-in open function returns a file object that is also a context manager. So using with and open and a filename is the standard way to read or write a file. When the block ends, the file is closed automatically. You do not need to call close yourself. This pattern is used everywhere in production code for files, sockets, and database connections.

Context managers guarantee cleanup. The with statement calls the special dunder method enter before the block and the special dunder method exit when leaving. Exit always runs. It can receive exception details and choose to suppress or propagate. Use with for resources that must be released and for any setup and cleanup pair. This keeps code safe and readable.
