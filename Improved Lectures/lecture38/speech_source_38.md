# Speech Source — Lecture 38: Big Project Stage 5 — Caching and Memory Discipline

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Stage 4, we added measurement to the pipeline: cProfile for development-time hotspot identification, @measuredStage for continuous production metrics, and __slots__ with tracemalloc for memory-efficient infrastructure. We now know exactly where time and memory go. Today, Stage 5 applies that knowledge. The profiler revealed that certain computations are called repeatedly with the same inputs. The tracemalloc audit showed that unbounded data accumulation can exhaust memory. Stage 5 adds two disciplines: caching to amortize repeated expensive computation, and bounded memory design to prevent the pipeline from growing indefinitely.

---

## CONCEPT 1.1 — Why Caching Matters in a Production Pipeline

**Problem it solves:**
The analytics pipeline calls certain functions repeatedly with the same arguments. Schema validation for a record type is called once per record — but the same five sensor types appear in every batch, so the same five validations run again and again. Calibration lookups map sensorId to calibration parameters — the same 10 sensors are queried 50,000 times each. Configuration parsing runs every time a stage is constructed. Without caching, these computations are executed redundantly. Each execution is cheap individually — perhaps 0.1ms — but 50,000 identical calibration lookups adds 5 seconds of pure overhead.

**Why invented:**
Memoization — caching the result of a function call keyed by its arguments — is one of the oldest optimizations in computer science. The name comes from the Latin "memorandum" — a thing to be remembered. The insight is that a pure function (same inputs → same output, no side effects) can be called once and its result stored; all subsequent calls with the same arguments return the cached result instantly. `functools.lru_cache` implements memoization in Python's standard library with a Least Recently Used eviction policy — when the cache reaches its maximum size, the least recently used entry is evicted to make room for a new one. `maxsize=None` creates an unbounded cache; `maxsize=128` limits to 128 entries.

**What happens without it:**
Every schema validation, calibration lookup, and configuration parse is repeated from scratch. For a sensor pipeline reading 10 sensors × 5000 records each, the same 10 calibration lookups execute 50,000 times. Each lookup might involve a dictionary search, some arithmetic, and object creation — perhaps 0.2ms. Total: 10 seconds of calibration overhead per 50,000 records. With caching: the same 10 lookups execute once — 0.002ms. The caching removes 9.998 seconds of redundant work.

**Industry impact:**
Caching is ubiquitous in production Python systems. Django's `@cached_property` caches the result of a property on the instance. FastAPI's dependency injection uses caching for expensive shared dependencies. Database ORM systems (SQLAlchemy, Django ORM) use per-request cache (identity map) to avoid querying the same row twice in one request. NumPy's `linalg` functions cache intermediate factorizations. Any system that repeatedly calls the same pure function with the same arguments should cache the results.

---

## EXAMPLE 1.1 — functools.lru_cache for Calibration Lookups

Narration: We simulate a calibration lookup service that maps sensorId to calibration parameters (offset and scale). The lookup itself involves dictionary access and arithmetic — cheap individually. We demonstrate: without caching, 5000 sensor records each triggering a calibration lookup costs measurably more than with caching. We use functools.lru_cache with maxsize=16 (one entry per sensor, and we have 10 sensors — 16 gives headroom). The lru_cache.cache_info() method shows: hits (calls served from cache), misses (calls that computed and stored), and current size. After processing all records, we should see 10 misses (one per sensor) and 4990 hits (every subsequent call served from cache).

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

---

## CONCEPT 2.1 — TTL Cache: When Cached Results Should Expire

**Problem it solves:**
`functools.lru_cache` is a pure memoization cache — it caches a result forever (or until evicted by the LRU policy). This is correct for truly pure functions: functions whose output depends only on their inputs and never changes. But many production functions have inputs that are logically pure but physically stale: a configuration read from a file, a calibration fetched from a database, a feature flag read from a remote service. These values are stable for minutes or hours — worth caching — but they do change occasionally. A cache that never expires returns stale values for arbitrarily long after the source changes.

**Why invented:**
TTL (Time-To-Live) caching adds an expiry timestamp to each cached entry. An entry is valid for TTL seconds after it was computed; after TTL seconds, the next access triggers a fresh computation and updates the cache. This is the standard caching model in web systems: CDNs (Content Delivery Networks), DNS resolvers, and HTTP caches all implement TTL. In Python, there is no standard library TTL cache (as of Python 3.11) — you implement it with a dictionary storing (result, timestamp) pairs, or use the `cachetools` third-party library.

**What happens without it:**
Without TTL, you choose between two bad options: use lru_cache and accept stale values indefinitely, or do not cache at all and pay the full computation cost on every call. TTL is the principled middle ground: aggressive caching for the expected lifetime of the value, automatic expiry when the value may have changed. The TTL is a system parameter tuned to the update frequency of the source: seconds for rapidly-changing config, hours for stable calibrations, days for reference data that changes only on releases.

**Industry impact:**
TTL caching is fundamental to distributed systems. Redis, Memcached, and Varnish all implement TTL. AWS ElastiCache is a managed TTL cache service. HTTP's Cache-Control header specifies TTL as `max-age`. In Python microservice architectures, TTL caches for database lookups, external API responses, and feature flags are standard practice. The `cachetools` library provides TTLCache with an identical interface to dict but with automatic entry expiry.

---

## EXAMPLE 2.1 — Custom TTL Cache Implementation

Narration: We implement a simple TTL cache as a class. The cache stores (value, expiry_time) pairs in a dictionary. On access, we check whether the current time exceeds the expiry time — if yes, the entry is stale and we recompute. On write, we set expiry_time = current_time + ttl_seconds. We wrap the calibration lookup in a TTL cache with a 5-second TTL and demonstrate that within 5 seconds, all calls are served from cache; after 5 seconds, the next call triggers a fresh lookup. We also demonstrate programmatic cache invalidation — useful when you know the source has changed.

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

---

## CONCEPT 3.1 — Bounded Memory: Preventing Unlimited Growth

**Problem it solves:**
A streaming pipeline that runs indefinitely has a fundamental memory hazard: any data structure that accumulates without bounds will eventually exhaust memory. An alert accumulator that appends every triggered alert grows without limit. A history buffer that stores every processed record fills RAM over hours. The solution is bounded data structures: structures with a maximum size that evict old entries when the limit is reached. Python's `collections.deque(maxlen=N)` is the standard bounded sequence. A `TTLCache` with `maxSize` is a bounded cache. A thread-safe bounded queue with `queue.Queue(maxsize=N)` provides backpressure — producers block when the queue is full, preventing memory overflow.

**Why invented:**
Bounded data structures are the standard defense against unbounded memory growth in long-running systems. The bound is a system parameter that translates resource constraints into data structure limits: "I have 256 MB of memory for history buffers; each record is 500 bytes; 256 MB / 500 bytes = 512,000 records maximum." Every accumulator, buffer, cache, and queue in a production streaming system should have an explicit bound. The bound defines the system's memory footprint and its behavior under overload (graceful dropping of old data) rather than under unlimited growth (eventual crash).

**What happens without it:**
A streaming pipeline without bounded buffers can run correctly for hours in testing (small test data, short runs) and crash in production after 6 hours (real data, continuous operation). The crash is not dramatic — Python's memory allocator simply raises MemoryError when the process exceeds the system limit, or the OS kills the process when it exceeds the container's memory limit. The monitoring dashboard shows memory climbing steadily from deployment until crash. This is one of the most common production failure modes for streaming systems.

**Industry impact:**
Every production streaming framework enforces buffer bounds. Apache Kafka consumers have configurable maximum fetch sizes. Flink and Spark streaming use bounded windowed aggregations. Python's `asyncio.Queue(maxsize=N)` provides backpressure. Redis' LRU eviction policy bounds cache size. In the analytics pipeline context, every stage's output buffer, alert accumulator, and metric history should declare its maximum size explicitly.

---

## EXAMPLE 3.1 — Bounded Alert Accumulator and Memory Audit

Narration: We add a BoundedAlertAccumulator to the analytics pipeline. It stores the last N alerts using collections.deque(maxlen=N). When the deque is full, appending a new alert automatically drops the oldest. We run the pipeline for 10,000 records with a threshold that triggers many alerts, and verify that memory usage is bounded and stable — the deque never grows beyond its declared maximum. We use sys.getsizeof and tracemalloc to verify the memory bound is respected.

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

---

## CONCEPT 4.1 — Final Takeaway: Lecture 38

Stage 5 of the big project adds two disciplines: caching and bounded memory. `functools.lru_cache` amortizes repeated pure function calls — calibration lookups, schema validation, configuration parsing — from O(N calls × cost) to O(distinct inputs × cost). The custom TTL cache extends this to semi-stable values that change on a known schedule. Bounded data structures — `deque(maxlen=N)`, bounded queues, sized caches — constrain memory growth to declared limits, preventing the "works in testing, crashes in production after 6 hours" failure mode.

The design discipline: every accumulator, buffer, and cache in a production streaming system must have an explicit size bound. Every expensive function that is called repeatedly with the same arguments should be cached. Every cache must have either a size limit or a TTL (or both) to prevent indefinite growth.

In the next stage (Lecture 39), we add resilience — the ability to detect failures and recover from them without human intervention. We build retry logic with exponential backoff, circuit breakers for downstream dependencies, and graceful degradation patterns that keep the pipeline partially functional even when components fail.

