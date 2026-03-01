# Slides — Lecture 30: Race Conditions and Deadlock in Concurrent Python

---

## Slide 1 — Lecture Overview

**The Fundamental Failure Modes of Concurrent Code**

- Race conditions: non-deterministic data corruption from uncoordinated shared access
- Demonstrating a race with threading: the lost-update pattern
- threading.Lock: making read-modify-write atomic
- Deadlock: two threads each holding what the other needs
- Deadlock prevention: consistent global lock ordering
- threading.RLock and threading.Condition for advanced patterns

---

## Slide 2 — What a Race Condition Is

**Non-deterministic corruption from uncoordinated read-modify-write**

- Increment counter = THREE steps: read, compute, write
- These steps are NOT atomic — another thread can interrupt between them

```
Thread A: read 5
Thread B: read 5   ← both read the same value
Thread A: write 6
Thread B: write 6  ← overwrites A's result
Counter = 6, not 7. One increment LOST.
```

- "Lost update" — the silent failure mode
- No exception, no crash, no error message — just wrong data
- Non-deterministic: may not appear in testing, appears under production load

---

## Slide 3 — Why Race Conditions Are Hard to Find

**Timing-dependent, non-reproducible, load-sensitive**

- Race window: nanoseconds between read and write
- Low load (1 thread): no race possible — always correct
- Medium load (few threads): occasional races — intermittent failures
- High load (many threads, fast machine): constant races — systematic failures
- The "works on my machine" bug is often a race condition
- Testing rarely reproduces production load → races appear in production only
- Fix must be structural (locking) — not based on "it hasn't failed yet"

---

## Slide 4 — threading.Lock: Making Read-Modify-Write Atomic

**Mutual exclusion primitive — only one thread holds it at a time**

```python
class SafeCounter:
    def __init__(self):
        self.value = 0
        self._lock = threading.Lock()
    
    def increment(self, amount):
        with self._lock:           # acquire
            current = self.value   # read  ← now atomic as a group
            self.value = current + amount  # write
                                   # release (guaranteed, even on exception)
```

- Lock blocks any other thread trying to acquire it — they wait
- `with self._lock:` — context manager guarantees release on all exit paths
- Never use `lock.acquire()` + `lock.release()` directly — exception safety

---

## Slide 5 — Example 30.1: Race Condition Demonstration

**Unsafe vs. Safe counter — visible data corruption**

- 10 threads × 100 increments each = expected value: 1000
- UnsafeCounter: `time.sleep(0.00001)` between read and write widens the race window
- Unsafe result: typically 700–950 (lost = 50–300 increments)
- SafeCounter: `with self._lock:` wraps the full read-modify-write
- Safe result: always exactly 1000

Key insight: the sleep exaggerates the window; production code without sleep still has the window — it just closes faster, making the race less frequent but not impossible.

---

## Slide 6 — What Deadlock Is

**Circular dependency — threads hold what the other needs**

```
Thread A: holds lockA, waiting for lockB
Thread B: holds lockB, waiting for lockA
```

- Neither can proceed — neither can release what it holds while waiting
- No error, no exception — threads simply stop making progress
- Silent: requests never complete, connections are never closed
- In a web server: deadlocked threads accumulate → server runs out of threads
- Recovery requires killing one of the deadlocked threads externally

---

## Slide 7 — The Four Conditions for Deadlock (Coffman, 1971)

**Eliminate ANY ONE condition to prevent deadlock**

1. **Mutual exclusion** — resources cannot be shared (locks are exclusive)
2. **Hold and wait** — thread holds a resource while waiting for another
3. **No preemption** — locks cannot be forcibly taken from a holder
4. **Circular wait** — a cycle of threads each waiting for the next

- In Python threading, conditions 1-3 are inherent to the Lock design
- We can eliminate condition 4: **consistent lock ordering**
- If no circular dependency can form → no deadlock possible

---

## Slide 8 — Example 30.2: Deadlock vs. Safe Ordering

**Inconsistent ordering creates circular wait — consistent ordering prevents it**

```python
# DEADLOCK RISK: inconsistent ordering
def task1(): lockA → lockB    # holds A, waits for B
def task2(): lockB → lockA    # holds B, waits for A → CIRCULAR

# SAFE: consistent ordering — always lockA before lockB
def taskSafe1(): lockA → lockB
def taskSafe2(): lockA → lockB
# task2 waits for lockA while task1 holds it — then proceeds
```

- Consistent ordering: any thread that reaches lockB already released or never held a lock that the lockA-waiter needs
- The rule: define a global lock hierarchy. Document it. Enforce it in review.

---

## Slide 9 — threading.RLock: Reentrant Locking

**A lock a thread can acquire multiple times without deadlocking itself**

- `threading.Lock`: if thread holds lock and tries to acquire again → self-deadlock
- `threading.RLock`: tracks the holding thread's identity; same thread can acquire N times
- Only fully released when acquisition count returns to zero
- Use case: recursive functions; class hierarchies where a public method calls another method that acquires the same lock

```python
class RecursiveProcessor:
    def __init__(self):
        self._lock = threading.RLock()
    
    def process(self, data):
        with self._lock:
            if isinstance(data, list):
                return [self.process(item) for item in data]  # re-acquires _lock
            return data * 2
```

---

## Slide 10 — threading.Condition: Wait-and-Notify

**Coordinated waiting on state changes — no busy-waiting**

```python
condition = threading.Condition()

# Consumer (waiter):
with condition:
    while not data_available:      # always loop — not if
        condition.wait()           # atomically: release lock + suspend thread
    item = shared_queue.pop()

# Producer (notifier):
with condition:
    shared_queue.append(item)
    condition.notify()             # wake one waiter
```

- `wait()`: atomically releases the lock and suspends the thread
- `notify()`: wakes one waiting thread; it re-acquires the lock
- Always loop on the condition — spurious wakeups can occur
- This is the internal mechanism of `queue.Queue`

---

## Slide 11 — Summary: The Concurrency Safety Rules

**The four rules that prevent both race conditions and deadlock**

1. **Protect shared mutable state with a Lock** — no exceptions for "quick reads"
2. **Use `with lock:` always** — never bare acquire/release (exception safety)
3. **Acquire multiple locks in a consistent global order** — document and enforce
4. **Use Condition for wait-until-state-changes** — never poll shared state in a loop

- RLock for recursive/reentrant lock requirements
- Condition for producer-consumer coordination
- These rules are non-negotiable in production concurrent code

---

## Slide 12 — Lecture 30 Key Principles

**What to carry forward**

- Race condition = two threads do read-modify-write without coordination → lost update
- Non-deterministic: appears under load, invisible in light testing
- `threading.Lock` + `with` statement = atomic critical section
- Deadlock = circular lock dependency → threads freeze silently
- Prevention: always acquire locks in the same global order (consistent ordering)
- `threading.RLock`: reentrant — same thread can acquire N times
- `threading.Condition`: atomic wait-and-notify for state changes
- Next: concurrency architecture patterns — fan-out/fan-in, circuit breakers, three-tier design

---
