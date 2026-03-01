# Code — Lecture 19: Threading for I/O-Bound Work

---

## Example 19.1 — ThreadPoolExecutor with as_completed

```python
# Example 19.1
# line 1
import time
# line 2
import concurrent.futures
# line 3
import urllib.request

# line 5
URLS = [
    # line 6
    "http://httpbin.org/delay/0",
    # line 7
    "http://httpbin.org/uuid",
    # line 8
    "http://httpbin.org/ip",
    # line 9
    "http://httpbin.org/headers",
    # line 10
    "http://httpbin.org/get",
]

# line 13
def fetchPage(url):
    # line 14
    start = time.perf_counter()
    # line 15
    with urllib.request.urlopen(url, timeout=10) as response:
        # line 16
        content = response.read()
    # line 17
    elapsed = time.perf_counter() - start
    # line 18
    return url, len(content), elapsed

# line 21
start = time.perf_counter()
# line 22
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    # line 23
    futureToUrl = {executor.submit(fetchPage, url): url for url in URLS}
    # line 24
    for future in concurrent.futures.as_completed(futureToUrl):
        # line 25
        url, size, elapsed = future.result()
        # line 26
        print(f"{url}: {size} bytes in {elapsed:.2f}s")
# line 27
totalTime = time.perf_counter() - start
# line 28
print(f"All done in {totalTime:.2f}s")
```

**Line-by-line explanation:**

- **Lines 1–3:** Import `time` for measuring elapsed wall-clock time, `concurrent.futures` for the high-level thread pool API, and `urllib.request` for making HTTP requests from the standard library. No third-party dependencies are needed.

- **Lines 5–11:** Define the list of five URLs to fetch. These are all httpbin.org endpoints that return small, predictable responses. Fetching them sequentially would take the sum of all their response times; fetching them concurrently will take roughly the time of the slowest single request because threads overlap their I/O waits.

- **Lines 13–18:** Define `fetchPage`, the worker function that each thread will run. It records a start timestamp, opens the URL (blocking until the server responds), reads the full response body, then computes how long the request took. It returns a tuple of the URL, the byte count of the content, and the elapsed time. This function does all the I/O work — while a thread is waiting inside `urlopen` or `read`, the GIL is released and other threads can run.

- **Line 21:** Record the overall wall-clock start time so we can measure total elapsed time across all concurrent fetches.

- **Line 22:** Create a `ThreadPoolExecutor` with `max_workers=5`, one worker thread per URL. Using a context manager (`with`) guarantees that the executor calls `shutdown(wait=True)` on exit — meaning the block does not finish until all submitted tasks have completed. This is important: without the context manager you risk the program exiting before threads finish.

- **Line 23:** This is the dispatch step. A dictionary comprehension submits all five URLs to the pool at once. `executor.submit(fetchPage, url)` schedules `fetchPage(url)` to run on a thread and immediately returns a `Future` object — a promise of a result that will arrive later. The dictionary maps each `Future` to its URL so that, if needed, we can look up which URL a future corresponds to. All five tasks are in flight simultaneously after this line.

- **Line 24:** `concurrent.futures.as_completed(futureToUrl)` returns an iterator that yields each `Future` as soon as it finishes — not in submission order, but in completion order. The fastest request prints first. This is the key difference from `executor.map()`, which always returns results in submission order and hides which task finished first.

- **Line 25:** `future.result()` blocks until that specific future is done (it may already be done since `as_completed` only yields finished futures) and either returns the value or re-raises any exception the worker thread raised. The tuple is unpacked into `url`, `size`, and `elapsed`.

- **Line 26:** Print the result for this URL as soon as it is available, giving live feedback rather than waiting for all five to finish before printing anything.

- **Lines 27–28:** After the `with` block exits (all threads joined), compute and print the total wall-clock time. Because all five requests overlapped, the total time should be close to the slowest single request, not the sum of all five.

---

## Example 19.2 — Race Condition Demonstration and Fix with Lock

```python
# Example 19.2
# line 1
import threading
# line 2
import time

# line 4
class UnsafeCounter:
    # line 5
    def __init__(self):
        # line 6
        self.count = 0

    # line 8
    def increment(self):
        # line 9
        currentValue = self.count
        # line 10
        time.sleep(0)
        # line 11
        self.count = currentValue + 1

# line 13
class SafeCounter:
    # line 14
    def __init__(self):
        # line 15
        self.count = 0
        # line 16
        self._lock = threading.Lock()

    # line 18
    def increment(self):
        # line 19
        with self._lock:
            # line 20
            self.count += 1

# line 22
THREAD_COUNT = 100

# line 24
unsafeCounter = UnsafeCounter()
# line 25
threads = [threading.Thread(target=unsafeCounter.increment) for _ in range(THREAD_COUNT)]
# line 26
for t in threads: t.start()
# line 27
for t in threads: t.join()
# line 28
print(f"Unsafe counter (expected 100): {unsafeCounter.count}")

# line 30
safeCounter = SafeCounter()
# line 31
threads = [threading.Thread(target=safeCounter.increment) for _ in range(THREAD_COUNT)]
# line 32
for t in threads: t.start()
# line 33
for t in threads: t.join()
# line 34
print(f"Safe counter (expected 100): {safeCounter.count}")
```

**Line-by-line explanation:**

- **Lines 1–2:** Import `threading` for creating threads and locks, and `time` so we can artificially widen the race window with `sleep`.

- **Lines 4–11:** Define `UnsafeCounter`, the broken version. The `increment` method deliberately splits what looks like a single operation into three visible steps: (1) read `self.count` into a local variable, (2) sleep for zero seconds to yield the GIL and force a thread switch, (3) write `currentValue + 1` back to `self.count`. In real code the read-modify-write happens in Python bytecode instructions that are individually interruptible — we use `sleep(0)` to make the interleaving guaranteed and visible rather than probabilistic.

- **Line 10:** `time.sleep(0)` is the key to making the race condition reliable. Normally the GIL might not switch threads between the read and the write, making the bug rare and hard to reproduce. `sleep(0)` tells the OS "I am voluntarily yielding my time slice," which almost always causes another thread to run. This exposes the window between the read and the write, so many threads read the same stale value, increment it, and write the same result — losing each other's updates.

- **Lines 13–20:** Define `SafeCounter`, the corrected version. It adds a `threading.Lock()` in `__init__`. A `Lock` can be held by at most one thread at a time — all others block at the `with self._lock:` line until the current holder releases it by exiting the `with` block. Using `with` is the correct pattern: it guarantees the lock is released even if an exception is raised inside the block.

- **Line 20:** Inside the lock, `self.count += 1` is now safe because no other thread can enter this block simultaneously. Even though `+=` compiles to multiple bytecode instructions (load, add, store), the lock wraps all of them as an atomic unit from the perspective of other threads.

- **Lines 22–28:** Run the unsafe version. Create 100 threads, start them all, then join them all (wait for all to finish). The printed count is almost always less than 100 because threads overwrote each other's increments. The exact number varies between runs, which is a hallmark of a race condition.

- **Lines 30–34:** Run the safe version with the identical pattern. Because the lock serialises the critical section, every increment is counted and the output is always exactly 100.

---

## Example 19.3 — Producer-Consumer with Queue

```python
# Example 19.3
# line 1
import threading
# line 2
import queue
# line 3
import time
# line 4
import random

# line 6
taskQueue = queue.Queue(maxsize=10)
# line 7
results = []
# line 8
resultsLock = threading.Lock()

# line 10
def producer(numItems):
    # line 11
    for itemId in range(numItems):
        # line 12
        taskQueue.put(itemId)
        # line 13
        print(f"Produced task {itemId}")
        # line 14
        time.sleep(0.01)
    # line 15
    taskQueue.put(None)

# line 17
def consumer(workerId):
    # line 18
    while True:
        # line 19
        task = taskQueue.get()
        # line 20
        if task is None:
            # line 21
            taskQueue.put(None)
            # line 22
            break
        # line 23
        time.sleep(random.uniform(0.01, 0.05))
        # line 24
        result = f"Worker {workerId} processed task {task}"
        # line 25
        with resultsLock:
            # line 26
            results.append(result)
        # line 27
        taskQueue.task_done()

# line 29
producerThread = threading.Thread(target=producer, args=(5,))
# line 30
workers = [threading.Thread(target=consumer, args=(i,)) for i in range(3)]
# line 31
producerThread.start()
# line 32
for w in workers: w.start()
# line 33
producerThread.join()
# line 34
for w in workers: w.join()
# line 35
print(f"Processed {len(results)} tasks")
```

**Line-by-line explanation:**

- **Lines 1–4:** Import `threading` for threads and locks, `queue` for the thread-safe queue, `time` for simulating work delays, and `random` so each consumer takes a variable amount of time — reflecting the unpredictable nature of real I/O.

- **Line 6:** Create a `queue.Queue` with `maxsize=10`. The queue is the communication channel between the producer and consumer threads. Setting `maxsize` applies backpressure: if the queue already holds 10 items, `put()` will block the producer until a consumer removes an item. This prevents the producer from outrunning consumers and exhausting memory if consumers are slow.

- **Lines 7–8:** `results` is a plain list that all consumer threads will write to. Because `list.append` is not guaranteed to be atomic across all Python implementations and contexts, we protect it with `resultsLock`. A separate lock for the results list is a common pattern — it keeps the results lock independent from the queue's internal lock, avoiding unnecessary contention.

- **Lines 10–15:** The `producer` function puts `numItems` integer task IDs onto the queue one at a time, sleeping 10 ms between each to simulate the pace of arriving work (like reading lines from a file or receiving network requests). After all real tasks are produced, it puts a single `None` sentinel value onto the queue. The sentinel signals consumers that no more work is coming.

- **Lines 17–27:** The `consumer` function runs in an infinite loop. `taskQueue.get()` blocks until an item is available, then removes and returns it. This is the correct blocking pattern — the consumer thread sleeps inside `get()` without burning CPU while the queue is empty. When it receives `None` (the sentinel), it re-puts `None` back onto the queue before breaking — this is the fan-out sentinel pattern. Because there are 3 consumer threads but only 1 sentinel, re-putting it ensures all 3 consumers eventually see it and shut down cleanly.

- **Line 23:** `time.sleep(random.uniform(0.01, 0.05))` simulates variable-duration I/O work. During this sleep the GIL is released, so the other consumer threads and the producer thread can make progress concurrently.

- **Lines 25–26:** Append the result string to the shared list inside `resultsLock`. Even though `append` tends to be safe in CPython due to the GIL, relying on GIL implementation details is fragile — an explicit lock makes the thread-safety contract clear and portable.

- **Line 27:** `taskQueue.task_done()` signals that the item retrieved by the most recent `get()` has been fully processed. This is required if any code calls `taskQueue.join()` (which blocks until all items have been processed and `task_done` called for each). Good practice to include it even when not currently calling `join()`.

- **Lines 29–35:** Create and start the producer thread and three consumer threads. Join the producer first (wait for all items to be placed on the queue), then join all consumers (wait for them to finish processing and exit their loops). Finally, print the count of results, which should equal the number of tasks produced.
