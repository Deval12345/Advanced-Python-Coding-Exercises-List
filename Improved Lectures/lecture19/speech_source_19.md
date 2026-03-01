# Speech Source — Lecture 19: Threading for I/O-Bound Work — Thread Pools, Safety, and Practical Patterns

---

## CONCEPT 0.1 — Transition from Previous Lecture

In the last lecture we classified work as either I/O-bound or CPU-bound, and we surveyed three concurrency models available in Python: threading, multiprocessing, and async/await. Now we go deep on the first of those three — threading. Threads are the right tool for I/O-bound work, and Python's ThreadPoolExecutor makes them straightforward to use correctly. But threads introduce shared mutable state between concurrent execution paths, and shared state introduces race conditions. We need to understand both the power and the hazards before we write production threading code.

---

## CONCEPT 1.1 — Why Threading Works for I/O-Bound Work

**Problem it solves:** Sequential I/O is wasteful. When your program is waiting for a network response, the CPU is idle. If you have 100 URLs to fetch, fetching them one at a time means your program spends most of its runtime blocked, doing nothing.

**Why threading was invented for this:** Threads give the OS scheduler a way to keep the CPU busy during those waits. A thread blocked on network I/O releases the GIL (Python's Global Interpreter Lock), which lets another thread run Python code or start its own I/O operation. The result: multiple threads can be waiting for network responses simultaneously.

**What happens without it:** A web scraper fetching 100 URLs sequentially at 1 second each takes 100 seconds. The same scraper with 10 threads can complete in roughly 10 seconds — because most of the elapsed time is spent waiting for servers to respond, and threads let you stack those waits on top of each other instead of one after another.

**Industry impact:** Production systems that perform bulk API calls, database queries, or file reads use thread pools to maximize throughput without rewriting business logic in an async style. Python's threading model maps directly to OS threads, and modern operating systems are very efficient at scheduling hundreds of I/O-blocked threads.

The threading module creates OS-level threads managed by the OS scheduler. Thread creation carries overhead — typically 50 to 100 microseconds per thread. For many short-lived tasks, creating a fresh thread per task is wasteful. Thread pools solve this by keeping a fixed set of threads alive and reusing them across many tasks.

---

## CONCEPT 1.2 — ThreadPoolExecutor: The Right Abstraction

**Problem it solves:** Raw thread management with threading.Thread requires you to start threads, track them, join them, and handle exceptions manually. For a handful of threads this is manageable, but at scale it becomes error-prone boilerplate.

**Why ThreadPoolExecutor was invented:** concurrent.futures.ThreadPoolExecutor provides a high-level, clean abstraction. You submit callable tasks; the executor assigns them to available threads from its pool; you get back Future objects that represent pending results. The executor handles thread lifecycle, exception propagation, and cleanup.

**What happens without it:** Developers managing raw Thread objects often forget to join threads, lose exceptions silently, or create too many threads and crash under load. Thread pools with a fixed max_workers cap prevent runaway thread creation.

**Industry impact:** The concurrent.futures module was designed to make safe concurrency accessible. The same API works for both thread pools and process pools, so switching execution strategies is a one-line change.

**Key parameters and patterns:**
- `max_workers` controls pool size. For I/O-bound work you can have more workers than CPU cores because threads spend most time waiting. A common formula is `min(32, os.cpu_count() + 4)` for I/O workers.
- Too many threads is counterproductive: OS context switching overhead, memory per thread (roughly 1 MB stack), and contention on shared resources all increase.
- The Executor used as a context manager ensures all submitted tasks finish before the `with` block exits.
- `executor.submit()` returns a Future immediately and is non-blocking.
- `executor.map()` applies a function to a sequence of inputs and yields results in order.
- `concurrent.futures.as_completed()` yields futures as they complete, regardless of submission order — useful when you want to process results as soon as they arrive.

---

## EXAMPLE 1.1 — Web Scraper Using ThreadPoolExecutor with as_completed

**Narration:** Let's see a real use case: fetching multiple URLs concurrently. We define a `fetchPage` function that opens a URL, reads the response, and returns the URL, content size, and elapsed time. We then use a ThreadPoolExecutor with 5 workers. We submit all URLs simultaneously using a dictionary comprehension mapping each Future back to its URL. Then we iterate with `as_completed`, which gives us each Future the moment it finishes. The program prints results as they arrive — not in submission order, but in completion order. The total elapsed time should be close to the slowest single request, not the sum of all requests.

```python
# Example 19.1
import time                                            # line 1
import concurrent.futures                              # line 2
import urllib.request                                  # line 3

URLS = [                                               # line 4
    "http://httpbin.org/delay/0",                      # line 5
    "http://httpbin.org/uuid",                         # line 6
    "http://httpbin.org/ip",                           # line 7
    "http://httpbin.org/headers",                      # line 8
    "http://httpbin.org/get",                          # line 9
]

def fetchPage(url):                                    # line 10
    start = time.perf_counter()                        # line 11
    with urllib.request.urlopen(url, timeout=10) as response:  # line 12
        content = response.read()                      # line 13
    elapsed = time.perf_counter() - start              # line 14
    return url, len(content), elapsed                  # line 15

start = time.perf_counter()                            # line 16
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:  # line 17
    futureToUrl = {executor.submit(fetchPage, url): url for url in URLS}  # line 18
    for future in concurrent.futures.as_completed(futureToUrl):  # line 19
        url, size, elapsed = future.result()           # line 20
        print(f"{url}: {size} bytes in {elapsed:.2f}s")  # line 21
totalTime = time.perf_counter() - start                # line 22
print(f"All done in {totalTime:.2f}s")                 # line 23
```

---

## CONCEPT 2.1 — Shared State and Thread Safety

**Problem it solves:** Threads sharing memory is efficient — no serialization, no copying. But without coordination, two threads reading and modifying the same variable concurrently produce unpredictable results.

**Why this is dangerous:** The `+=` operation on a Python integer is NOT atomic. It compiles to multiple bytecode instructions: load the current value, add 1, store the new value. The OS scheduler can interrupt a thread between any two bytecode instructions. If thread A reads the count (say, 5), gets interrupted, thread B reads the same count (still 5), thread B writes 6, then thread A writes 6 — we've lost one increment. This is a race condition.

**What happens without synchronization:** A counter incremented by 100 threads might show a final value of 87, or 93, or 100 — it varies by run. Race conditions are non-deterministic and notoriously difficult to reproduce in testing.

**Industry impact:** Race conditions have caused financial losses, data corruption, and security vulnerabilities in production systems. They are one of the most common bugs in concurrent code. The solution is explicit synchronization.

**The fix — threading.Lock:** A Lock ensures only one thread can enter a critical section at a time. `lock.acquire()` blocks the calling thread if another thread holds the lock. `lock.release()` releases it. Using a Lock as a context manager (`with lock:`) guarantees release even if an exception occurs inside the block. The critical section should be as small as possible to minimize contention — only protect the shared state modification, not surrounding computation.

---

## EXAMPLE 2.1 — Race Condition Demonstrated, Then Fixed with Lock

**Narration:** We define two counter classes — UnsafeCounter and SafeCounter. The unsafe version reads the count into a local variable, deliberately yields (with `time.sleep(0)` to increase the chance of a context switch), then writes back. The safe version wraps the increment with a Lock. We create 100 threads for each and run them. The unsafe counter will often show a result less than 100 due to lost updates. The safe counter always shows exactly 100.

```python
# Example 19.2
import threading                                       # line 1
import time                                            # line 2

class UnsafeCounter:                                   # line 3
    def __init__(self):                                # line 4
        self.count = 0                                 # line 5

    def increment(self):                               # line 6
        currentValue = self.count                      # line 7
        time.sleep(0)                                  # line 8
        self.count = currentValue + 1                  # line 9

class SafeCounter:                                     # line 10
    def __init__(self):                                # line 11
        self.count = 0                                 # line 12
        self._lock = threading.Lock()                  # line 13

    def increment(self):                               # line 14
        with self._lock:                               # line 15
            self.count += 1                            # line 16

THREAD_COUNT = 100                                     # line 17

unsafeCounter = UnsafeCounter()                        # line 18
threads = [threading.Thread(target=unsafeCounter.increment) for _ in range(THREAD_COUNT)]  # line 19
for t in threads: t.start()                            # line 20
for t in threads: t.join()                             # line 21
print(f"Unsafe counter (expected 100): {unsafeCounter.count}")  # line 22

safeCounter = SafeCounter()                            # line 23
threads = [threading.Thread(target=safeCounter.increment) for _ in range(THREAD_COUNT)]  # line 24
for t in threads: t.start()                            # line 25
for t in threads: t.join()                             # line 26
print(f"Safe counter (expected 100): {safeCounter.count}")  # line 27
```

---

## CONCEPT 3.1 — Thread-Safe Data Structures and Queue

**Problem it solves:** Using a Lock to protect a shared list for communication between threads is verbose and error-prone. Forgetting to acquire the lock before appending, or holding the lock too long, both cause bugs. The pattern of one thread producing work and another consuming it is so common it deserves a dedicated abstraction.

**Why threading.Queue was invented:** `queue.Queue` is a thread-safe FIFO queue with internal locks baked in. `put()` and `get()` are atomic. You don't need to write any synchronization code to use a Queue safely from multiple threads. The Queue also handles backpressure: a Queue with `maxsize` set will block `put()` if the queue is full, preventing the producer from outrunning the consumer indefinitely.

**The producer-consumer pattern:** One or more producer threads generate work items and call `queue.put()`. One or more consumer threads call `queue.get()` in a loop, process each item, and call `task_done()` to signal completion. A sentinel value (commonly `None`) signals consumers to stop. When a consumer receives the sentinel, it re-puts it so other consumers also see the stop signal.

**What happens without Queue:** Developers using a plain list shared between threads need to protect every append and pop with a Lock. Missing one call to acquire causes a race condition. Queue encapsulates this pattern correctly and efficiently.

**Industry impact:** The producer-consumer pattern is fundamental to pipeline architectures, job queues, event processing systems, and anywhere work is generated at a different rate than it is consumed. Python's Queue is the standard way to implement it with threads.

---

## EXAMPLE 3.1 — Producer-Consumer with Queue

**Narration:** We set up a Queue with a maxsize of 10 to limit buffering. A producer thread generates 5 task items, putting each on the queue with a small delay. Consumer threads loop calling `queue.get()`, process the task (simulated with a random sleep), append results to a shared list protected by a Lock, and call `task_done()`. The sentinel pattern (`None`) signals all consumers to stop. We start the producer and 3 consumer threads, join them all, and print the result count.

```python
# Example 19.3
import threading                                       # line 1
import queue                                           # line 2
import time                                            # line 3
import random                                          # line 4

taskQueue = queue.Queue(maxsize=10)                    # line 5
results = []                                           # line 6
resultsLock = threading.Lock()                         # line 7

def producer(numItems):                                # line 8
    for itemId in range(numItems):                     # line 9
        taskQueue.put(itemId)                          # line 10
        print(f"Produced task {itemId}")               # line 11
        time.sleep(0.01)                               # line 12
    taskQueue.put(None)                                # line 13

def consumer(workerId):                                # line 14
    while True:                                        # line 15
        task = taskQueue.get()                         # line 16
        if task is None:                               # line 17
            taskQueue.put(None)                        # line 18
            break                                      # line 19
        time.sleep(random.uniform(0.01, 0.05))         # line 20
        result = f"Worker {workerId} processed task {task}"  # line 21
        with resultsLock:                              # line 22
            results.append(result)                     # line 23
        taskQueue.task_done()                          # line 24

producerThread = threading.Thread(target=producer, args=(5,))  # line 25
workers = [threading.Thread(target=consumer, args=(i,)) for i in range(3)]  # line 26
producerThread.start()                                 # line 27
for w in workers: w.start()                            # line 28
producerThread.join()                                  # line 29
for w in workers: w.join()                             # line 30
print(f"Processed {len(results)} tasks")               # line 31
```

---

## CONCEPT 4.1 — Final Takeaway Lecture 19

Threading is the right tool for I/O-bound work because threads release the GIL during I/O waits, allowing multiple threads to block concurrently. ThreadPoolExecutor provides a clean, safe thread pool abstraction that handles lifecycle, exception propagation, and cleanup. Shared state between threads requires explicit synchronization — `threading.Lock` ensures only one thread modifies shared state at a time. `threading.Queue` provides a thread-safe communication channel for producer-consumer patterns without manual lock management. The next lecture examines the GIL itself in depth, explaining why it exists and what it means for CPU-bound work.

---
