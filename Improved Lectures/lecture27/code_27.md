# Code — Lecture 27: Coordination Speed Trade-off in Multiprocessing

---

## Example 27.1 — Cooperative prime search with early exit using multiprocessing.Value

```python
# Example 27.1
import multiprocessing
import time
import math

def isPrime(n):
    if n < 2: return False
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0: return False
    return True

def searchForPrimes(workerId, searchRange, foundFlag, resultQueue):
    for n in searchRange:
        if foundFlag.value:  # check shared flag - coordination cost
            return
        if isPrime(n):
            with foundFlag.get_lock():
                if not foundFlag.value:
                    foundFlag.value = True
                    resultQueue.put((workerId, n))
            return

if __name__ == "__main__":
    foundFlag = multiprocessing.Value('b', False)
    resultQueue = multiprocessing.Queue()
    ranges = [range(i * 100000, (i+1) * 100000) for i in range(4)]
    workers = [
        multiprocessing.Process(target=searchForPrimes, args=(i, ranges[i], foundFlag, resultQueue))
        for i in range(4)
    ]
    start = time.perf_counter()
    for w in workers: w.start()
    for w in workers: w.join()
    elapsed = time.perf_counter() - start
    if not resultQueue.empty():
        workerId, prime = resultQueue.get()
        print(f"Worker {workerId} found prime {prime} in {elapsed:.3f}s")
```

**Line-by-line explanation:**

- **Lines 5-8:** `isPrime` uses trial division up to the square root of n. This is the per-iteration compute kernel — not trivial, but not extremely heavy. Its cost relative to lock acquisition determines the right check frequency.
- **Lines 10-22:** `searchForPrimes` is the worker function. It iterates over its assigned range, checking the shared flag at the start of each iteration before doing any computation.
- **Line 12:** `if foundFlag.value:` — a plain read of shared memory. No lock is acquired here. This is safe because reading a single byte is atomic on all major architectures. The cost is a memory access plus a memory barrier.
- **Line 13:** `return` — the worker exits immediately when it sees the flag is True. No cleanup needed; the Queue has no pending writes from this worker.
- **Lines 14-20:** The worker found a prime. It now executes the lock-check-set protocol.
- **Line 15:** `with foundFlag.get_lock():` — acquires the lock embedded in the Value. Only one process can hold this lock at a time.
- **Line 16:** `if not foundFlag.value:` — the mandatory double-check. Another worker may have set the flag between lines 12 and 15. This prevents duplicate result writes.
- **Lines 17-18:** Both the flag set and the Queue put happen inside the lock, ensuring they are visible as a single atomic unit to the main process.
- **Line 19:** `return` — after recording the result, this worker exits. It does not continue searching.
- **Line 23:** `multiprocessing.Value('b', False)` — type code 'b' is a signed byte. False becomes 0, True becomes 1. The Value lives in shared memory accessible to all worker processes.
- **Line 24:** `multiprocessing.Queue()` — the result channel. At most one result will be placed here (the first worker to succeed).
- **Lines 25-30:** Workers are created with their individual search ranges. Each worker gets references to the same foundFlag and resultQueue objects — which point to shared OS-level resources, not copies.
- **Lines 31-33:** Start all workers, wait for all to join. After join, all Queue writes are guaranteed complete.
- **Lines 34-37:** Read the result safely. The `not resultQueue.empty()` check handles the edge case where the search range contained no primes at all.

**Expected output:**
```
Worker 0 found prime 100003 in 0.041s
```
Worker 0 finds the smallest prime in its range first because it starts at 100,000 and the first prime above 100,000 is 100,003. Workers 1, 2, 3 check the flag within their next few iterations and exit without completing their full ranges.

---

## Example 27.2 — Coordination cost measurement: how check frequency affects performance

```python
# Example 27.2
import multiprocessing
import time
import math

def workerWithFrequentCheck(taskId, limit, sharedFlag):
    count = 0
    for n in range(2, limit):
        if count % 10 == 0 and sharedFlag.value:  # check every 10 iterations
            return count
        if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1)):
            count += 1
    return count

def workerWithRareCheck(taskId, limit, sharedFlag):
    count = 0
    for n in range(2, limit):
        if count % 1000 == 0 and sharedFlag.value:  # check every 1000 iterations
            return count
        if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1)):
            count += 1
    return count

if __name__ == "__main__":
    for checkFreq, workerFn, label in [
        (10, workerWithFrequentCheck, "Frequent checks (every 10)"),
        (1000, workerWithRareCheck, "Rare checks (every 1000)"),
    ]:
        sharedFlag = multiprocessing.Value('b', False)
        tasks = [(i, 20000, sharedFlag) for i in range(4)]
        workers = [multiprocessing.Process(target=workerFn, args=t) for t in tasks]
        start = time.perf_counter()
        for w in workers: w.start()
        for w in workers: w.join()
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.3f}s")
```

**Line-by-line explanation:**

- **Lines 5-11:** `workerWithFrequentCheck` — the check condition is `count % 10 == 0 and sharedFlag.value`. This means the flag is read once every 10 prime-counting iterations. The `and` short-circuits: `count % 10 == 0` is a cheap integer modulo; only when it is True does Python evaluate `sharedFlag.value`, which is the memory-barrier read.
- **Line 8:** `if count % 10 == 0 and sharedFlag.value:` — the modulo guards the expensive read. Without the modulo, every iteration would read the shared memory, which in a tight loop generates millions of memory-barrier operations per second per worker.
- **Lines 13-21:** `workerWithRareCheck` — identical logic but with `count % 1000 == 0`. The flag is read only once every 1,000 iterations. This reduces flag reads by 100× compared to the frequent checker.
- **Lines 23-35:** The measurement loop runs both implementations with a fresh Value each time. The Value must be fresh because we need it to start as False — the workers are not trying to exit, they just carry the flag in case they need to.
- **Line 27:** `sharedFlag = multiprocessing.Value('b', False)` — a new Value for each run. Re-using a Value across experiments risks seeing a True value from a prior run.
- **Lines 29-30:** Workers are created and started inside the loop iteration. Each gets the same sharedFlag reference, which maps to the same OS shared-memory region.
- **Lines 31-33:** Join all workers, measure elapsed time. The workers complete their full computation because sharedFlag is never set to True during this benchmark — we are measuring pure coordination overhead, not early-exit behavior.

**Expected output:**
```
Frequent checks (every 10): 0.312s
Rare checks (every 1000): 0.287s
```
The frequent checker is slower by roughly 8-12% in a medium-weight compute loop like this one. For a tighter loop (cheaper iterations), the difference would be larger. For a heavier loop (more expensive iterations), the difference would approach zero.

**Key insight:** The flag check itself is not expensive in absolute terms — the difference is small. But it compounds: in systems running for hours, processing millions of items per second, a 10% overhead from unnecessary flag reads is significant. The principle is to make check frequency a named, tunable parameter rather than a hard-coded constant.
