# EAFP and the Exception Model in Python

## Lesson Opening

In the previous session we used context managers to guarantee cleanup. That relied on the fact that when an exception is raised, Python still runs the exit method. So exceptions are not just errors; they are part of the control flow. Python encourages a style called EAFP: easier to ask forgiveness than permission.

Today we look at what that means. Instead of checking every condition before acting, you try the action and handle the exception if it fails. We will see why this leads to clearer code and why it matters in concurrent and real-world systems.

By the end of this lesson you will understand EAFP versus look-before-you-leap, how exceptions separate the happy path from failure handling, and when to use try and except in production code.

------------------------------------------------------------------------

## Section 1 — Two Styles: EAFP and LBYL

### CONCEPT 0.1

Context managers and exit showed that exceptions are normal. Python’s design assumes you will use exceptions for failure paths. So the main logic stays linear and readable. Failure handling lives in except blocks. That is the EAFP style.

### CONCEPT 1.1

Look before you leap means you check that a key exists, that a type is correct, that a value is in range, and only then do the operation. That leads to nested conditionals and defensive code everywhere. EAFP means you do the operation and if it raises an exception, you handle it. So the happy path is one straight line. The exception path is separate. That is easier to read and often safer in the presence of concurrency, because the state cannot change between the check and the use.

------------------------------------------------------------------------

## Section 2 — Why EAFP Fits Python

### CONCEPT 2.1

In Python, operations raise exceptions when something goes wrong. Accessing a missing key raises an error. Converting an invalid string to int raises an error. So the language already gives you a clear signal. Catching that signal in an except block keeps the main flow clean. You do not need to pre-check every possibility. You try, and you handle the cases that fail.

### EXAMPLE 2.1

Here we have a list of response dictionaries. Some have a user key and an amount key; some are malformed. We loop over the list. Inside the loop we try to process the user and amount. If the key is missing, we catch the exception and skip that response with a message. So valid data is processed. Invalid data is isolated. We did not write nested if checks for every key. We tried and asked for forgiveness. The code is shorter and the intent is clear.

------------------------------------------------------------------------

## Section 3 — Defensive Code Versus EAFP

### CONCEPT 3.1

Defensive code that checks type and presence and range before every step becomes hard to maintain. EAFP lets you assume the data is valid and handle the rare case when it is not. That inverts the structure: the common case is one block; the rare case is in the except. That matches how we think about production systems. Most requests succeed; we handle failures in one place.

### EXAMPLE 3.1

Here we define a function named parseRequest that takes a parameter named data. We try to get the user id from data and convert it to an integer, and we get an optional discount code. We return a tuple of user id and discount code. If a key is missing or the value cannot be converted, we catch the exception and return None. So the function has a single try block. Callers get either a result or None. They do not need to know whether the failure was a missing key or a bad type. The exception model hides the detail and exposes a simple contract.

------------------------------------------------------------------------

## Section 4 — Exceptions as Control Flow

### CONCEPT 4.1

Exceptions are not only for crashes. They are the standard way to signal that an operation could not be completed. So using try and except is normal Python style. You should catch specific exception types when you know what can go wrong, and let unexpected exceptions propagate so they are not hidden. That keeps the code predictable and debuggable.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 5.1

EAFP means try the operation and handle the exception. It keeps the happy path linear and puts failure handling in except blocks. It avoids race conditions that can occur when you check then use in separate steps. In Python, exceptions are the normal mechanism for failure. Use try and except for expected failures; catch specific types; let unexpected errors propagate. This style is central to readable and robust Python code.
