# Speech Source — Lecture 39: Big Project Stage 6 — Resilience Layer

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Stage 5 we added caching and bounded memory — the analytics engine now amortizes expensive repeated computations and has a declared memory budget for all its data structures. Today is Stage 6: resilience. The pipeline runs continuously. Over hours and days, components fail. A sensor's network connection drops. A downstream database becomes temporarily unavailable. A data enrichment service times out. A process pool worker crashes. In a system with no resilience layer, any of these events causes the entire pipeline to fail — silently, or with an exception that propagates until it crashes the main process. In a resilient system, these events are handled: retried automatically, failed fast when recovery is impossible, and degraded gracefully so that the pipeline continues to produce useful output even when some components are down.

---

## CONCEPT 1.1 — Retry with Exponential Backoff

**Problem it solves:**
Transient failures are the most common failure mode in distributed systems. A network timeout, a database deadlock, a momentary resource exhaustion — these failures are self-correcting if you wait a moment and try again. But the right waiting strategy matters. Immediately retrying puts additional load on a system that may already be struggling. A fixed retry interval creates synchronized retry storms when many clients fail simultaneously. Exponential backoff with jitter is the correct strategy: after the first failure, wait 1 second; after the second failure, wait 2 seconds; after the third, wait 4 seconds; each retry doubles the wait time. Jitter adds a random component to prevent synchronized retries from multiple clients.

**Why invented:**
Exponential backoff was formalized in the context of Ethernet collision avoidance (the CSMA/CD algorithm, 1980). The intuition is correct for any shared resource: if many clients are contending and failing simultaneously, spreading out their retries reduces contention and allows recovery. AWS, Google Cloud, and Azure all specify exponential backoff with jitter as the required retry strategy for their SDKs. It is the industry standard precisely because flat retry intervals create "thundering herd" problems — all clients retry simultaneously, overwhelming the recovering service.

**What happens without it:**
Without exponential backoff, retrying immediately after failure amplifies the problem. A database that is overloaded and timing out 100 client connections receives 100 immediate retries — and 100 more — creating more load than it can handle at exactly the moment it is trying to recover. The recovery never completes. With exponential backoff, clients wait progressively longer. After a few seconds, the load drops to a level the recovering database can handle, and recovery completes.

**Industry impact:**
AWS SDK (Boto3), Google Cloud client libraries, and Azure SDK all implement exponential backoff with jitter by default. RFC 6298 (TCP retransmission timeout) uses exponential backoff. gRPC's default retry policy uses exponential backoff. The Python `tenacity` library provides a production-grade retry decorator with exponential backoff, jitter, stop conditions, and exception filtering. The simple implementation — `wait = min(base ** attempt + random.uniform(0, 1), maxWait)` — is the starting point for any retry infrastructure.

---

## EXAMPLE 1.1 — Retry Decorator with Exponential Backoff

Narration: We implement a `retryWithBackoff` decorator that wraps any function call in retry logic. On exception, it waits for `base ** attempt` seconds plus random jitter (uniform between 0 and 1 second). It retries up to `maxAttempts` times. On each retry it logs the attempt number, wait time, and exception message. On success it returns the result normally. On exhausting all retries it re-raises the last exception. We demonstrate with a `FlakySensor` that fails 70% of the time and succeeds 30% of the time. The decorator makes the pipeline resilient to these transient failures.

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

---

## CONCEPT 2.1 — Circuit Breaker Applied to the Pipeline

**Problem it solves:**
The retry decorator handles transient failures — failures that resolve quickly. But some failures are not transient. A downstream service is down for maintenance. A data enrichment API has hit its rate limit and will not recover for ten minutes. A database is undergoing a failover that will take 30 seconds. For these sustained failures, retrying with backoff still burns time and resources — waiting 1, 2, 4, 8, 16 seconds while the service is known-down. The circuit breaker complements the retry decorator: the retry decorator handles transient failures (seconds), while the circuit breaker handles sustained failures (minutes or hours).

**Why invented:**
The circuit breaker pattern was described by Michael Nygard in "Release It!" (2007). It was motivated by the observation that in microservice architectures, one failing dependency often causes a cascade: the clients of the failing service queue up requests, holding resources, preventing recovery. The circuit breaker is the software equivalent of the electrical circuit breaker — when failure rate exceeds a threshold, it "opens" and all subsequent requests fail immediately without contacting the failing service. This gives the service time to recover without load and releases the client's resources.

**What happens without it:**
Without a circuit breaker, the retry decorator keeps attempting. After 4 attempts with exponential backoff (1 + 2 + 4 + 8 = 15 seconds of waiting), the function fails. If the service is down for 30 minutes and a new record arrives every second, there are 1800 retry sequences in 30 minutes — 1800 × 15 seconds = 27,000 seconds of accumulated blocked time. The pipeline is effectively serialized behind retry waits. With a circuit breaker, after 5 failures in 10 seconds, the circuit opens. Subsequent calls fail immediately (< 1ms). After a reset interval, one probe request goes through. If the service is back, the circuit closes. The 30-minute outage costs 5 real failures and all subsequent attempts cost < 1ms each.

**Industry impact:**
Netflix's Hystrix (now Resilience4j) popularized circuit breakers in JVM microservice architectures. The Python `circuitbreaker` package and `pybreaker` provide similar functionality. In async Python pipelines, circuit breakers are commonly implemented as decorators that wrap the async call with state tracking. AWS Service Control Policies implement circuit-breaker-like throttling at the infrastructure level.

---

## EXAMPLE 2.1 — AsyncCircuitBreaker for the Pipeline

Narration: We implement an AsyncCircuitBreaker as a class with three states: closed (normal operation), open (failing fast), and half_open (probing for recovery). The breaker wraps the actual async call with state-dependent logic. We integrate it with a flaky async sensor service and demonstrate the three-state transition: calls succeed while closed, failures accumulate until the circuit opens, subsequent calls fail fast without waiting, after a reset interval one probe succeeds and the circuit closes.

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

---

## CONCEPT 3.1 — Graceful Degradation: Partial Failure Is Better Than Total Failure

**Problem it solves:**
When a pipeline component fails, the binary choice of "run normally" vs. "crash the pipeline" is too coarse. A sensor pipeline with five sensors can produce useful analytics with four sensors — it is degraded but functional. A data enrichment pipeline that cannot reach the enrichment service can still process records without enrichment — returning records without the enriched field. A feature computation stage that fails for one record type can still process other record types. Graceful degradation is the discipline of identifying partial failure modes and designing explicit fallback behavior for each one — so that the pipeline continues to produce useful output when components fail.

**Why invented:**
Graceful degradation emerged from reliability engineering and has been a design principle in high-availability systems since at least the 1970s. NASA's Apollo guidance computer implemented graceful degradation — certain sensors could fail without mission abort. Modern web systems implement it through fallback values, cached stale data, simplified feature sets, and partial result acceptance. The principle: a system that produces 90% of its normal output during a partial failure is far more valuable than a system that produces 0% output while one component is down.

**What happens without it:**
Without graceful degradation, one failing sensor crashes the pipeline. The monitoring team gets an alert; someone intervenes; the pipeline is restarted. If the sensor failure is intermittent, the pipeline repeatedly crashes and restarts, producing no output during the outage. With graceful degradation, the pipeline logs that sensor S3 is unavailable, continues processing sensors S1, S2, S4, and S5, and flags the output as "sensor S3 missing" in the result metadata. The monitoring team is still notified, but the pipeline continues to produce value.

**Industry impact:**
Graceful degradation is a core principle in Netflix's chaos engineering practice. Netflix routinely disables individual services in production to verify that the system degrades gracefully rather than failing completely. Amazon's "Cell-Based Architecture" is designed so that failures in one region's components do not prevent other regions from functioning. In Python data pipelines, graceful degradation is implemented through exception handling at the stage level, fallback return values, and health status tracking.

---

## EXAMPLE 3.1 — Graceful Degradation in the Multi-Sensor Pipeline

Narration: We have five sensors. Each sensor read is wrapped with the retry decorator and circuit breaker from Examples 39.1 and 39.2. When a sensor fails all retries, instead of crashing the batch, we catch the failure, log it, and continue with the remaining sensors. The batch result includes a "sensorStatus" dict mapping each sensorId to either "ok" or "unavailable". Downstream stages receive only the readings that succeeded. Alerts include metadata about which sensors were unavailable. The pipeline never crashes; it degrades.

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

---

## CONCEPT 4.1 — Final Takeaway: Lecture 39

Stage 6 adds resilience to the pipeline: the ability to recover from failure without human intervention. Three patterns: retry with exponential backoff for transient failures; circuit breaker for sustained failures; graceful degradation for partial failures. These patterns are complementary, not alternatives. A robust resilience layer uses all three: retry for the first few failures, circuit breaker to stop retrying after a sustained failure, and graceful degradation to keep the pipeline productive while recovery happens.

The design principle: failure is not exceptional — it is normal. In a system running continuously with many external dependencies, some dependency will fail at some point. The question is not whether to handle failures but how. A system designed for resilience is one where every component's failure modes are enumerated, and each has an explicit handling strategy.

In the next stage (Lecture 40), we add observability: structured logging, runtime introspection, and a pipeline health dashboard. Observability is the tool that reveals what is happening when the resilience layer is active — which sensors are failing, how often the circuit breaker opens, what the retry success rates are.

