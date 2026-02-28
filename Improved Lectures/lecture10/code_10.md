# Code — Lecture 10: Decorators — Behavior Transformation Without Code Modification

---

## Example 10.1 — logCall: Basic Decorator

```python
# Example 10.1
import functools                                        # line 1

def logCall(targetFunction):                           # line 2
    def wrapper(*args, **kwargs):                      # line 3
        print(f"Calling: {targetFunction.__name__}")   # line 4
        result = targetFunction(*args, **kwargs)       # line 5
        return result                                  # line 6
    return wrapper                                     # line 7

@logCall                                               # line 8
def processPayment(amount, currency):                  # line 9
    return f"Processed {amount} {currency}"            # line 10

output = processPayment(100, "USD")                    # line 11
print(output)                                          # line 12
```

**Output:**
```
Calling: processPayment
Processed 100 USD
```

**Explanation:**
- Line 2: logCall receives the function to be decorated as targetFunction
- Line 3: wrapper accepts *args and **kwargs to forward any arguments transparently
- Line 4: the log message uses targetFunction's name, not a hardcoded string
- Line 5: targetFunction is called with the original arguments; result is captured
- Line 6: the result is returned so the caller gets the correct value
- Line 7: logCall returns wrapper, which becomes the new binding for processPayment
- Line 8: @logCall is equivalent to: processPayment = logCall(processPayment)
- Line 11: calling processPayment now calls wrapper, which logs then delegates

---

## Example 10.2 — measureTime with functools.wraps

```python
# Example 10.2
import time                                                          # line 1
import functools                                                     # line 2

def measureTime(targetFunction):                                     # line 3
    @functools.wraps(targetFunction)                                 # line 4
    def wrapper(*args, **kwargs):                                    # line 5
        startTime = time.perf_counter()                              # line 6
        result = targetFunction(*args, **kwargs)                     # line 7
        endTime = time.perf_counter()                                # line 8
        elapsedSeconds = endTime - startTime                         # line 9
        print(f"{targetFunction.__name__} took {elapsedSeconds:.6f}s") # line 10
        return result                                                # line 11
    return wrapper                                                   # line 12

@measureTime                                                         # line 13
def computeSum(upperBound):                                          # line 14
    return sum(range(upperBound))                                    # line 15

total = computeSum(1_000_000)                                        # line 16
print(f"Result: {total}")                                            # line 17
print(f"Function name: {computeSum.__name__}")                       # line 18
```

**Output:**
```
computeSum took 0.031247s
Result: 499999500000
Function name: computeSum
```

**Explanation:**
- Line 4: @functools.wraps(targetFunction) copies __name__, __doc__, and __module__ to wrapper
- Line 6: perf_counter provides a high-resolution timer; stored before the call
- Line 7: targetFunction is called with all forwarded arguments; result captured
- Line 8: end time recorded immediately after the call completes
- Line 9: elapsed time computed as the difference
- Line 10: the log message uses targetFunction.__name__ which equals "computeSum"
- Line 18: computeSum.__name__ prints "computeSum", not "wrapper", because of functools.wraps

---

## Example 10.3 — retryOnFailure with Configurable maxAttempts

```python
# Example 10.3
import functools                                                       # line 1
import random                                                          # line 2

def retryOnFailure(maxAttempts):                                       # line 3
    def decorator(targetFunction):                                     # line 4
        @functools.wraps(targetFunction)                               # line 5
        def wrapper(*args, **kwargs):                                  # line 6
            lastException = None                                       # line 7
            for attemptNumber in range(maxAttempts):                   # line 8
                try:                                                   # line 9
                    result = targetFunction(*args, **kwargs)           # line 10
                    return result                                      # line 11
                except Exception as caughtException:                   # line 12
                    lastException = caughtException                    # line 13
                    remainingAttempts = maxAttempts - attemptNumber - 1 # line 14
                    print(f"Attempt {attemptNumber + 1} failed. "
                          f"{remainingAttempts} remaining.")           # line 15
            raise lastException                                        # line 16
        return wrapper                                                 # line 17
    return decorator                                                   # line 18

@retryOnFailure(maxAttempts=3)                                         # line 19
def fetchDataFromServer(serverUrl):                                    # line 20
    if random.random() < 0.7:                                          # line 21
        raise ConnectionError(f"Could not connect to {serverUrl}")    # line 22
    return f"Data from {serverUrl}"                                    # line 23

try:                                                                   # line 24
    response = fetchDataFromServer("https://api.example.com")         # line 25
    print(response)                                                    # line 26
except ConnectionError as connectionError:                             # line 27
    print(f"All attempts failed: {connectionError}")                  # line 28
```

**Output (example — random behavior):**
```
Attempt 1 failed. 2 remaining.
Attempt 2 failed. 1 remaining.
Data from https://api.example.com
```

**Explanation:**
- Line 3: retryOnFailure receives configuration (maxAttempts) and returns a decorator
- Line 4: decorator receives the function to wrap; this is the actual decorator
- Line 5: functools.wraps applied here so the wrapped function keeps its identity
- Line 7: lastException stored so it can be re-raised after all attempts fail
- Line 8: loop runs exactly maxAttempts times
- Line 10: targetFunction called; if it succeeds, result is returned immediately on line 11
- Line 12: any exception is caught and stored as lastException
- Line 16: after the loop exhausts all attempts, the last captured exception is raised
- Line 18: retryOnFailure returns decorator
- Line 19: @retryOnFailure(maxAttempts=3) calls retryOnFailure first, gets decorator, applies it

---

## Example 10.4 — Access Control Decorator with Role Checking

```python
# Example 10.4
import functools                                                         # line 1

currentUser = {"name": "alice", "role": "editor"}                       # line 2

def requireRole(requiredRole):                                           # line 3
    def decorator(targetFunction):                                       # line 4
        @functools.wraps(targetFunction)                                 # line 5
        def wrapper(*args, **kwargs):                                    # line 6
            userRole = currentUser.get("role", "guest")                  # line 7
            userName = currentUser.get("name", "unknown")                # line 8
            if userRole != requiredRole:                                  # line 9
                raise PermissionError(                                   # line 10
                    f"User '{userName}' has role '{userRole}' "
                    f"but '{requiredRole}' is required."                 # line 11
                )
            print(f"Access granted to '{userName}' as '{userRole}'.")   # line 12
            return targetFunction(*args, **kwargs)                       # line 13
        return wrapper                                                   # line 14
    return decorator                                                     # line 15

@measureTime                                                             # line 16
@requireRole("admin")                                                    # line 17
def deleteRecord(recordId):                                              # line 18
    return f"Record {recordId} deleted."                                 # line 19

try:                                                                     # line 20
    print(deleteRecord(42))                                              # line 21
except PermissionError as permissionError:                               # line 22
    print(f"Blocked: {permissionError}")                                 # line 23

currentUser["role"] = "admin"                                            # line 24
print(deleteRecord(42))                                                  # line 25
```

**Output:**
```
Blocked: User 'alice' has role 'editor' but 'admin' is required.
Access granted to 'alice' as 'admin'.
deleteRecord took 0.000003s
Record 42 deleted.
```

**Explanation:**
- Line 2: currentUser simulates a session-level user context
- Line 3: requireRole receives the required role string and returns a decorator
- Line 7: the actual user role is read from currentUser at call time, not definition time
- Line 9: the role check runs before targetFunction is called; this is the pre-condition pattern
- Line 10: PermissionError raised if the role does not match; targetFunction never runs
- Line 12: access confirmation logged only when the check passes
- Line 13: targetFunction called and result returned only after the role check succeeds
- Line 16: @measureTime is the outer decorator; it wraps the already-requireRole-wrapped function
- Line 17: @requireRole("admin") is applied first (bottom-to-top), then measureTime wraps that
- Line 24: updating currentUser at runtime affects the next call because the check runs each time
- Decorator stacking: call order is measureTime wrapper first, then requireRole wrapper, then deleteRecord

