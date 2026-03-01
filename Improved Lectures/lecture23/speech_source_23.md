# Lecture 23: Futures and Executors — The Bridge Between Sync and Async
## speech_source_23.md — Synchronization Master File

---

## CONCEPT 1.1 — What a Future Actually Is

**Problem it solves:**
Before Futures, concurrent programming relied heavily on callbacks. You would pass a function to another function and say: "when this task finishes, call this function with the result." On the surface that sounds clean. But real programs are not a single task — they're dozens, hundreds, thousands of concurrent operations. Callbacks fragment your sequential logic across a web of nested functions. Exception handling becomes a nightmare: you can't use try/except across callback boundaries. Stack traces become meaningless. The pattern is so painful it earned the name "callback hell."

**Why invented:**
The Future (sometimes called a Promise in other languages) was invented to restore sequential reasoning to concurrent code. Instead of saying "here is what to do when this task completes," you say "give me a handle to this task's result." That handle — the Future — travels through your code like any other object. You submit a task, get a Future immediately, do other work, then when you need the result you ask the Future for it.

**What happens without it:**
Without Futures, you manage completion manually: shared queues, boolean flags, event objects, callback registries. Every team builds their own ad-hoc version. Error propagation from worker threads or processes requires explicit checks on shared error containers. Results come back out-of-band, disconnected from the call site.

**Industry impact:**
The concurrent.futures module — introduced in Python 3.2 — unified Python's threading and multiprocessing behind a single, clean abstraction. The same API works whether you're using threads or separate processes. JavaScript adopted Promises (same concept) to escape callback hell. Java has CompletableFuture. Every modern language has converged on this abstraction because it works.

**Key mechanics:**
A concurrent.futures.Future moves through states: PENDING when submitted, RUNNING when a worker picks it up, then DONE — which can mean FINISHED (success), CANCELLED, or storing an EXCEPTION. Key methods: future.result() blocks the calling thread until the task is done and returns the result, or raises the exception if the task failed. future.exception() returns the exception object without raising it. future.done() is a non-blocking check you can poll. future.cancel() asks to cancel — only works if the task hasn't started yet.

**The power:** You decouple when you START a task from when you NEED its result. Submit 100 tasks, continue doing other work, collect results exactly when you need them — in any order, at any time.

---

## CONCEPT 2.1 — ProcessPoolExecutor for CPU-Bound Work

**Problem it solves:**
We covered the GIL in Lecture 20 — the Global Interpreter Lock that prevents multiple threads from executing Python bytecode simultaneously. ThreadPoolExecutor is excellent for I/O-bound work: network requests, file reads, database queries — anything where threads spend most of their time waiting. But for CPU-bound work — number crunching, image processing, data transformation — threads fight over the GIL and you get no real parallelism. You run on one core no matter how many threads you create.

**Why invented:**
ProcessPoolExecutor was designed to give Python true CPU parallelism. Each process gets its own Python interpreter, its own GIL, its own memory space. They run on separate cores simultaneously with no GIL contention. The genius of the design: it uses exactly the same API as ThreadPoolExecutor. You submit tasks with executor.submit(), iterate results with executor.map(), and collect futures the same way.

**What happens without it:**
Before ProcessPoolExecutor, you used the multiprocessing module directly — managing Process objects, Queue objects for communication, Lock objects for synchronization. It was powerful but verbose. ProcessPoolExecutor hides all of that behind the Future abstraction.

**The cost — Inter-Process Communication:**
Processes don't share memory. When you submit a task to ProcessPoolExecutor, Python must serialize the function and its arguments using pickle, send them to the worker process, and deserialize on the other side. The result comes back the same way. This serialization cost — called IPC overhead — is real and matters for task design.

**The rule:** Use ProcessPoolExecutor when each task takes substantial CPU time — roughly 100 milliseconds or more — so the pickle cost is negligible compared to the work done. For many tiny tasks, the IPC overhead dominates and ProcessPoolExecutor can be SLOWER than running sequentially. Profile before assuming parallelism helps.

**Critical gotcha — the if __name__ == "__main__" guard:**
On Windows and macOS, Python spawns new processes by forking or importing the main script. Without the guard, each spawned process imports the script and tries to create its own ProcessPoolExecutor — which tries to spawn more processes — which import the script — infinite spawn, program hangs or crashes. Always wrap your executor code in `if __name__ == "__main__":` when using ProcessPoolExecutor.

---

## EXAMPLE 2.1 — CPU-bound tasks with ProcessPoolExecutor

**Narration:**
This example demonstrates the performance difference between sequential execution and ProcessPoolExecutor for a genuinely CPU-bound task — counting primes up to a limit using trial division. We measure both approaches, compute the speedup, and make the one-line switch visible.

```python
# Example 23.1
import concurrent.futures                              # line 1
import time                                            # line 2
import math                                            # line 3

def computePrimeCount(limit):                          # line 4
    count = 0                                          # line 5
    for n in range(2, limit):                          # line 6
        if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1)):  # line 7
            count += 1                                 # line 8
    return count                                       # line 9

def runSequential(limits):                             # line 10
    return [computePrimeCount(limit) for limit in limits]  # line 11

def runWithProcessPool(limits, maxWorkers):            # line 12
    with concurrent.futures.ProcessPoolExecutor(max_workers=maxWorkers) as executor:  # line 13
        results = list(executor.map(computePrimeCount, limits))  # line 14
    return results                                     # line 15

if __name__ == "__main__":                             # line 16
    workloads = [50000, 50000, 50000, 50000]           # line 17

    start = time.perf_counter()                        # line 18
    seqResults = runSequential(workloads)               # line 19
    seqTime = time.perf_counter() - start              # line 20
    print(f"Sequential: {seqTime:.2f}s → {seqResults}")  # line 21

    start = time.perf_counter()                        # line 22
    poolResults = runWithProcessPool(workloads, maxWorkers=4)  # line 23
    poolTime = time.perf_counter() - start             # line 24
    print(f"4 Processes: {poolTime:.2f}s → {poolResults}")  # line 25
    print(f"Speedup: {seqTime/poolTime:.1f}x")         # line 26
```

---

## CONCEPT 3.1 — Future Callbacks and Error Handling

**Problem it solves:**
Sometimes you want to process a result as soon as it's available, without blocking anywhere. You want to say: "when this task finishes, run this function immediately — I'll continue doing other things." That's what callbacks on Futures give you. But unlike raw callback-based code, here the callback attaches to a clean Future object, not entangled with your submission logic.

**Why invented:**
future.add_done_callback() was added to make the non-blocking reaction pattern composable. You can chain multiple callbacks onto a single Future. You register them before or after the task completes — if already complete, the callback fires immediately. This is the foundation of reactive programming patterns in Python.

**Error propagation — the elegant solution:**
When a worker thread or process raises an exception, that exception is captured and stored inside the Future object. When you call future.result(), the exception is re-raised in the calling thread. This means error handling from concurrent code uses ordinary Python try/except blocks — the same syntax you use for sequential code. Compare this to the pre-Future era: exceptions from threads were lost unless you explicitly caught them in the worker and communicated them via a queue or flag.

**as_completed() vs executor.map():**
executor.map() yields results in submission order — task 0 first, then task 1, then task 2, regardless of which finishes first. If task 0 takes 10 seconds and task 2 takes 0.1 seconds, you wait 10 seconds before getting task 2's result. concurrent.futures.as_completed() takes a collection of futures and yields them as they finish — first come, first served. For programs where you want to process results as fast as possible, as_completed() is the right tool.

**Industry impact:**
This pattern — submit work, attach callbacks, collect via as_completed — is used in production web scrapers, API fan-out clients, and data pipeline systems. It lets you write systems that saturate your network or CPU without ever blocking unnecessarily.

---

## EXAMPLE 3.1 — Futures with callbacks and error handling

**Narration:**
This example shows futures in their full form: submission, callback registration, as_completed iteration, and error handling — all in one cohesive pattern. Tasks randomly fail to simulate real-world unreliability. Notice how the callback fires for both successes and failures, and how as_completed naturally handles the race between them.

```python
# Example 23.2
import concurrent.futures                              # line 1
import time                                            # line 2
import random                                          # line 3

def riskyTask(taskId):                                 # line 4
    time.sleep(random.uniform(0.1, 0.5))               # line 5
    if random.random() < 0.3:                          # line 6
        raise ValueError(f"Task {taskId} failed randomly")  # line 7
    return f"Task {taskId} succeeded"                  # line 8

def onComplete(future):                                # line 9
    if future.exception():                             # line 10
        print(f"CALLBACK: caught exception: {future.exception()}")  # line 11
    else:                                              # line 12
        print(f"CALLBACK: result = {future.result()}")  # line 13

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:  # line 14
    futures = []                                       # line 15
    for taskId in range(8):                            # line 16
        future = executor.submit(riskyTask, taskId)    # line 17
        future.add_done_callback(onComplete)           # line 18
        futures.append(future)                         # line 19

    print("\n--- Collecting results with error handling ---")  # line 20
    successCount = 0                                   # line 21
    failCount = 0                                      # line 22
    for future in concurrent.futures.as_completed(futures):  # line 23
        try:                                           # line 24
            result = future.result()                   # line 25
            successCount += 1                          # line 26
        except ValueError as e:                        # line 27
            failCount += 1                             # line 28
            print(f"Handled error: {e}")               # line 29

    print(f"Done: {successCount} succeeded, {failCount} failed")  # line 30
```

---

## CONCEPT 4.1 — asyncio.Future vs concurrent.futures.Future

**Problem it solves:**
As your system grows, you'll have two worlds: async code running in an event loop handling high-concurrency I/O, and synchronous legacy library calls — PDF generators, old database drivers, image processing libraries — that block. You need a clean bridge between them. The asyncio.Future is that bridge.

**Why invented:**
asyncio.Future was created as the async-native counterpart to concurrent.futures.Future. It is awaitable — you can use the await keyword on it inside a coroutine. It lives inside the event loop and cooperates with the scheduling cycle. While concurrent.futures.Future uses blocking .result() calls, asyncio.Future suspends its coroutine at the await point and returns control to the event loop to run other tasks.

**How they connect:**
loop.run_in_executor() is the key function. It takes a callable, submits it to a thread pool (or process pool), and returns an asyncio.Future that resolves when the thread finishes. You await that future from inside a coroutine. The event loop remains unblocked — it happily handles other coroutines while the thread does its work. asyncio.wrap_future() can manually convert a concurrent.futures.Future into an asyncio.Future if you need finer control.

**The industry hybrid pattern:**
Production systems at scale frequently combine both worlds. The event loop handles thousands of concurrent connections — it's exceptional at that. But when a request needs to call a synchronous library (render a PDF, compress an image, query a legacy ORM that isn't async), you run_in_executor to hand that work to a thread pool. The event loop stays responsive. The thread pool absorbs the blocking calls. The two worlds coexist cleanly through the Future abstraction.

**What happens without it:**
A blocking call inside a coroutine — time.sleep(), a synchronous database query, a heavy computation — blocks the entire event loop. Every other coroutine waits. In production, this manifests as request timeouts, unresponsive websockets, and latency spikes that are notoriously hard to diagnose.

---

## EXAMPLE 4.1 — asyncio.Future and wrapping thread-pool futures

**Narration:**
This example demonstrates the hybrid pattern: a legacy synchronous function that sleeps for 200 milliseconds is called four times concurrently via run_in_executor. Because each call runs in its own thread while the event loop stays free, all four complete in roughly 200 milliseconds total — not 800. This is the fundamental performance argument for the async-plus-thread-pool architecture.

```python
# Example 23.3
import asyncio                                         # line 1
import concurrent.futures                              # line 2
import time                                            # line 3

def legacyBlockingCompute(value):                      # line 4
    time.sleep(0.2)                                    # line 5
    return value * value                               # line 6

async def asyncWrapper(executor, value):               # line 7
    loop = asyncio.get_event_loop()                    # line 8
    result = await loop.run_in_executor(               # line 9
        executor,                                      # line 10
        legacyBlockingCompute,                         # line 11
        value                                          # line 12
    )                                                  # line 13
    return result                                      # line 14

async def main():                                      # line 15
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:  # line 16
        start = time.perf_counter()                    # line 17
        results = await asyncio.gather(                # line 18
            asyncWrapper(executor, 10),                # line 19
            asyncWrapper(executor, 20),                # line 20
            asyncWrapper(executor, 30),                # line 21
            asyncWrapper(executor, 40),                # line 22
        )                                              # line 23
        elapsed = time.perf_counter() - start          # line 24
        print(f"Results: {results}")                   # line 25
        print(f"Time: {elapsed:.2f}s (should be ~0.2s, not 0.8s)")  # line 26

asyncio.run(main())                                    # line 27
```

---

## CONCEPT 5.1 — Final Takeaway: Lecture 23

**The unified picture:**
Futures are not a complexity — they are a simplification. Before Futures, concurrency code in Python required managing queues, flags, locks, and callbacks manually. The Future abstraction gives you a single object that represents an in-flight computation, carries its result or exception when done, and provides a consistent interface whether that computation ran in a thread, a process, or an async event loop.

**The three executor relationships:**
ThreadPoolExecutor creates concurrent.futures.Futures backed by threads — ideal for I/O-bound work, limited by the GIL for CPU work. ProcessPoolExecutor creates concurrent.futures.Futures backed by separate processes — true CPU parallelism, same API, requires pickling and the if __name__ == "__main__" guard. loop.run_in_executor() creates asyncio.Futures backed by a thread pool — the bridge that lets async code call blocking functions without stalling the event loop.

**Error handling is normalized:**
Regardless of whether your computation ran in a thread or a process, exceptions are stored in the Future and re-raised cleanly when you call future.result(). This means your concurrent error handling looks like your sequential error handling — a massive reduction in cognitive complexity.

**Callbacks and completion order:**
add_done_callback() lets you react to completion without blocking. as_completed() lets you process futures in the order they finish, not the order they were submitted — critical for latency-sensitive pipelines where you want to act on the fastest results first.

**What comes next:**
Lecture 24 moves to high-level concurrency coordination: asyncio.gather with return_exceptions, asyncio.wait with FIRST_COMPLETED and FIRST_EXCEPTION policies, asyncio.Semaphore for rate-limiting, and production patterns for controlling concurrency at scale without overwhelming external services.
