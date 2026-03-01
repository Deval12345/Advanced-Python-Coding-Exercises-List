# Slides — Lecture 38: Big Project Stage 5 — Caching and Memory Discipline

---

## Slide 1 — Stage 5 Goal: Caching and Bounded Memory

**Two disciplines for long-running production pipelines**

- Problem 1: Expensive computations called repeatedly with the same inputs
  → Solution: Caching (functools.lru_cache, TTL cache)
- Problem 2: Unbounded data structures grow until memory exhaustion
  → Solution: Bounded containers (deque maxlen, Queue maxsize, sized caches)
- Tools: functools.lru_cache, threading.Lock, collections.deque, tracemalloc
- Design principle: every accumulator must declare its maximum size

---

## Slide 2 — The Redundancy Problem

**50,000 records, 10 sensors, 50,000 calibration lookups — 49,990 are redundant**

```
Record stream: S0, S3, S7, S0, S2, S3, S0, ...
                     10 distinct sensorIds
                     50,000 total records
                     → same 10 lookups repeated 5,000 times each

Without cache: 50,000 × 0.1ms = 5 seconds of calibration overhead per batch
With cache:    10 × 0.1ms     = 0.001 seconds (50,000 cache hits after 10 misses)
```

- Pattern: pure function called many times with the same arguments
- Pure function: same inputs → same outputs; no side effects
- This pattern appears everywhere: schema validation, config parsing, feature flags, lookups

---

## Slide 3 — functools.lru_cache

**Standard library memoization with LRU eviction**

```python
@functools.lru_cache(maxsize=16)
def calibrationLookupCached(sensorId):
    time.sleep(0.0001)            # expensive: database or file I/O
    return CALIBRATION_DB[sensorId]

# After 5000 calls with 10 distinct sensorIds:
info = calibrationLookupCached.cache_info()
# CacheInfo(hits=4990, misses=10, maxsize=16, currsize=10)
```

- `maxsize=16`: cache stores up to 16 entries; oldest-use evicted when full
- `maxsize=None`: unbounded cache (use only when you know the key space is small and finite)
- Arguments must be **hashable**: strings ✓, integers ✓, tuples ✓, lists ✗, dicts ✗
- `cache_clear()`: evict all entries (use when the underlying data changes)
- `cache_info()`: hits, misses, currsize, maxsize — observability built in

---

## Slide 4 — Example 38.1: Measuring Cache Speedup

**Uncached vs. cached calibration — quantitative comparison**

```python
records = [{"sensorId": f"S{i % 10}", "value": ...} for i in range(5000)]

# Uncached: 5000 lookups × 0.1ms = ~0.5 seconds
t0 = time.perf_counter()
resultUncached = applyCalibrationUncached(records)
uncachedTime = time.perf_counter() - t0

# Cached: 10 real lookups + 4990 cache hits ≈ 0.001 seconds + overhead
calibrationLookupCached.cache_clear()
t1 = time.perf_counter()
resultCached = applyCalibrationCached(records)
cachedTime = time.perf_counter() - t1

# Speedup: 50–100×
# Cache info: hits=4990, misses=10, currsize=10
```

- `cache_clear()` before test: ensures a cold cache (no prior state)
- `cache_info()` after: verify the expected hit/miss ratio

---

## Slide 5 — TTL Cache: Caching Values That Change Over Time

**functools.lru_cache caches forever — not appropriate for mutable data**

| Source | How often changes | Correct cache |
|--------|------------------|---------------|
| Physical calibration constants | Never (or rarely) | lru_cache (no TTL) |
| Configuration file values | Minutes–hours | TTL cache, TTL = minutes |
| Feature flags | Seconds–minutes | TTL cache, TTL = seconds |
| Real-time sensor readings | Every record | No caching — always fresh |

```
TTL = 5 seconds:
  t=0s: lookup("S0") → miss → compute → store with expiry=5s
  t=1s: lookup("S0") → hit  → return cached (expiry not reached)
  t=4s: lookup("S0") → hit  → return cached
  t=6s: lookup("S0") → miss → recompute → store with new expiry=11s
```

---

## Slide 6 — Example 38.2: TTL Cache Implementation

**Thread-safe TTL cache with bounded size and programmatic invalidation**

```python
class TTLCache:
    def __init__(self, ttlSeconds, maxSize=128):
        self._store = {}       # key → (value, expiry_time)
        self._ttl = ttlSeconds
        self._maxSize = maxSize
        self._lock = threading.Lock()

    def get(self, key):
        with self._lock:
            if key in self._store:
                value, expiry = self._store[key]
                if time.monotonic() < expiry:
                    return value, True    # cache hit
                del self._store[key]     # expired: evict
        return None, False               # cache miss

    def set(self, key, value):
        with self._lock:
            if len(self._store) >= self._maxSize:
                oldest = min(self._store, key=lambda k: self._store[k][1])
                del self._store[oldest]  # evict oldest-expiry entry
            self._store[key] = (value, time.monotonic() + self._ttl)
```

- `time.monotonic()`: never goes backward (safe for duration measurement)
- `threading.Lock()`: required — multiple threads may access simultaneously
- `maxSize` eviction: drop the entry whose expiry is soonest (already closest to stale)

---

## Slide 7 — Bounded Memory: The Long-Running Pipeline Problem

**Unbounded accumulators → gradual memory exhaustion → crash**

Scenario: alert accumulator appends every triggered alert to a list:
```
Testing (5 min):  1,000 alerts × 200 bytes  = 200 KB  → fine
Production (6h):  500,000 alerts × 200 bytes = 100 MB → fine
Production (24h): 2,000,000 alerts × 200 bytes = 400 MB → limit exceeded → crash
```

**The fix: bounded accumulator with deque(maxlen=N)**
```python
self._alerts = deque(maxlen=500)
# When deque is full and you append, oldest is automatically dropped
# Memory stays at: 500 × 200 bytes = 100 KB — always
```

- Design rule: **every accumulator, buffer, and cache must declare its maximum size**
- "Works in testing, crashes in production" is almost always an unbounded data structure

---

## Slide 8 — Example 38.3: BoundedAlertAccumulator with Memory Audit

**deque(maxlen=500) + __slots__ + tracemalloc verification**

```python
class BoundedAlertAccumulator:
    __slots__ = ("_maxAlerts", "_alerts", "_totalTriggered")

    def __init__(self, maxAlerts=500):
        self._maxAlerts = maxAlerts
        self._alerts = deque(maxlen=maxAlerts)
        self._totalTriggered = 0

    def check(self, record, threshold):
        if record["value"] > threshold:
            self._totalTriggered += 1
            self._alerts.append({...})   # oldest dropped automatically when full
            return True
        return False
```

After 10,000 records, many alerts triggered:
- `totalTriggered`: actual number of threshold crossings (e.g. 3,500)
- `len(self._alerts)`: always ≤ 500 (bounded by deque maxlen)
- Memory: constant, regardless of run duration or alert rate

---

## Slide 9 — Choosing the Right Bounded Container

**Each has a different eviction behavior — the choice depends on use case**

| Container | Bound mechanism | Eviction policy | Use case |
|-----------|----------------|-----------------|----------|
| `deque(maxlen=N)` | Append drops oldest | Oldest entry | Recent history, sliding window |
| `lru_cache(maxsize=N)` | LRU eviction | Least recently used | Function memoization |
| `TTLCache(maxSize=N)` | Oldest-expiry eviction | Soonest-to-expire | Mutable data caching |
| `queue.Queue(maxsize=N)` | Backpressure (blocks) | Producer waits | Pipeline buffers |
| `asyncio.Queue(maxsize=N)` | Backpressure (awaits) | Awaiting producer | Async pipeline buffers |

- **deque**: drop old data; pipeline keeps the most recent N readings
- **LRU cache**: drop rarely-used data; popular entries stay in cache
- **Queue with backpressure**: slow down the producer; no data lost, but producer may wait

---

## Slide 10 — Stage 5 Design Checklist

**What every Stage 5 component must declare**

For every cache:
- [ ] What function does it cache? (Must be pure or semi-pure)
- [ ] What is the maxsize? (Prevents unbounded memory growth)
- [ ] What is the TTL, if any? (Prevents indefinite staleness)

For every accumulator / buffer:
- [ ] What is the maximum number of entries? (deque maxlen, queue maxsize)
- [ ] What happens when the limit is reached? (Drop old, block producer, or error)

For every pipeline queue:
- [ ] Is it bounded? (asyncio.Queue maxsize or queue.Queue maxsize)
- [ ] What is the backpressure behavior? (Block? Raise? Log and drop?)

---

## Slide 11 — Lecture 38 Key Principles

**What to carry into Stage 6**

- `functools.lru_cache`: amortize expensive pure functions from O(N) to O(distinct inputs)
- TTL cache: for semi-stable values (configurations, calibrations) that change on a known schedule
- `deque(maxlen=N)`: bounded recent-history buffer with automatic oldest-drop eviction
- Every cache needs maxsize OR TTL (or both) — never an unbounded cache
- Every accumulator, buffer, and queue needs a declared maximum size
- tracemalloc + deque maxlen: verify the memory bound holds under real load
- Next: Stage 6 — resilience: retry logic, circuit breakers, graceful degradation

---
