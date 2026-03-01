# Slides — Lecture 39: Big Project Stage 6 — Resilience Layer

---

## Slide 1 — Stage 6 Goal: Resilient Pipeline

**Defined failure behavior instead of undefined crashes**

- In production: networks drop, databases deadlock, APIs rate-limit, workers crash
- Three failure profiles → three resilience patterns:
  - **Transient failures** (seconds) → Retry with exponential backoff
  - **Sustained failures** (minutes) → Circuit breaker (fail fast, probe recovery)
  - **Partial failures** (some components down) → Graceful degradation
- Goal: pipeline continues producing value even when components fail

---

## Slide 2 — Transient vs. Sustained vs. Partial Failures

**Different failure profiles require different handling**

| Failure Type | Duration | Example | Correct Pattern |
|-------------|----------|---------|-----------------|
| Transient | ms–seconds | Network blip, DB deadlock | Retry with backoff |
| Sustained | Minutes–hours | Service maintenance, rate limit | Circuit breaker |
| Partial | Ongoing | One of 5 sensors down | Graceful degradation |

- Retry alone on a sustained failure: pipeline serializes behind retry waits
- No retry on a transient failure: unnecessary permanent failures
- Binary fail/pass on partial failure: 1 component failure = 100% output loss

---

## Slide 3 — Retry with Exponential Backoff

**Wait longer after each failure; add jitter to prevent synchronized storms**

```python
def retryWithBackoff(maxAttempts=4, baseDelaySeconds=1.0, maxDelaySeconds=30.0):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(maxAttempts):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    if attempt < maxAttempts - 1:
                        delay = min(base * (2**attempt) + random.uniform(0,1), maxDelay)
                        time.sleep(delay)
```

Backoff schedule for baseDelay=1.0:
- Attempt 1 fails → wait ~1.0 + jitter seconds
- Attempt 2 fails → wait ~2.0 + jitter seconds
- Attempt 3 fails → wait ~4.0 + jitter seconds
- Attempt 4 fails → re-raise

**Without jitter:** 100 clients retry simultaneously → thundering herd

---

## Slide 4 — Example 39.1: FlakySensor with retryWithBackoff

**70% fail probability → 4 attempts → 0.7⁴ = 2.4% permanent failure rate**

```python
class FlakySensor:
    @retryWithBackoff(maxAttempts=4, baseDelaySeconds=0.1)
    def read(self):
        if random.random() < self._failProb:
            raise ConnectionError(f"Sensor timeout")
        return {"sensorId": self.sensorId, "value": random.gauss(50, 10)}
```

- Without retry: 70% of calls fail
- With 4 retries: 0.7 × 0.7 × 0.7 × 0.7 = **2.4% of calls fail** (30× improvement)
- @functools.wraps: preserved function name and docstring in stack traces
- Configurable exceptions: `exceptions=(ConnectionError, TimeoutError)` — don't retry programming errors

---

## Slide 5 — The Retry Failure: Sustained Outages

**Retry + backoff fails when the service is down for minutes, not seconds**

Scenario: service down for 30 minutes; records arrive every 1 second

```
Without circuit breaker:
  Each record: try → fail → wait 1s → fail → wait 2s → fail → wait 4s → fail → wait 8s → FAIL
  Total wait per record: 15 seconds
  1800 records × 15s = 27,000 seconds of blocked pipeline time

With circuit breaker:
  First 3 failures: real attempts (0.1s each)
  Circuit opens → all subsequent calls: fail in < 1ms
  Actual pipeline time lost: 3 real failures + 1797 fast-fails (< 2 seconds total)
```

---

## Slide 6 — Circuit Breaker: Three States

**Fail fast during open state; probe during half-open; normal when closed**

```
  failures < threshold          threshold reached
       ┌──────────────────────────────────────────┐
       │              CLOSED                      │
       │         (normal operation)               │
       └───────────────────┬──────────────────────┘
                           │ failures >= threshold
                           ▼
       ┌──────────────────────────────────────────┐
       │               OPEN                       │
       │         (fail fast — no I/O)             │◄── probe fails
       └───────────────────┬──────────────────────┘
                           │ resetTimeout elapsed
                           ▼
       ┌──────────────────────────────────────────┐
       │            HALF-OPEN                     │
       │     (one probe call allowed through)     │─── probe succeeds ──► CLOSED
       └──────────────────────────────────────────┘
```

---

## Slide 7 — Example 39.2: AsyncCircuitBreaker

**State-machine implementation with async integration**

```python
class AsyncCircuitBreaker:
    async def call(self, coroutineFn, *args, **kwargs):
        if self._state == "open":
            if elapsed >= self._resetTimeout:
                self._state = "half_open"      # allow one probe
            else:
                raise CircuitOpenError(...)    # fail fast — no I/O

        try:
            result = await coroutineFn(*args, **kwargs)
            if self._state == "half_open":
                self._state = "closed"         # probe succeeded: close
            return result
        except Exception:
            self._failureCount += 1
            if self._failureCount >= threshold:
                self._state = "open"           # too many failures: open
            raise
```

- `CircuitOpenError` is distinguishable from real service errors — handle differently in callers
- `time.monotonic()` for reset timeout — immune to clock adjustments

---

## Slide 8 — Graceful Degradation: Partial Failure Handling

**One failed component should not cause total output loss**

5-sensor pipeline, sensor S3 fails:

```
Without degradation:                With degradation:
  S0 → reads OK                      S0 → reads OK       ✓
  S1 → reads OK                      S1 → reads OK       ✓
  S2 → reads OK                      S2 → reads OK       ✓
  S3 → fails → CRASH                 S3 → fails → log + continue
  S4 → never read                    S4 → reads OK       ✓
  Output: 0 records                  Output: 4/5 sensors — flagged as DEGRADED
```

- Key: capture the fallback at the source layer — all downstream stages work normally
- Batch metadata: `"unavailableSensors": ["S3"], "degraded": True`
- Monitoring alert: "batch degraded" — but pipeline continues running

---

## Slide 9 — Example 39.3: Graceful Degradation in Multi-Sensor Collection

**readSensorWithFallback + asyncio.gather for parallel degraded collection**

```python
async def readSensorWithFallback(sensorId, breaker, failProb):
    try:
        # Attempt to read sensor
        return {"sensorId": sensorId, "value": reading, "status": "ok"}
    except Exception as e:
        logging.warning(f"Sensor {sensorId} failed: {e}")
        return {"sensorId": sensorId, "value": None, "status": "unavailable"}

async def collectBatch(sensorIds, failProbs):
    readings = await asyncio.gather(*[
        readSensorWithFallback(sid, ...) for sid in sensorIds
    ])
    available = [r for r in readings if r["status"] == "ok"]
    unavailable = [r["sensorId"] for r in readings if r["status"] == "unavailable"]
    return {"readings": available, "unavailableSensors": unavailable, "degraded": bool(unavailable)}
```

---

## Slide 10 — Stacking Resilience Patterns

**Retry + circuit breaker + graceful degradation work together**

```
                    readSensor(sensorId)
                          │
         ┌────────────────▼────────────────┐
         │  @retryWithBackoff(attempts=3)  │  ← handles transient failures
         │         ↓ all retries fail       │
         └────────────────┬────────────────┘
                          │ sustained failure
         ┌────────────────▼────────────────┐
         │      AsyncCircuitBreaker        │  ← stops retrying sustained failures
         │         ↓ circuit open          │
         └────────────────┬────────────────┘
                          │ any failure
         ┌────────────────▼────────────────┐
         │   readSensorWithFallback       │  ← returns sentinel, pipeline continues
         └─────────────────────────────────┘
```

---

## Slide 11 — Lecture 39 Key Principles

**What to carry into Stage 7**

- Retry + exponential backoff: transient failures — `2**attempt + jitter` seconds
- Never retry on programming errors (TypeError, ValueError) — only I/O failures
- Circuit breaker: sustained failures — open (fast fail), half-open (probe), closed (normal)
- Circuit breaker + retry: stack them — retry first, circuit breaker for sustained failure
- Graceful degradation: partial failures — fallback sentinel, batch metadata, downstream works normally
- Production rule: every external dependency needs retry logic + a timeout
- Next: Stage 7 — observability: structured logging, health dashboard, introspection

---
