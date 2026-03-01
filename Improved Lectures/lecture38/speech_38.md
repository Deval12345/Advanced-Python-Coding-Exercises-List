# Lecture 38: Big Project Stage 5 — Caching and Memory Discipline
## speech_38.md — Instructor Spoken Narrative

---

Stage 4 gave us measurement. We can now see exactly where time goes in the pipeline — which functions are the hotspots, what the throughput of each stage is, how much memory the infrastructure consumes. Today, Stage 5 applies that knowledge in two concrete directions: caching, and bounded memory.

Let me start with a concrete scenario that shows why caching matters.

Our pipeline processes 50,000 sensor records per batch. Each record carries a sensor ID — one of ten sensors. For each record, the pipeline performs a calibration lookup: it queries a calibration database to get the offset and scale factor for that sensor's measurements. The lookup involves a dictionary access, some arithmetic, maybe a small validation. Very fast individually — perhaps 0.1 milliseconds. But 50,000 records, each triggering a calibration lookup, means 50,000 lookups. That is 5 seconds of calibration overhead per batch.

But look carefully at what we are computing. There are ten sensors. There are 50,000 records. Which means each of the ten calibrations is computed 5,000 times — with exactly the same inputs, producing exactly the same outputs. Nine-thousand-nine-hundred-and-ninety of those five-thousand computations are pure redundancy.

Memoization eliminates the redundancy. Cache the result of the first call. Every subsequent call with the same arguments returns the cached result instantly.

---

`functools.lru_cache` is Python's standard memoization tool. You apply it as a decorator to any pure function — any function whose output depends only on its inputs and has no side effects. The decorator wraps the function in a cache that maps argument tuples to return values. On the first call with given arguments, the function executes and the result is stored. On every subsequent call with the same arguments, the stored result is returned immediately.

The LRU part stands for Least Recently Used — the eviction policy. When the cache reaches its maximum size (`maxsize` parameter), the least recently used entry is evicted to make room. For our calibration lookup with ten sensors, `maxsize=16` is generous — sixteen slots for ten distinct lookups.

`lru_cache` gives you a `cache_info()` method that returns the hit count, miss count, current size, and maximum size. After processing 50,000 records with ten distinct sensors: ten misses (one per sensor, on first access), 49,990 hits (every subsequent access). The calibration computation runs ten times instead of 50,000 times.

In Example 38.1, we measure the actual difference. The uncached version processes 5,000 records in something like 0.5 seconds — dominated by the artificial 0.1ms delay per lookup. The cached version processes the same 5,000 records in a few milliseconds — ten computations instead of 5,000. The speedup is often 50 to 100 times for this pattern.

There is one critical requirement for `lru_cache`: the function's arguments must be hashable. The cache uses arguments as dictionary keys. Lists, dicts, and sets cannot be dictionary keys — you cannot cache a function that takes a list argument without first converting it to a tuple. For the calibration lookup, sensor IDs are strings — hashable. For more complex cases, you can use `functools.lru_cache` with keyword-only arguments, or convert mutable arguments to frozensets or tuples before passing them.

---

Now — `lru_cache` caches forever, or until the LRU eviction removes the entry. This is correct for truly pure functions: if the calibration parameters for sensor S0 are determined at hardware manufacturing time and never change, caching them forever is right. But in practice, calibration parameters might be updated when a sensor is recalibrated. A configuration value might be updated without restarting the service. A feature flag might change.

For these cases, we need TTL caching — Time-To-Live. Each cached entry has a timestamp; entries older than the TTL are considered stale and trigger a fresh computation on the next access.

The TTL cache in Example 38.2 uses a straightforward design. A dictionary stores `(value, expiry_time)` pairs. On get: check if the key exists and if the current time is before the expiry — if both, return the cached value. On set: store the value with `expiry_time = current_time + ttl_seconds`. The cache is protected by a threading.Lock because multiple pipeline coroutines or threads may access it simultaneously.

The TTL is a system design parameter. For calibration parameters updated weekly: TTL of hours or days. For configuration values updated by operators: TTL of minutes. For feature flags that can change rapidly: TTL of seconds. The cache trades staleness risk for computation savings — the right TTL depends on how costly staleness is versus how expensive the computation is.

There is also a `maxSize` parameter on the TTL cache. When the cache is full and a new entry arrives, the oldest entry (by expiry time) is evicted. This bounds the cache's memory footprint, which brings us directly to the second discipline of this stage.

---

Bounded memory. Every data structure in a streaming pipeline that runs indefinitely must have a declared maximum size. I want to be very clear about why this is non-negotiable.

Consider an alert accumulator that appends every triggered alert to a list. In testing, the pipeline runs for five minutes. A thousand alerts accumulate. Memory usage is fine. The pipeline is deployed to production. It runs continuously. After six hours, ten million alerts have accumulated. Each alert dict is roughly 200 bytes. Ten million times 200 bytes is 2 gigabytes. The container's memory limit is 1 gigabyte. The process is killed.

The failure looks random because it depends on alert rate, which varies with sensor conditions. The test suite never caught it because no test runs for six hours. This is the classic "works in testing, crashes in production after N hours" failure mode. And it is entirely preventable by declaring every accumulator's maximum size.

`collections.deque(maxlen=N)` is Python's bounded sequence. When the deque is full and you append a new element, the oldest element is automatically dropped from the other end. No exception, no error — graceful eviction. For the alert accumulator, we keep the last 500 alerts. The 501st alert causes the first to be dropped. Memory stays constant at the size of 500 alerts, forever, regardless of how long the pipeline runs.

The BoundedAlertAccumulator in Example 38.3 demonstrates this pattern. We run the pipeline for 10,000 records, triggering hundreds of alerts. After the run, `len(self._alerts)` is at most 500 — regardless of how many total alerts were triggered. The `_totalTriggered` counter records how many alerts occurred in total; the deque stores only the most recent 500.

The tracemalloc audit in Example 38.3 verifies the memory bound quantitatively. By comparing snapshots before and after the pipeline run, we can see exactly how many kilobytes were allocated and confirm that the growth is bounded — that memory does not increase with run duration.

---

There is one more principle I want to establish before we move on to Stage 6.

The discipline of Stage 5 is systematic. Every cache must have either a size limit or a TTL — or both. Every accumulator, buffer, and queue must declare its maximum size. These are not optional improvements for "later" — they are the difference between a pipeline that runs indefinitely and one that crashes unpredictably.

When you design a new component, ask: does this component accumulate data over time? If yes, what is the maximum amount it should accumulate, and what happens when that maximum is reached? Eviction? Blocking? Error? The answer is a design decision, and it must be explicit.

`functools.lru_cache` with `maxsize` — bounded by LRU eviction. `collections.deque(maxlen=N)` — bounded by automatic oldest-element eviction. `queue.Queue(maxsize=N)` — bounded by backpressure (producers block when full). `TTLCache(maxSize=N)` — bounded by oldest-expiry eviction. Each one is a specific choice about what to sacrifice when the limit is reached, and each choice is appropriate for specific use cases. Knowing which to use is the craft of production systems engineering.

In Stage 6, we add resilience: the ability of the pipeline to detect when a component has failed and recover gracefully. Retry logic with exponential backoff. Circuit breakers that we studied in the architecture patterns lecture, now applied concretely to the project. And graceful degradation — the discipline of producing a useful partial result when one component is down, rather than failing completely.

