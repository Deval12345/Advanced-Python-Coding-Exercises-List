# Code — Lecture 38: Big Project Stage 5 — Caching and Memory Discipline

---

## Example 38.1 — functools.lru_cache for Calibration Lookups

```python
# Example 38.1
import functools
import time
import random

CALIBRATION_DB = {
    f"S{i}": {"offset": round(random.uniform(-5, 5), 3), "scale": round(random.uniform(0.9, 1.1), 4)}
    for i in range(10)
}

def calibrationLookupUncached(sensorId):
    time.sleep(0.0001)
    return CALIBRATION_DB[sensorId]

@functools.lru_cache(maxsize=16)
def calibrationLookupCached(sensorId):
    time.sleep(0.0001)
    return CALIBRATION_DB[sensorId]

def applyCalibrationUncached(records):
    out = []
    for r in records:
        cal = calibrationLookupUncached(r["sensorId"])
        calibrated = r["value"] * cal["scale"] + cal["offset"]
        out.append({**r, "calibrated": calibrated})
    return out

def applyCalibrationCached(records):
    out = []
    for r in records:
        cal = calibrationLookupCached(r["sensorId"])
        calibrated = r["value"] * cal["scale"] + cal["offset"]
        out.append({**r, "calibrated": calibrated})
    return out

if __name__ == "__main__":
    records = [{"sensorId": f"S{i % 10}", "value": random.gauss(50, 10)} for i in range(5000)]

    t0 = time.perf_counter()
    resultUncached = applyCalibrationUncached(records)
    uncachedTime = time.perf_counter() - t0

    calibrationLookupCached.cache_clear()
    t1 = time.perf_counter()
    resultCached = applyCalibrationCached(records)
    cachedTime = time.perf_counter() - t1

    info = calibrationLookupCached.cache_info()
    print(f"Uncached: {uncachedTime:.3f}s")
    print(f"Cached:   {cachedTime:.3f}s")
    print(f"Speedup:  {uncachedTime/cachedTime:.1f}x")
    print(f"Cache info: hits={info.hits}, misses={info.misses}, size={info.currsize}")
```

### Line-by-Line Explanation

**`CALIBRATION_DB = {f"S{i}": {...} for i in range(10)}`**
Module-level dictionary representing a calibration database. Keys are sensor IDs ("S0" through "S9"); values are dicts with offset and scale factors. In production, this would be fetched from a database or configuration service — potentially with I/O latency. The module-level placement ensures the data is initialized once and shared across all function calls.

**`def calibrationLookupUncached(sensorId):`**
Simulates a calibration lookup with 0.1ms I/O latency. `time.sleep(0.0001)` represents network or disk access. Without caching, every call pays this latency. For 5,000 records with 10 distinct sensors, this function is called 5,000 times — 500ms of artificial I/O per batch.

**`@functools.lru_cache(maxsize=16)`**
Decorator that wraps `calibrationLookupCached` in a Least Recently Used cache. The cache key is built from the function's arguments — here, just `sensorId` (a string, which is hashable). When `calibrationLookupCached("S3")` is called for the first time, it executes the function body and stores the result. Every subsequent call with `"S3"` returns the stored result without executing the body.

**`maxsize=16`**
The cache holds up to 16 entries. When a 17th distinct key is added, the least recently used entry is evicted. For this pipeline with 10 distinct sensors, `maxsize=16` means all 10 sensors fit in the cache simultaneously — no evictions. Choose `maxsize` to be slightly larger than the expected number of distinct argument values.

**`calibrationLookupCached.cache_clear()`**
Empties the cache before the timed test. Without this, a warm cache from a previous call would artificially inflate the speedup measurement. Always start a benchmark from a cold cache.

**`info = calibrationLookupCached.cache_info()`**
`cache_info()` returns a named tuple with: `hits` (calls served from cache), `misses` (calls that executed the function), `maxsize` (declared maximum), `currsize` (current number of entries). After 5,000 calls with 10 distinct sensors: `hits=4990, misses=10, currsize=10`. The miss rate is 10/5000 = 0.2% — the cache serves 99.8% of calls without computation.

---

## Example 38.2 — Custom TTL Cache Implementation

```python
# Example 38.2
import time
import threading

class TTLCache:
    def __init__(self, ttlSeconds, maxSize=128):
        self._store = {}
        self._ttl = ttlSeconds
        self._maxSize = maxSize
        self._lock = threading.Lock()

    def get(self, key):
        with self._lock:
            if key in self._store:
                value, expiry = self._store[key]
                if time.monotonic() < expiry:
                    return value, True
                del self._store[key]
        return None, False

    def set(self, key, value):
        with self._lock:
            if len(self._store) >= self._maxSize:
                oldest = min(self._store, key=lambda k: self._store[k][1])
                del self._store[oldest]
            self._store[key] = (value, time.monotonic() + self._ttl)

    def invalidate(self, key):
        with self._lock:
            self._store.pop(key, None)

    def size(self):
        with self._lock:
            return len(self._store)

def makeCalibratedLookupWithTTL(ttlCache):
    def lookup(sensorId):
        cached, hit = ttlCache.get(sensorId)
        if hit:
            return cached
        result = CALIBRATION_DB[sensorId]
        ttlCache.set(sensorId, result)
        return result
    return lookup

CALIBRATION_DB = {f"S{i}": {"offset": i * 0.1, "scale": 1.0 + i * 0.01} for i in range(10)}

if __name__ == "__main__":
    cache = TTLCache(ttlSeconds=2.0, maxSize=16)
    lookup = makeCalibratedLookupWithTTL(cache)

    sensorIds = [f"S{i % 10}" for i in range(100)]
    for sid in sensorIds:
        lookup(sid)
    print(f"Cache size after 100 lookups: {cache.size()}")

    print("Waiting 2.5 seconds for TTL expiry...")
    time.sleep(2.5)

    hits, misses = 0, 0
    for sid in sensorIds[:10]:
        _, hit = cache.get(sid)
        if hit:
            hits += 1
        else:
            misses += 1
    print(f"After TTL expiry: hits={hits}, misses={misses} (expected 0 hits, 10 misses)")
```

### Line-by-Line Explanation

**`self._store = {}`**
The cache storage dictionary. Keys are cache keys (e.g. sensorIds); values are `(value, expiry_time)` tuples. Python dict lookups are O(1) average case — efficient for the frequent get operations.

**`self._lock = threading.Lock()`**
Protects the cache for concurrent access. Without the lock, two threads could simultaneously find an entry absent, both compute the value, and both attempt to write — a race condition. The lock ensures that all read-check-write sequences are atomic.

**`if time.monotonic() < expiry:`**
`time.monotonic()` returns seconds since an arbitrary fixed point, guaranteed to never go backward (unlike `time.time()` which can be adjusted by NTP or manual clock changes). Using monotonic time for expiry checks ensures TTL behavior is correct even when the system clock is adjusted.

**`del self._store[key]`**
Explicit eviction of an expired entry on access. This is "lazy expiry" — we only clean up stale entries when they are accessed, not proactively. For a cache with many entries that are rarely accessed, a background expiry thread might be appropriate. For a bounded cache with frequent access, lazy expiry is sufficient.

**`oldest = min(self._store, key=lambda k: self._store[k][1])`**
When the cache is full, evicts the entry with the smallest (earliest) expiry time — the entry that is soonest-to-expire anyway. This is more principled than random eviction: we prefer to keep entries that will remain valid longer.

**`def makeCalibratedLookupWithTTL(ttlCache):`**
A closure factory that captures the TTL cache and returns a lookup function. The returned `lookup` function checks the cache first; on a miss, computes the result and stores it. This pattern separates the cache infrastructure from the business logic — `lookup` knows about the business (calibration), but `TTLCache` knows nothing about calibrations.

**`self._store.pop(key, None)`**
Removes an entry if it exists, does nothing if it does not. The `None` default prevents `KeyError` when invalidating a key that was never cached or already evicted. `invalidate` is called when you know the source data has changed and want to force a fresh fetch.

---

## Example 38.3 — Bounded Alert Accumulator and Memory Audit

```python
# Example 38.3
import sys
import time
import random
import tracemalloc
from collections import deque

class BoundedAlertAccumulator:
    __slots__ = ("_maxAlerts", "_alerts", "_totalTriggered")

    def __init__(self, maxAlerts=500):
        self._maxAlerts = maxAlerts
        self._alerts = deque(maxlen=maxAlerts)
        self._totalTriggered = 0

    def check(self, record, threshold):
        if record["value"] > threshold:
            self._totalTriggered += 1
            self._alerts.append({
                "sensorId": record["sensorId"],
                "value": record["value"],
                "timestamp": record.get("timestamp", time.time()),
                "alertId": self._totalTriggered
            })
            return True
        return False

    def recentAlerts(self, n=10):
        return list(self._alerts)[-n:]

    def stats(self):
        return {
            "totalTriggered": self._totalTriggered,
            "currentlyStored": len(self._alerts),
            "maxCapacity": self._maxAlerts,
        }

if __name__ == "__main__":
    tracemalloc.start()
    snapshot1 = tracemalloc.take_snapshot()

    accumulator = BoundedAlertAccumulator(maxAlerts=500)
    records = [{"sensorId": f"S{i % 5}", "value": random.gauss(50, 20)} for i in range(10000)]

    for r in records:
        accumulator.check(r, threshold=60.0)

    snapshot2 = tracemalloc.take_snapshot()
    stats = snapshot2.compare_to(snapshot1, "lineno")
    netKb = sum(s.size_diff for s in stats) / 1024

    print(f"Alert stats: {accumulator.stats()}")
    print(f"Deque memory: {sys.getsizeof(accumulator._alerts)} bytes (bounded by maxlen=500)")
    print(f"Net memory allocated for full run: {netKb:.1f} KB")
    print(f"Recent alerts: {accumulator.recentAlerts(3)}")
    tracemalloc.stop()
```

### Line-by-Line Explanation

**`__slots__ = ("_maxAlerts", "_alerts", "_totalTriggered")`**
The accumulator uses `__slots__` to eliminate the `__dict__` overhead — consistent with Stage 4's memory discipline. The accumulator may be instantiated once per pipeline run or once per stage; `__slots__` ensures its own footprint is minimal.

**`self._alerts = deque(maxlen=maxAlerts)`**
The bounded alert store. `deque(maxlen=N)` creates a double-ended queue with a fixed maximum length. When the deque has `maxlen=500` elements and you call `deque.append(x)`, the leftmost (oldest) element is automatically discarded. No exception, no check required — the deque maintains the bound automatically.

**`self._totalTriggered += 1`**
The total count is maintained separately from the deque. The deque stores only the most recent 500 alerts; `_totalTriggered` records the total number of threshold crossings over the entire run. This separation allows operators to know both "how many alerts happened" and "what do the most recent alerts look like" without confusing the two.

**`tracemalloc.start()` / `tracemalloc.take_snapshot()`**
Memory profiling wrapper. `snapshot1` captures memory before the accumulator is created and used. `snapshot2` captures memory after processing 10,000 records. `snapshot2.compare_to(snapshot1, "lineno")` attributes net allocations to source lines. The total `netKb` should be stable and bounded — it should not grow proportionally with the number of records processed, because the deque evicts old entries.

**`sys.getsizeof(accumulator._alerts)`**
The shallow size of the deque object itself — not including the size of the dictionaries it contains. For a `deque(maxlen=500)`, this size is constant regardless of the deque's current length. Contrast with a plain list: `sys.getsizeof(list_obj)` grows as the list grows (more internal slots allocated).

**`accumulator.stats()`**
Returns a structured summary: `totalTriggered` (integrity metric — how many alerts actually occurred), `currentlyStored` (should always be ≤ 500 after the bound is reached), `maxCapacity` (the declared bound). In production monitoring, `currentlyStored / maxCapacity` near 1.0 indicates that the historical buffer is saturated and old alerts are being dropped — a signal that the bound may need to be increased or that alert rate is abnormally high.

---
