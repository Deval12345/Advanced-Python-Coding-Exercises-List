# Lecture 30: Race Conditions and Deadlock in Concurrent Python
## speech_30.md — Instructor Spoken Narrative

---

Every concurrent system — every program where two or more things happen simultaneously — has the potential for two fundamental failure modes. Today we examine both: race conditions and deadlock. Not as abstract theory; as concrete bugs with concrete symptoms, concrete causes, and concrete fixes.

Let me start with a question. If two threads both increment a counter at the same time, do you get the correct answer? The instinctive answer is yes — incrementing is a simple operation. The correct answer is no — and here is exactly why.

Incrementing a counter is not one operation. It is three: read the current value, add one, write the new value back. These three steps are not atomic. Between the read and the write, another thread can execute its own read. Thread A reads 5. Thread B reads 5. Thread A computes 6 and writes. Thread B computes 6 and writes. Both threads incremented, but the counter reads 6, not 7. One increment is permanently lost.

This is the race condition. The bug is that two threads raced to complete their read-modify-write cycles, and the one that completed last simply overwrote the other's result. The data structure is now incorrect; no error was raised, no exception was thrown, no crash occurred. The corruption is silent.

What makes race conditions particularly dangerous is that they are non-deterministic. Under light load, with one thread running, everything works correctly. Under heavy load, with many threads, the probability of two threads catching the same window simultaneously increases with each iteration. In testing, the load may never be high enough to trigger the race. In production, under real traffic, the race condition appears for the first time. The bug "works on my machine" is often a race condition that was never triggered in development.

Example 30.1 makes this visible. UnsafeCounter deliberately widens the race window with a tiny sleep between the read and the write. With 10 threads each doing 100 increments, the expected counter value is 1,000. With the race window open, you will consistently see values like 850 or 720 — sometimes dramatically less. SafeCounter wraps the entire read-modify-write sequence in a lock. With the lock, the final value is always exactly 1,000.

The fix is threading.Lock. A lock is a mutual exclusion primitive: only one thread can hold it at a time. When a thread acquires a lock and another thread tries to acquire the same lock, the second thread blocks — it waits — until the first releases it. By putting the read-modify-write sequence inside a with-lock block, you make the sequence atomic from the perspective of all other threads. No other thread can enter the same block while one is executing it.

The with statement is critical here. The pattern `with self._lock:` acquires the lock at the start of the block and releases it at the end — even if an exception is raised inside the block. The alternative — lock.acquire at the top, lock.release at the bottom — fails silently when an exception occurs: the release is never called, the lock is never returned, and every subsequent thread that tries to acquire it blocks forever. Always use the context manager. Always.

Now let us talk about deadlock. A deadlock is a situation where two threads are each waiting for a resource held by the other — and neither can proceed. Thread A holds lock L1 and is waiting to acquire lock L2. Thread B holds lock L2 and is waiting to acquire lock L1. Neither can release what it holds because it is blocked waiting. Both are frozen. Forever.

Deadlock is insidious for a different reason than race conditions. Race conditions produce wrong answers. Deadlock produces no answer. The threads simply stop making progress — silently, with no error, no exception, no crash. In a web server, a deadlocked request never completes. The client waits. The server holds the thread resources for that request indefinitely. Eventually, enough requests deadlock and the server runs out of available threads, at which point new requests can no longer be handled. The server appears to freeze progressively under load.

Example 30.2 shows the deadlock scenario structurally. taskDeadlock1 acquires lockA, sleeps, then waits for lockB. taskDeadlock2 acquires lockB, sleeps, then waits for lockA. If both start simultaneously, task1 holds A and waits for B; task2 holds B and waits for A. Circular dependency. Deadlock. We do not execute this deadlock in the example — we show the fix instead.

The fix is consistent lock ordering. If every thread in the system always acquires locks in the same global order — always A before B, never B before A — then circular dependencies cannot form. If task1 and task2 both acquire A before B, then whichever task gets A first holds it while acquiring B, and the other task waits for A. No circular wait. No deadlock.

This is not just a Python pattern. The Linux kernel maintains a formal lock-ordering graph and validates it at runtime in debug builds. Database systems order row locks by transaction ID to prevent deadlock between competing transactions. The discipline is universal: document the lock ordering, enforce it in code review, and never deviate from it.

Let me briefly address two more coordination primitives that appear in production code.

threading.RLock is a reentrant lock — a lock that the same thread can acquire multiple times. A regular Lock deadlocks a thread with itself if it tries to acquire a lock it already holds. RLock tracks which thread holds it and allows that thread to acquire it again without blocking. The lock is only fully released when the acquisition count reaches zero. You need this in recursive algorithms or class hierarchies where a method calls another method that might acquire the same lock.

threading.Condition solves the producer-consumer coordination problem without busy-waiting. A Condition wraps a lock. A thread that needs to wait for a condition calls condition.wait inside a while loop — this atomically releases the lock, suspends the thread, and re-acquires the lock when woken. A thread that changes the relevant state calls condition.notify to wake one waiter, or condition.notify_all to wake all of them. The waiting thread wakes, re-acquires the lock, checks the condition again in the while loop, and proceeds when it is satisfied.

This is how Python's queue.Queue works internally — a Condition for the not-empty state, a Condition for the not-full state. When a producer puts an item in, it notifies the not-empty Condition. Any consumer waiting on not-empty wakes up and takes an item. When a consumer takes an item, it notifies the not-full Condition. Any producer waiting on not-full wakes up and adds an item. Efficient, correct, no busy-waiting.

The core discipline of this lecture: if shared state exists, protect it with a lock. Use the with statement — never acquire without a guaranteed release path. For multiple locks, define and enforce a global acquisition order. For waiting on state changes, use Condition rather than polling.
