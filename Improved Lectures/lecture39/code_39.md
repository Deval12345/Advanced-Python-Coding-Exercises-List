# Code — Lecture 39: Big Project Stage 6 — Resilience Layer

---

## Example 39.1 — Retry Decorator with Exponential Backoff

```python
# Example 39.1
import time
import random
import functools
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")

def retryWithBackoff(maxAttempts=4, baseDelaySeconds=1.0, maxDelaySeconds=30.0, exceptions=(Exception,)):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            lastException = None
            for attempt in range(maxAttempts):
                try:
                    return fn(*args, **kwargs)
                except exceptions as e:
                    lastException = e
                    if attempt < maxAttempts - 1:
                        delay = min(baseDelaySeconds * (2 ** attempt) + random.uniform(0, 1), maxDelaySeconds)
                        logging.info(f"Attempt {attempt+1}/{maxAttempts} failed: {e}. Retrying in {delay:.2f}s")
                        time.sleep(delay)
                    else:
                        logging.info(f"All {maxAttempts} attempts exhausted: {e}")
            raise lastException
        return wrapper
    return decorator

class FlakySensor:
    def __init__(self, sensorId, failProbability=0.7):
        self.sensorId = sensorId
        self._failProb = failProbability

    @retryWithBackoff(maxAttempts=4, baseDelaySeconds=0.1)
    def read(self):
        if random.random() < self._failProb:
            raise ConnectionError(f"Sensor {self.sensorId}: connection timeout")
        return {"sensorId": self.sensorId, "value": random.gauss(50, 10)}

if __name__ == "__main__":
    sensor = FlakySensor("S1", failProbability=0.6)
    successes = 0
    failures = 0
    for _ in range(10):
        try:
            reading = sensor.read()
            print(f"Success: {reading}")
            successes += 1
        except ConnectionError as e:
            print(f"All retries failed: {e}")
            failures += 1
    print(f"\nResults: {successes} successes, {failures} permanent failures out of 10 reads")
```

### Line-by-Line Explanation

**`def retryWithBackoff(maxAttempts=4, baseDelaySeconds=1.0, maxDelaySeconds=30.0, exceptions=(Exception,)):`**
Decorator factory with configurable parameters. `maxAttempts` caps the total number of tries. `baseDelaySeconds` is the base wait before doubling. `maxDelaySeconds` prevents unbounded backoff (e.g. after 10 failures, `2^10 = 1024` seconds would be unreasonable — capped at 30). `exceptions` is a tuple of exception types to catch and retry; exceptions not in this tuple propagate immediately (you do not want to retry `TypeError` or `ValueError` — those are programming errors, not transient failures).

**`for attempt in range(maxAttempts):`**
The retry loop. `attempt` runs from 0 to `maxAttempts - 1`. On success, `return fn(*args, **kwargs)` exits the function immediately — no further iterations. On failure, the except clause handles the exception and either schedules a retry or re-raises.

**`delay = min(baseDelaySeconds * (2 ** attempt) + random.uniform(0, 1), maxDelaySeconds)`**
The core of exponential backoff with jitter. `2 ** attempt` doubles the delay each iteration: attempt 0 → 1, attempt 1 → 2, attempt 2 → 4, attempt 3 → 8. `random.uniform(0, 1)` adds up to 1 second of jitter — prevents synchronized retries from multiple concurrent callers. `min(..., maxDelaySeconds)` caps the delay to prevent pathological waits.

**`if attempt < maxAttempts - 1:`**
On the last attempt, skip the delay and re-raise immediately. This prevents waiting after the final failure — there is no point sleeping before re-raising.

**`raise lastException`**
Re-raises the exception from the final attempt. At this point, `lastException` holds the most recent exception from the last try. The re-raise preserves the original exception type and message, making the caller's error handling straightforward.

**`@retryWithBackoff(maxAttempts=4, baseDelaySeconds=0.1)`**
Applied to `FlakySensor.read()`. The `baseDelaySeconds=0.1` is small for demonstration purposes (0.1s, 0.2s, 0.4s, 0.8s between attempts). In production, `baseDelaySeconds=1.0` is typical. The decorator is applied at class definition time — `read` is wrapped once when the class is defined, not once per instance.

---

## Example 39.2 — AsyncCircuitBreaker for the Pipeline

```python
# Example 39.2
import asyncio
import time
import random
import logging

class CircuitOpenError(Exception):
    pass

class AsyncCircuitBreaker:
    def __init__(self, failureThreshold=3, resetTimeoutSeconds=5.0):
        self._state = "closed"
        self._failureCount = 0
        self._failureThreshold = failureThreshold
        self._lastFailureTime = None
        self._resetTimeout = resetTimeoutSeconds

    @property
    def state(self):
        return self._state

    async def call(self, coroutineFn, *args, **kwargs):
        if self._state == "open":
            if time.monotonic() - self._lastFailureTime >= self._resetTimeout:
                self._state = "half_open"
                logging.info("Circuit: OPEN → HALF_OPEN (probing)")
            else:
                raise CircuitOpenError(f"Circuit open — fast fail (reset in {self._resetTimeout}s)")

        try:
            result = await coroutineFn(*args, **kwargs)
            if self._state == "half_open":
                self._state = "closed"
                self._failureCount = 0
                logging.info("Circuit: HALF_OPEN → CLOSED (probe succeeded)")
            else:
                self._failureCount = 0
            return result
        except Exception as e:
            self._failureCount += 1
            self._lastFailureTime = time.monotonic()
            if self._failureCount >= self._failureThreshold or self._state == "half_open":
                self._state = "open"
                logging.info(f"Circuit: → OPEN (failures={self._failureCount}): {e}")
            raise

async def flakySensorService(sensorId, failProb=0.5):
    await asyncio.sleep(0.05)
    if random.random() < failProb:
        raise ConnectionError(f"Sensor {sensorId} timeout")
    return {"sensorId": sensorId, "value": random.gauss(50, 10)}

async def main():
    breaker = AsyncCircuitBreaker(failureThreshold=3, resetTimeoutSeconds=2.0)
    results = []
    for i in range(20):
        try:
            reading = await breaker.call(flakySensorService, "S1", failProb=0.7)
            results.append(("success", reading["value"]))
        except CircuitOpenError as e:
            results.append(("fast_fail", str(e)))
        except ConnectionError as e:
            results.append(("real_failure", str(e)))
        await asyncio.sleep(0.1)

    print(f"Circuit breaker state after 20 calls: {breaker.state}")
    for i, (outcome, detail) in enumerate(results):
        print(f"  Call {i+1:2d}: {outcome}")

asyncio.run(main())
```

### Line-by-Line Explanation

**`class CircuitOpenError(Exception):`**
A dedicated exception class for circuit-open fast-fails. This is important: callers can catch `CircuitOpenError` separately from real service errors (`ConnectionError`). The distinction matters — a `CircuitOpenError` means "we chose not to try"; a `ConnectionError` means "we tried and failed." The handling may differ: a `CircuitOpenError` might use a cached result or return degraded output; a `ConnectionError` might trigger an alert.

**`self._state = "closed"`**
The circuit starts closed — normal operation. The three valid states are "closed", "open", and "half_open". Python strings are used here for clarity; an `enum.Enum` would be more rigorous in production code.

**`if self._state == "open": if elapsed >= self._resetTimeout: self._state = "half_open"`**
The transition from open to half_open. `time.monotonic() - self._lastFailureTime` is the time since the last failure. When this exceeds `_resetTimeout`, the circuit allows one probe through. The transition to "half_open" happens lazily — on the next call — rather than on a timer. This is simpler and correct.

**`raise CircuitOpenError(...)`**
The fast-fail path. This code path executes in microseconds — no coroutine awaiting, no network I/O. The caller receives an immediate error indicating the circuit is open. This preserves the caller's resources and allows the failing service time to recover.

**`result = await coroutineFn(*args, **kwargs)`**
The actual async service call. `coroutineFn` is a reference to the async function (e.g. `flakySensorService`); `*args, **kwargs` forward the caller's arguments. The `await` suspends the breaker coroutine and returns control to the event loop while the service call is in progress.

**`if self._state == "half_open": self._state = "closed"; self._failureCount = 0`**
The successful probe transitions the circuit from half_open back to closed. `_failureCount = 0` resets the failure counter — the circuit starts fresh with no accumulated failures.

**`if self._failureCount >= self._failureThreshold or self._state == "half_open":`**
Two conditions trigger the open state: (1) accumulated failures reach the threshold — normal circuit opening; (2) any failure in half_open state — the probe failed, so the service is still down, re-open immediately.

---

## Example 39.3 — Graceful Degradation in Multi-Sensor Collection

```python
# Example 39.3
import asyncio
import random
import logging
import time

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")

async def readSensorWithFallback(sensorId, breaker, failProb=0.3):
    try:
        await asyncio.sleep(0.05)
        if random.random() < failProb:
            raise ConnectionError(f"Sensor {sensorId} unavailable")
        return {"sensorId": sensorId, "value": random.gauss(50, 10), "status": "ok"}
    except (ConnectionError, Exception) as e:
        logging.warning(f"Sensor {sensorId} failed: {e} — using fallback")
        return {"sensorId": sensorId, "value": None, "status": "unavailable"}

async def collectBatch(sensorIds, failProbabilities):
    tasks = [
        readSensorWithFallback(sid, None, failProbabilities.get(sid, 0.1))
        for sid in sensorIds
    ]
    readings = await asyncio.gather(*tasks)

    available = [r for r in readings if r["status"] == "ok"]
    unavailable = [r["sensorId"] for r in readings if r["status"] == "unavailable"]

    batchResult = {
        "readings": available,
        "unavailableSensors": unavailable,
        "degraded": len(unavailable) > 0,
        "availableCount": len(available),
        "totalSensors": len(sensorIds),
    }
    return batchResult

async def main():
    sensorIds = [f"S{i}" for i in range(5)]
    failProbs = {"S0": 0.1, "S1": 0.9, "S2": 0.1, "S3": 0.8, "S4": 0.1}

    for batchNum in range(5):
        result = await collectBatch(sensorIds, failProbs)
        status = "DEGRADED" if result["degraded"] else "HEALTHY"
        print(f"Batch {batchNum+1}: {status} — "
              f"{result['availableCount']}/{result['totalSensors']} sensors OK "
              f"— unavailable: {result['unavailableSensors'] or 'none'}")

asyncio.run(main())
```

### Line-by-Line Explanation

**`async def readSensorWithFallback(sensorId, breaker, failProb=0.3):`**
Each sensor's read function is individually wrapped in degradation logic. The function always returns a dict — never raises an exception that would propagate to the batch collector. This is the key design choice: errors are captured at the source and converted to sentinel values, preventing one sensor failure from affecting others.

**`return {"sensorId": sensorId, "value": None, "status": "unavailable"}`**
The fallback sentinel. `value: None` is an explicit marker — downstream stages can check for `None` if needed, but in this architecture, the batch collector separates available from unavailable readings before passing data downstream, so stages never see `None` values.

**`except (ConnectionError, Exception) as e:`**
Catches all exceptions broadly — `ConnectionError` explicitly for sensor network failures, `Exception` as a catch-all for any unexpected error. In production, you would be more specific about which exceptions to catch: catch `ConnectionError`, `TimeoutError`, and known service-specific exceptions; let `SystemExit` and `KeyboardInterrupt` propagate.

**`readings = await asyncio.gather(*tasks)`**
All five sensor reads run simultaneously. `asyncio.gather` waits for all to complete — but because each `readSensorWithFallback` catches its own exceptions and returns a sentinel, `gather` never raises. All five coroutines complete, each returning either a successful reading or a fallback sentinel.

**`available = [r for r in readings if r["status"] == "ok"]`**
Separates successful readings from failed ones at the batch level. Downstream stages receive only `available` — they process normal data without knowing that some sensors failed.

**`"degraded": len(unavailable) > 0`**
Explicit boolean flag in batch metadata. Monitoring systems can alert on "degraded" batches without parsing the sensor list. `"unavailableSensors": ["S1", "S3"]` provides the detail needed for root-cause analysis.

---
