# Speech Source — Lecture 24: High-Level Concurrency APIs — Controlling Async at Scale

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 23 we explored Futures and Executors: how asyncio.Future represents a placeholder for a result that does not exist yet, how ProcessPoolExecutor distributes CPU-bound work across multiple processes, and how to bridge synchronous executor results back into an async event loop using loop.run_in_executor and asyncio.wrap_future. You now understand the full mechanical stack — coroutines, the event loop, and the executor bridge. Today we step up one level of abstraction. Writing async code that works is one skill. Writing async code that is safe, bounded, and reliable under real-world scale constraints is another. This lecture covers the coordination and control tools that make that possible: Semaphore for rate limiting, wait() for fine-grained task coordination, wait_for for timeouts, Queue for producer-consumer pipelines, and Event for one-shot signalling between coroutines.

---

## CONCEPT 1.1 — The Problem of Unbounded Concurrency

**Problem it solves:** When you write asyncio.gather followed by a list comprehension that spawns ten thousand coroutines, every one of those coroutines starts immediately. Each one opens a network connection. The target server receives ten thousand simultaneous connection requests and responds with HTTP 429 — Too Many Requests. Your own process exhausts its file descriptor limit. The "feature" of async concurrency — that you can have many tasks in flight at once — becomes a liability the moment there is no upper bound on that "many."

**Why invented:** Real systems have resource constraints at every layer. APIs publish rate limits: one hundred requests per second, five concurrent connections per key. Database connection pools have a maximum size: twenty connections, no more. Operating systems enforce file descriptor limits per process. asyncio.Semaphore was created to encode these real-world ceilings directly into your concurrency model, so that your async code respects the same limits that the systems around it impose.

**What happens without it:** A well-intentioned async scraper with no concurrency limit becomes a de facto distributed denial-of-service attack on the target server. With ten thousand coroutines all trying to connect simultaneously, the target's connection queue overflows. Rate limiters engage. Your IP gets blocked. Meanwhile, your own process may crash with "Too many open files" — a cryptic error that traces back to the fundamental fact that every open socket consumes a file descriptor.

**Industry impact:** asyncio.Semaphore is the standard solution. The analogy is a parking lot with N spaces. Each arriving car — each coroutine — acquires a space by entering the semaphore's critical section. If all spaces are taken, new cars wait at the entrance. As a car leaves, it releases its space, and the next waiting car can enter. The parking lot never overflows; it simply regulates the rate of entry. In production Python code, Semaphore-wrapped HTTP clients are the default pattern for any fan-out operation against a rate-limited external API.

---

## EXAMPLE 1.1 — asyncio.Semaphore for rate limiting

Narration: We set a maximum of three concurrent sessions and create the semaphore with that limit. Each coroutine enters the semaphore with async with — a blocking operation if the semaphore is at capacity. Once inside, it prints a timestamped start message, simulates a 0.3-second network call, prints completion, and returns its result. In main, we build nine coroutines and pass them all to gather at once. Without the semaphore, all nine would run simultaneously. With it, the event loop can only have three inside the critical section at any moment. The first three enter immediately. The remaining six wait. As each of the first three finishes and exits the async with block, one waiting coroutine is admitted. The output timestamps confirm the batching: three sessions start at roughly zero seconds, three more at roughly 0.3 seconds, three more at roughly 0.6 seconds. Total time is approximately 0.9 seconds — three sequential batches of three — rather than the 0.3 seconds of true unbounded concurrency, but without the collateral damage.

```python
# Example 24.1
import asyncio                                         # line 1
import time                                            # line 2

MAX_CONCURRENT = 3                                     # line 3
semaphore = asyncio.Semaphore(MAX_CONCURRENT)          # line 4

async def fetchWithLimit(sessionId, delay):            # line 5
    async with semaphore:                              # line 6
        print(f"[{time.perf_counter():.2f}] Session {sessionId}: started")  # line 7
        await asyncio.sleep(delay)                     # line 8
        print(f"[{time.perf_counter():.2f}] Session {sessionId}: done")  # line 9
        return f"result_{sessionId}"                   # line 10

async def main():                                      # line 11
    tasks = [                                          # line 12
        fetchWithLimit(sessionId=i, delay=0.3)         # line 13
        for i in range(9)                              # line 14
    ]                                                  # line 15
    start = time.perf_counter()                        # line 16
    results = await asyncio.gather(*tasks)             # line 17
    elapsed = time.perf_counter() - start              # line 18
    print(f"All done in {elapsed:.2f}s")               # line 19
    print(f"Results: {results}")                       # line 20

asyncio.run(main())                                    # line 21
```

---

## CONCEPT 2.1 — asyncio.wait — Fine-Grained Task Coordination

**Problem it solves:** asyncio.gather is convenient but opinionated. It waits for every task to finish before returning. If one task raises, gather cancels the rest by default. You cannot inspect results as tasks complete, and you cannot act on the first success without waiting for everyone else. For many real coordination patterns — racing multiple redundant requests, failing fast on the first error, processing results as they arrive — gather is too blunt an instrument.

**Why invented:** asyncio.wait was designed to return control to the programmer as soon as a meaningful event occurs in the task set. The return_when parameter encodes three common coordination strategies: FIRST_COMPLETED for race patterns, FIRST_EXCEPTION for fail-fast error propagation, and ALL_COMPLETED for flexible gathering that lets you inspect individual task results and exceptions separately rather than having gather re-raise them.

**What happens without it:** Without wait, implementing a CDN race pattern requires either hacking gather with shared state and manual cancellation, or building a custom coordination loop from scratch using low-level Future callbacks. Both approaches are fragile and verbose. wait encapsulates the pattern cleanly: give it a set of tasks, tell it what event to wait for, receive two sets back — done and pending — and act on them immediately.

**Industry impact:** The "race" pattern is ubiquitous in latency-sensitive distributed systems. A global content delivery network submits the same request to multiple regional endpoints simultaneously and uses whichever responds first. A database client submits a read to a primary and a replica simultaneously and uses the first response. With asyncio.wait(return_when=FIRST_COMPLETED), the winning task's result is available immediately and the losing tasks are cancelled, freeing their resources cleanly.

---

## EXAMPLE 2.1 — asyncio.wait with FIRST_COMPLETED race pattern

Narration: We create three tasks, each simulating a query to a different geographic region — us-east with a base delay of 0.2 seconds, eu-west at 0.3 seconds, ap-south at 0.4 seconds — each with random jitter added so the winner varies between runs. We pass all three tasks to asyncio.wait with return_when set to FIRST_COMPLETED. The event loop runs all three concurrently. The moment the first one finishes, wait returns immediately with that task in the done set and the other two in the pending set. We pop the winning task from done and print its result. We then cancel every task in pending and await them through gather with return_exceptions=True — this gives the event loop the opportunity to deliver the cancellation and let the cancelled tasks clean up before we exit. Without that final gather, we would leave dangling tasks that the event loop would warn about on shutdown.

```python
# Example 24.2
import asyncio                                         # line 1
import time                                            # line 2
import random                                          # line 3

async def queryRegion(regionName, baseDelay):          # line 4
    jitter = random.uniform(0, 0.3)                    # line 5
    delay = baseDelay + jitter                         # line 6
    await asyncio.sleep(delay)                         # line 7
    return f"Response from {regionName} in {delay:.2f}s"  # line 8

async def main():                                      # line 9
    regions = [                                        # line 10
        asyncio.create_task(queryRegion("us-east", 0.2)),   # line 11
        asyncio.create_task(queryRegion("eu-west", 0.3)),   # line 12
        asyncio.create_task(queryRegion("ap-south", 0.4)),  # line 13
    ]                                                  # line 14

    done, pending = await asyncio.wait(                # line 15
        regions,                                       # line 16
        return_when=asyncio.FIRST_COMPLETED            # line 17
    )                                                  # line 18

    fastestTask = done.pop()                           # line 19
    print(f"Winner: {fastestTask.result()}")            # line 20

    for task in pending:                               # line 21
        task.cancel()                                  # line 22
    await asyncio.gather(*pending, return_exceptions=True)  # line 23
    print(f"Cancelled {len(pending)} slower tasks")    # line 24

asyncio.run(main())                                    # line 25
```

---

## CONCEPT 3.1 — asyncio.timeout and asyncio.wait_for

**Problem it solves:** A coroutine waiting on a network call can wait forever if the target server is down, overloaded, or has entered an infinite retry loop. Without an explicit timeout, that coroutine holds its slot in the event loop indefinitely. In a server handling thousands of concurrent requests, a single hung coroutine is a minor annoyance; a flood of them — all waiting on a broken external dependency — is a cascading failure that takes the entire service down.

**Why invented:** asyncio.wait_for wraps any coroutine and raises asyncio.TimeoutError after a specified number of seconds if the coroutine has not returned. Python 3.11 added asyncio.timeout as a composable context manager, allowing timeouts to be applied to blocks of code rather than single coroutines. asyncio.timeout_at accepts an absolute deadline timestamp rather than a relative duration, useful when you have a fixed wall-clock deadline shared across multiple operations.

**What happens without it:** In production, the rule is simple: every external service call must have a timeout. Without one, a slow database or a hung third-party API holds the event loop's task indefinitely. The task queue grows. Memory consumption climbs. Eventually the service degrades under load in a way that is difficult to diagnose because no individual request fails cleanly — they just stop completing.

**Industry impact:** The critical subtlety is what happens when TimeoutError fires. The underlying coroutine is not simply abandoned — it is cancelled at its next await point. This means the cancelled coroutine's except CancelledError and finally blocks run, which gives it the opportunity to release resources: close sockets, roll back partial writes, log the timeout. Production retry patterns combine wait_for with an exponential backoff loop: attempt the call, catch TimeoutError, sleep briefly, retry — up to a maximum number of attempts. This is the foundation of resilient service clients.

---

## EXAMPLE 3.1 — asyncio.wait_for with timeout and retry

Narration: The unreliableApi coroutine simulates a service with variable response times — anywhere from 0.1 to 2.0 seconds. The callWithTimeout coroutine wraps it with asyncio.wait_for, setting a 0.5-second deadline. If the call returns within 0.5 seconds, we print success and return the result. If it does not, wait_for raises asyncio.TimeoutError, we catch it, print the timeout message, and the loop continues to the next attempt. After all retries are exhausted, we return a failure message. In main, we run three independent callers concurrently with gather. Each has its own retry state; a timeout in caller A does not affect caller B. The output shows a mixture of successes on varying attempt numbers and failures when all retries are consumed — a realistic simulation of a production service with unreliable upstream dependencies.

```python
# Example 24.3
import asyncio                                         # line 1
import random                                          # line 2

async def unreliableApi(apiId):                        # line 3
    delay = random.uniform(0.1, 2.0)                   # line 4
    print(f"API {apiId}: will respond in {delay:.2f}s")  # line 5
    await asyncio.sleep(delay)                         # line 6
    return f"API {apiId} response"                     # line 7

async def callWithTimeout(apiId, timeoutSeconds, maxRetries):  # line 8
    for attempt in range(1, maxRetries + 1):           # line 9
        try:                                           # line 10
            result = await asyncio.wait_for(           # line 11
                unreliableApi(apiId),                  # line 12
                timeout=timeoutSeconds                 # line 13
            )                                          # line 14
            print(f"API {apiId} succeeded on attempt {attempt}")  # line 15
            return result                              # line 16
        except asyncio.TimeoutError:                   # line 17
            print(f"API {apiId} attempt {attempt} timed out")  # line 18
    return f"API {apiId} failed after {maxRetries} attempts"   # line 19

async def main():                                      # line 20
    results = await asyncio.gather(                    # line 21
        callWithTimeout("A", timeoutSeconds=0.5, maxRetries=3),  # line 22
        callWithTimeout("B", timeoutSeconds=0.5, maxRetries=3),  # line 23
        callWithTimeout("C", timeoutSeconds=0.5, maxRetries=3),  # line 24
    )                                                  # line 25
    for result in results:                             # line 26
        print(result)                                  # line 27

asyncio.run(main())                                    # line 28
```

---

## CONCEPT 4.1 — asyncio.Queue: The Async Producer-Consumer Pattern

**Problem it solves:** asyncio.gather requires you to know all your tasks upfront. But many real workloads are dynamic: a data pipeline where the producer generates work items at a rate determined by an external feed; a web scraper where the set of URLs to visit grows as each page is parsed; a message processor where work arrives continuously from a queue service. For these patterns, the producer-consumer architecture is the right model: one or more producers generate work; one or more consumers process it; a queue between them acts as a buffer and a rate-matching mechanism.

**Why invented:** The threading module has queue.Queue for thread-safe producer-consumer coordination. asyncio.Queue is the equivalent for the async world. It is not thread-safe — it is designed exclusively for use within a single event loop — but it provides the same semantics: put() and get() are coroutines that you await; maxsize enforces a backpressure limit so that a fast producer cannot overwhelm a slow consumer.

**What happens without it:** Without a queue, a dynamic workload forces you to either collect all items before launching any tasks (losing the streaming benefit) or manage complex fan-out logic manually with shared state and locks. The queue separates the concerns cleanly: the producer only needs to know how to produce and how to call await queue.put; the consumers only need to know how to await queue.get and process an item.

**Industry impact:** asyncio.Queue is the building block of streaming data pipelines in pure Python: async ingestion from Kafka or Kinesis, WebSocket message processors, async job queues. The maxsize parameter is particularly important in production: it creates natural backpressure. When the consumer is slower than the producer, the queue fills up, and each subsequent await queue.put blocks the producer until space is available — automatically throttling the source rate to match the processing capacity.

---

## EXAMPLE 4.1 — Async producer-consumer with asyncio.Queue

Narration: We create a queue with a maximum size of five — this is the backpressure valve. The producer generates ten work items, each with a random integer value, and awaits queue.put for each one. If the queue is full — because the consumers are slower than the producer — this await blocks the producer, naturally throttling it. After all items are produced, the producer puts a None sentinel to signal shutdown. Each consumer loops forever on await queue.get. When it receives None, it re-puts the sentinel so the other consumer also receives the shutdown signal, then breaks. This re-put sentinel pattern ensures that a single None correctly shuts down all consumers regardless of how many there are. The consumer processes each item by multiplying its value by three and appending the result to a shared list. Because asyncio is single-threaded, the shared results list requires no lock — no two coroutines can modify it simultaneously. After gather returns, we print the total count of processed items.

```python
# Example 24.4
import asyncio                                         # line 1
import time                                            # line 2
import random                                          # line 3

async def dataProducer(queue, numItems):               # line 4
    for itemId in range(numItems):                     # line 5
        item = {"id": itemId, "value": random.randint(1, 100)}  # line 6
        await queue.put(item)                          # line 7
        print(f"Produced: {item}")                     # line 8
        await asyncio.sleep(0.05)                      # line 9
    await queue.put(None)                              # line 10  sentinel

async def dataConsumer(consumerId, queue, results):    # line 11
    while True:                                        # line 12
        item = await queue.get()                       # line 13
        if item is None:                               # line 14
            await queue.put(None)                      # line 15  re-put for other consumers
            break                                      # line 16
        processed = item["value"] * 3                  # line 17
        results.append({"id": item["id"], "result": processed})  # line 18
        print(f"Consumer {consumerId}: processed item {item['id']} → {processed}")  # line 19
        await asyncio.sleep(0.08)                      # line 20

async def main():                                      # line 21
    queue = asyncio.Queue(maxsize=5)                   # line 22
    results = []                                       # line 23
    await asyncio.gather(                              # line 24
        dataProducer(queue, numItems=10),              # line 25
        dataConsumer(1, queue, results),               # line 26
        dataConsumer(2, queue, results),               # line 27
    )                                                  # line 28
    print(f"Total processed: {len(results)}")          # line 29

asyncio.run(main())                                    # line 30
```

---

## CONCEPT 5.1 — asyncio.Event and asyncio.Condition: Coordination Primitives

**Problem it solves:** Sometimes coordination between coroutines is not about limiting access to a resource or passing data through a queue — it is about timing. One coroutine needs to wait until another coroutine signals that some condition is true. A connection pool needs to pause all callers while it is being initialized, then release them all at once when initialization completes. A set of worker coroutines should start processing simultaneously only after a configuration load coroutine has confirmed that all required settings are available.

**Why invented:** asyncio.Event models a one-shot binary signal. It starts in the "unset" state. Any number of coroutines can await event.wait() — they all pause at that line. When one coroutine calls event.set(), every waiting coroutine is immediately scheduled to resume. The event remains set; any future coroutine that calls wait() on an already-set event continues immediately without waiting. asyncio.Condition extends this with a predicate: coroutines wait not for any signal, but for a specific condition to become true — useful when the shared state can transition through multiple values and different waiters care about different thresholds.

**What happens without it:** Without Event, the "start signal" pattern requires polling — a coroutine loops, checks a shared flag, sleeps briefly, checks again. Polling wastes CPU cycles and introduces latency proportional to the polling interval. Event is edge-triggered: the wakeup happens the instant the flag is set, with no polling delay and no wasted work between checks.

**Industry impact:** asyncio.Event is used throughout the async ecosystem: in connection pools to signal "a connection is now available," in distributed lock managers to signal "the lock has been released," in startup sequences to signal "configuration is loaded and all workers can begin." One important distinction: asyncio.Event is not thread-safe. It is designed exclusively for coordination within a single event loop. If you need to signal across OS threads — for example, from a background thread running a file watcher to the event loop — you must use loop.call_soon_threadsafe or asyncio.Queue, not asyncio.Event directly.

---

## CONCEPT 6.1 — Final Takeaway Lecture 24

asyncio.Semaphore limits the number of coroutines that can be in a critical section simultaneously — it is the primary tool for encoding rate limits and connection pool ceilings directly into your async code. asyncio.wait with return_when gives fine-grained task coordination: FIRST_COMPLETED for racing, FIRST_EXCEPTION for fail-fast, ALL_COMPLETED for flexible gathering that lets you inspect individual results and exceptions. asyncio.wait_for adds a deadline to any coroutine — in production, every external service call must have one; hung calls without timeouts are the most common cause of cascading failures in async services. asyncio.Queue implements the async producer-consumer pattern with backpressure: maxsize ensures a fast producer cannot overwhelm slow consumers, and the sentinel pattern provides clean shutdown coordination. asyncio.Event provides a one-shot signal that wakes all waiting coroutines simultaneously — the cleanest solution for startup coordination and availability signals. These five tools are the composition layer of async Python. They are not alternatives to gather and create_task; they are the structures you build on top of them when your system grows beyond a simple fan-out. In Lecture 25 we move to multiprocessing and inter-process communication — the tools you reach for when I/O-optimized async is not enough and you need true CPU parallelism across multiple OS processes.
