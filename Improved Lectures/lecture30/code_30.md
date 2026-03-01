# Code — Lecture 30: Race Conditions and Deadlock in Concurrent Python

---

## Example 30.1 — Race condition demonstration and fix with threading.Lock

```python
# Example 30.1
import threading
import time

class UnsafeCounter:
    def __init__(self):
        self.value = 0
    
    def increment(self, amount):
        current = self.value      # read
        time.sleep(0.00001)       # simulate delay between read and write
        self.value = current + amount  # write -- race window here

class SafeCounter:
    def __init__(self):
        self.value = 0
        self._lock = threading.Lock()
    
    def increment(self, amount):
        with self._lock:
            current = self.value
            time.sleep(0.00001)
            self.value = current + amount

def runTest(counter, numThreads, incrementsPerThread):
    threads = [
        threading.Thread(target=lambda: [counter.increment(1) for _ in range(incrementsPerThread)])
        for _ in range(numThreads)
    ]
    for t in threads: t.start()
    for t in threads: t.join()

if __name__ == "__main__":
    numThreads = 10
    incrementsPerThread = 100
    expected = numThreads * incrementsPerThread
    
    unsafe = UnsafeCounter()
    runTest(unsafe, numThreads, incrementsPerThread)
    print(f"Unsafe counter: expected={expected}, got={unsafe.value} (lost={expected - unsafe.value})")
    
    safe = SafeCounter()
    runTest(safe, numThreads, incrementsPerThread)
    print(f"Safe counter: expected={expected}, got={safe.value} (correct: {safe.value == expected})")
```

**Line-by-line explanation:**

- **Lines 5-11:** `UnsafeCounter` has no synchronization. The increment method is a textbook read-modify-write that is not atomic.
- **Line 9:** `current = self.value` — Thread reads the current counter value into its local variable. This is the first step of the non-atomic sequence.
- **Line 10:** `time.sleep(0.00001)` — deliberately widens the race window to 10 microseconds. In production code there is no sleep here — the window is measured in nanoseconds — but it still exists. The sleep makes the race deterministic for demonstration.
- **Line 11:** `self.value = current + amount` — Thread writes back. If another thread read the same value as `current` between lines 9 and 11, its write will produce the same result, overwriting this thread's correct increment. One update is permanently lost.
- **Lines 13-20:** `SafeCounter` wraps the exact same read-modify-write in `with self._lock:`.
- **Line 18:** `with self._lock:` — acquires the lock. If another thread holds the lock, this thread blocks here until it is released.
- **Lines 19-20:** The read-modify-write is now protected. Only one thread can execute these two lines simultaneously. When the with block exits, the lock is released, allowing the next waiting thread to proceed.
- **Lines 22-29:** `runTest` creates N threads, each executing M increments. The lambda inside `Thread(target=...)` creates a closure over `counter` and `incrementsPerThread`. Note: all threads share the same counter object.
- **Line 24:** Lambda with list comprehension — `[counter.increment(1) for _ in range(incrementsPerThread)]` — calls increment M times. The lambda is required because Thread.target must be a callable with no arguments.
- **Lines 31-41:** Main block runs both tests with identical parameters.
- **Lines 36-37:** `unsafe` test: 10 threads × 100 increments = 1000 expected. Actual value will be less due to lost updates. `expected - unsafe.value` quantifies how many increments were lost.
- **Lines 39-40:** `safe` test: same parameters, but with lock. The final assertion `safe.value == expected` should always be True.

**Expected output:**
```
Unsafe counter: expected=1000, got=743 (lost=257)
Safe counter: expected=1000, got=1000 (correct: True)
```
The exact number of lost updates in the unsafe version varies by run — this is the non-determinism of race conditions.

---

## Example 30.2 — Deadlock demonstration and prevention via consistent lock ordering

```python
# Example 30.2
import threading
import time

lockA = threading.Lock()
lockB = threading.Lock()

def taskDeadlock1():
    with lockA:
        print("Task1: acquired lockA")
        time.sleep(0.1)
        print("Task1: waiting for lockB...")
        with lockB:
            print("Task1: acquired both locks")

def taskDeadlock2():
    with lockB:
        print("Task2: acquired lockB")
        time.sleep(0.1)
        print("Task2: waiting for lockA...")
        with lockA:
            print("Task2: acquired both locks")

def taskSafe1():
    with lockA:
        with lockB:
            print("SafeTask1: acquired lockA then lockB")
            time.sleep(0.05)

def taskSafe2():
    with lockA:
        with lockB:
            print("SafeTask2: acquired lockA then lockB")
            time.sleep(0.05)

if __name__ == "__main__":
    print("--- Safe execution (consistent ordering: always lockA first) ---")
    t1 = threading.Thread(target=taskSafe1)
    t2 = threading.Thread(target=taskSafe2)
    t1.start(); t2.start()
    t1.join(timeout=2); t2.join(timeout=2)
    print("Safe execution completed without deadlock")
    
    print("\n--- Deadlock risk (inconsistent ordering) ---")
    print("NOTE: This WOULD deadlock; shown for illustration only")
    print("Task1 acquires A then waits for B; Task2 acquires B then waits for A")
    print("Solution: always acquire locks in the same global order (A before B)")
```

**Line-by-line explanation:**

- **Lines 5-6:** `lockA` and `lockB` are module-level locks. In production, locks protecting different resources would be attributes of the resource objects they protect.
- **Lines 8-13:** `taskDeadlock1` acquires lockA first, then waits for lockB. The sleep between the two acquisitions is critical — it gives the scheduler time to switch to taskDeadlock2 so both tasks hold one lock each before either tries to acquire the second.
- **Lines 15-20:** `taskDeadlock2` acquires lockB first, then waits for lockA. This is the opposite order from taskDeadlock1 — the inconsistency that creates the potential deadlock.
- **Deadlock sequence:** t1 acquires lockA. Scheduler switches to t2. t2 acquires lockB. t1 resumes, tries to acquire lockB — blocks (t2 holds it). t2 tries to acquire lockA — blocks (t1 holds it). Both blocked. Circular dependency. Neither can proceed.
- **Lines 22-27:** `taskSafe1` and `taskSafe2` — identical lock ordering: always lockA then lockB.
- **Lines 22-27:** When t1 and t2 both use this order: one succeeds in acquiring lockA; the other blocks at line 23 trying to acquire lockA. The first thread then acquires lockB (no contention — the second thread is blocked before it ever tries to acquire lockB). First thread releases both. Second thread then acquires both in sequence. No cycle. No deadlock.
- **Lines 34-37:** The safe test runs to completion and joins within the 2-second timeout, confirming no deadlock occurred.
- **Lines 39-43:** The deadlock scenario is illustrated in comments rather than executed — running it would permanently hang the program. In a real demonstration, you would run the deadlock threads with a timeout join and show that join(timeout=2) returns with the threads still alive, confirming the deadlock.

**Key principle illustrated:**
The safe functions `taskSafe1` and `taskSafe2` are identical in their lock ordering even though they could safely differ — the principle is that the ordering convention must be consistent, not that it must be meaningful. Choosing alphabetical order (A before B), or definition order, or address order (`id(lockA) < id(lockB)`) — any total ordering works, as long as it is consistent across all code that acquires these locks together.
