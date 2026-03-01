# Lecture 39: Big Project Stage 6 — Resilience Layer
## speech_39.md — Instructor Spoken Narrative

---

Everything we have built so far assumes the world cooperates. Sensor readings arrive on time. Database lookups succeed. Network connections hold. The process pool workers stay alive.

In development, this is usually true. In production, it is not.

Networks have latency spikes and transient timeouts. Databases have momentary deadlocks. External APIs hit rate limits. A process pool worker can crash due to a memory error or a hardware fault. In a pipeline that runs continuously — 24 hours a day, 7 days a week — something will fail. The question is not whether your pipeline encounters failures. It is what your pipeline does when it does.

Today is Stage 6: the resilience layer. Three patterns, each addressing a different failure profile.

---

The first pattern is retry with exponential backoff.

Some failures are transient. A network timeout that resolves in 2 seconds. A database deadlock that clears in 500 milliseconds. A momentary spike in API latency. These failures are self-correcting if you wait a short time and try again. The retry decorator handles exactly this case.

The key insight is the backoff strategy. If you retry immediately — zero wait — you add more load to a system that is already struggling. If you retry at a fixed interval — say, every 1 second — and a hundred clients are doing the same thing, they all retry simultaneously, creating synchronized retry storms that overwhelm the recovering system. Exponential backoff with jitter solves both problems. After the first failure, wait 1 second plus a small random amount. After the second failure, wait 2 seconds plus jitter. After the third, 4 seconds. Each retry doubles the wait. The jitter — a random component — ensures that clients do not all retry at the exact same moment, spreading the load.

In Example 39.1, the `retryWithBackoff` decorator wraps any function in this logic. It uses `functools.wraps` to preserve the wrapped function's identity. It accepts configurable parameters: maximum attempts, base delay, maximum delay (to prevent unbounded backoff), and the specific exception types to catch. On each failure, it logs the attempt number and the wait time, sleeps, and retries. On exhausting all attempts, it re-raises the last exception.

The `FlakySensor` class demonstrates this with a sensor that fails 60% of the time. Applying `@retryWithBackoff(maxAttempts=4, baseDelaySeconds=0.1)` to its `read` method means: on each call, we try up to four times. With a 60% failure rate per attempt, the probability of failing all four is `0.6⁴ = 0.13` — a 13% chance of permanent failure per call, versus 60% without retry. The pipeline is dramatically more reliable without changing the underlying sensor.

---

But exponential backoff only works for transient failures. What happens when the failure is sustained?

Imagine the data enrichment service goes down for maintenance. It will be unavailable for 30 minutes. The retry decorator tries once, waits 1 second, tries again, waits 2 seconds, tries again, waits 4 seconds, tries a fourth time, waits 8 seconds. Total time per record: 15 seconds of accumulated waiting. If a new record arrives every second, the pipeline is now completely serialized — blocked behind retry waits. After 30 minutes, the retry attempts have consumed 1800 × 15 = 27,000 seconds of pipeline time.

The circuit breaker solves this. After a threshold number of failures in a time window — say, 3 failures in 5 seconds — the circuit "opens." In the open state, all calls to the failing service fail immediately — in under a millisecond — without waiting, without retrying. The pipeline moves on. After a reset interval — say, 5 seconds — the circuit enters the "half-open" state: it allows one probe call through. If the probe succeeds, the circuit "closes" and normal operation resumes. If the probe fails, the circuit opens again.

In Example 39.2, `AsyncCircuitBreaker` tracks state as a string: "closed", "open", or "half_open". The `call` method checks state first. If open and the reset interval has not elapsed, it raises `CircuitOpenError` immediately — no I/O, no wait, no network call. If open and the reset interval has elapsed, it transitions to half_open and allows the next call through. On success in half_open, it closes. On failure in half_open, it reopens.

Notice that the circuit breaker and the retry decorator are complementary. The retry decorator handles "failed, might succeed in a few seconds." The circuit breaker handles "has been failing, give it time to recover." In production, you typically stack them: the retry decorator wraps individual calls; the circuit breaker wraps the retry decorator. A transient failure triggers a retry; a sustained failure triggers the circuit breaker.

---

The third pattern is the most nuanced: graceful degradation.

Consider a five-sensor pipeline. If sensor S3 fails — its network connection drops — a naive pipeline crashes completely. No readings from any sensor are processed. The monitoring team is paged. Someone intervenes, restarts the pipeline. During the outage, no analytics are produced.

The gracefully degraded pipeline does something different. It attempts to read from S3. S3 fails. The pipeline logs the failure, marks S3 as "unavailable" in the batch metadata, and continues processing readings from S0, S1, S2, and S4. The batch output is flagged as "degraded" — 4 of 5 sensors available — and the downstream analytics receive partial data with explicit metadata about what is missing. The monitoring team is still notified — the "degraded" flag triggers an alert — but the pipeline continues to produce value.

Eighty percent of the pipeline's output is preserved. In a real-time sensor system, this can mean the difference between a partial equipment monitoring gap and a complete monitoring blackout.

Example 39.3 shows this pattern in the async multi-sensor pipeline. `readSensorWithFallback` wraps each sensor read in try/except. On success, it returns a reading with `status: "ok"`. On failure, it logs a warning and returns a sentinel with `status: "unavailable"` and `value: None`. The `collectBatch` function runs all five sensor reads simultaneously with `asyncio.gather`, then separates successful reads from failed ones. The batch result carries both the available readings and the metadata about which sensors were unavailable.

The downstream stages receive only the available readings — they do not need to handle the `None` fallback values. The degradation is isolated to the collection layer; all subsequent stages work normally on whatever data arrives.

---

Let me step back and address a question that sometimes comes up at this point. Why build all this complexity into the pipeline? Why not just let it crash and restart?

The answer depends on your requirements. For a batch job that runs once a day, crash and restart might be acceptable — the retry is manual, but the job runs infrequently enough that the overhead is acceptable. For a continuous streaming pipeline that processes real-time sensor data, crash and restart means missing data during the restart window, potentially losing minutes or hours of readings if the failure recurs. For a pipeline that other systems depend on — that drives real-time alerting or equipment safety monitoring — an outage of any duration has real consequences.

Resilience is not about anticipating every possible failure. It is about building a system that has a defined behavior for failure, rather than an undefined behavior. "Crash and restart" is a defined behavior — it just often produces long outages. "Retry then circuit break then degrade gracefully" is a defined behavior that produces much shorter outages and continued partial value during degraded periods.

The correct level of resilience depends on your system's requirements. But the patterns — retry, circuit breaker, graceful degradation — are the building blocks. You choose which to apply based on the failure modes you need to handle.

---

Stage 6 integrates these three patterns into the analytics engine. Every sensor read is decorated with retry logic. Downstream service calls are wrapped in circuit breakers. Batch collection uses the graceful degradation pattern. The pipeline's failure modes are now defined and handled, not undefined and catastrophic.

In Stage 7, we add observability — the tooling that lets you see what is happening inside the resilience layer. How often is the circuit breaker opening? What is the retry success rate? How many batches in the last hour were degraded, and which sensors caused the degradation? That is the information you need to make informed operational decisions. And it requires a structured observability layer, which is exactly what we build next.

