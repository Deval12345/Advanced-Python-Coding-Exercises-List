# Speech Source — Lecture 22: The Async Event Loop — How Cooperative Concurrency Works Under the Hood

---

## CONCEPT 0.1 — Transition from Previous Lecture

In Lecture 21 we learned how to write async code: how to define coroutines with async def, how to suspend them at await expressions, how to run multiple coroutines concurrently with asyncio.gather and asyncio.create_task, and how to open resources and iterate results with async with and async for. You now know how to USE the async machinery. Today we open the engine compartment and ask: how does the event loop actually work? What is happening under the hood when a coroutine hits an await? What does "single-threaded concurrency" really mean at the mechanical level? And — critically — what can go catastrophically wrong if you violate the model's one fundamental rule?

---

## CONCEPT 1.1 — The Event Loop: A Single Thread, Many Tasks

**Problem it solves:** How can ONE thread handle thousands of concurrent I/O operations without blocking? Before async I/O, the standard answer was one thread per connection. Every incoming connection got its own OS thread. The thread would block on the network read, wait, get its data, respond, and exit. This works fine at small scale — ten or twenty connections. But a production web server might handle ten thousand simultaneous connections. At one megabyte of stack per thread, that is ten gigabytes of RAM just for thread stacks. Add the OS scheduler thrashing between ten thousand threads during context switches, and the server is spending more time managing overhead than serving requests.

**Why invented:** The event loop model was pioneered in system-level programming (select/poll in C, libevent in C++) and popularized for application developers by Node.js in 2009. Python's asyncio (PEP 3156, formalized in Python 3.4, with native async/await syntax in Python 3.5) imports this architecture into Python. The core insight: most I/O operations spend the vast majority of their time waiting — not computing. A single thread that waits on I/O efficiently can serve thousands of concurrent connections.

**What happens without it:** A thread-per-connection server at 10,000 connections needs 10 GB of stack space. OS context switching between thousands of threads produces measurable latency overhead. Memory pressure causes the OS to swap pages. The server saturates long before its CPU or network bandwidth does.

**Industry impact:** asyncio is the foundation of FastAPI, aiohttp, Starlette, Sanic, and every modern high-performance Python service. Systems with hundreds of thousands of concurrent WebSocket connections are deployed on single machines using this model.

**The restaurant analogy:** Think of the event loop as a single waiter managing an entire restaurant. When a customer (a coroutine) places an order (starts an I/O operation), the waiter takes the order to the kitchen (initiates the I/O call to the OS) and immediately walks to the next table rather than standing idle waiting for the food to arrive. When the kitchen rings a bell (the OS signals the I/O is complete), the waiter returns to that table to deliver the result. One waiter — dozens of tables — no standing around.

**The mechanical model:**
- The event loop maintains two data structures: a "ready queue" of coroutines that can run right now, and an "I/O registry" of coroutines that are suspended, each paired with the file descriptor or timer they are waiting on
- When all ready coroutines have yielded (suspended at an await), the event loop calls the OS's I/O multiplexer — select on Windows, epoll on Linux, kqueue on macOS — with the list of file descriptors it cares about
- This is a single blocking OS call that sleeps until at least one I/O operation completes
- The OS wakes the event loop and says: "Socket A has data; socket C finished writing" — the event loop marks those coroutines as ready and resumes them in the next cycle

---

## CONCEPT 2.1 — The Event Loop's Execution Cycle

**Problem it solves:** How does the event loop decide which coroutine to run next, and how does it know when I/O has completed? Without a precise execution model, programmers cannot reason about ordering, fairness, or correctness.

**Why invented:** The three-step cycle formalizes cooperative multitasking. It makes the behavior deterministic and predictable between any two consecutive await points in the same coroutine.

**What happens without it:** Without a defined cycle, you get unpredictable interleaving, starvation (some coroutines never run), and lost wakeups (I/O completions that are never processed).

**Industry impact:** Understanding the cycle explains dozens of async gotchas that trip up every developer who moves from writing async code to debugging it under load.

**The three-step cycle (simplified):**
1. Run all currently-ready callbacks and coroutines until each one yields (hits an await point)
2. Poll the OS I/O system (epoll/select/kqueue) to discover which pending I/O operations have completed
3. Schedule the newly-ready coroutines (those whose I/O completed) back into the ready queue, then repeat from step 1

**Key properties of the cycle:**
- "Cooperative" means coroutines must voluntarily yield — the event loop NEVER forcibly interrupts a running coroutine. If a coroutine runs a long synchronous computation without an await, it monopolizes the event loop thread and nothing else runs for the entire duration.
- The event loop is single-threaded — this is its strength (no race conditions between coroutines between await points) and its weakness (one blocking call anywhere freezes the entire loop)
- asyncio.get_event_loop() returns the currently running loop from within an async context; asyncio.run() is the correct entry point for programs — it creates a new event loop, runs the given coroutine until completion, then closes the loop and cleans up

**The critical rule:** NEVER call time.sleep() inside a coroutine — use asyncio.sleep() instead. time.sleep() blocks the OS thread; asyncio.sleep() suspends the coroutine and yields to the event loop. One of these lets thousands of other coroutines make progress; the other freezes them all.

---

## EXAMPLE 2.1 — Visualizing the Event Loop Schedule

Narration: This example makes the event loop's execution cycle visible by combining CPU work (a busy-wait loop) with async I/O (asyncio.sleep). Each task prints a timestamp when it starts CPU work, when it yields to I/O, and when it resumes. Because all three tasks are launched with asyncio.gather, they start in rapid succession. But notice the order of events: task A starts its CPU work first and holds the event loop for 0.1 seconds; only when A's busy loop finishes and it hits its await does B get to start. Then B runs its 0.1 second CPU work; then C. All three hit their await asyncio.sleep lines in sequence. From that point they are all sleeping concurrently in the I/O registry, and they wake up in order of their shortest delay — C first (0.2s), then B (0.3s), then A (0.5s). Total elapsed time is roughly 0.3 seconds of CPU work plus 0.5 seconds of I/O overlap — not 2.7 seconds, which is what sequential execution would cost.

```python
# Example 22.1
import asyncio                                         # line 1
import time                                            # line 2

async def task(name, workDuration, ioDelay):           # line 3
    print(f"[{time.perf_counter():.2f}] {name}: starting CPU work")  # line 4
    # Simulate CPU work — no yield, monopolizes loop  # line 5
    end = time.perf_counter() + workDuration           # line 6
    while time.perf_counter() < end:                   # line 7
        pass                                           # line 8
    print(f"[{time.perf_counter():.2f}] {name}: yielding to I/O")  # line 9
    await asyncio.sleep(ioDelay)                       # line 10
    print(f"[{time.perf_counter():.2f}] {name}: I/O complete, resuming")  # line 11
    return f"{name} done"                              # line 12

async def main():                                      # line 13
    start = time.perf_counter()                        # line 14
    results = await asyncio.gather(                    # line 15
        task("A", workDuration=0.1, ioDelay=0.5),     # line 16
        task("B", workDuration=0.1, ioDelay=0.3),     # line 17
        task("C", workDuration=0.1, ioDelay=0.2),     # line 18
    )                                                  # line 19
    total = time.perf_counter() - start                # line 20
    print(f"All done in {total:.2f}s")                 # line 21
    print(results)                                     # line 22

asyncio.run(main())                                    # line 23
```

---

## CONCEPT 3.1 — Blocking Calls Are Catastrophic in Async Code

**Problem it solves:** One blocking call inside a coroutine freezes the ENTIRE event loop. All other coroutines stop making progress — not just the one that called the blocking function. This is not a performance degradation; it is a complete stall of all concurrent work.

**Why invented (the solutions):** The async ecosystem evolved two complementary strategies. First: async-native library equivalents. asyncio.sleep instead of time.sleep; aiohttp instead of requests; aiofiles instead of open. These cooperate with the event loop by suspending the coroutine instead of blocking the thread. Second: loop.run_in_executor, which offloads a blocking call to a separate thread or process pool. The blocking function runs on a worker thread (which can block without affecting the event loop thread), and the coroutine awaits the result as if it were an async operation.

**What happens without it:** The bug is insidious because it does not crash the server. Instead, under load, the server appears to slow down mysteriously — because while one coroutine is blocking, hundreds or thousands of others are frozen. Request latency spikes. Health checks time out. Debugging is difficult because the symptom (slowness) is distant from the cause (a single blocking call in a rarely-executed code path).

**Industry impact:** This is the most common performance bug in production async Python services. A new team member adds a requests.get call (instead of aiohttp) to an otherwise async service, deploys it, and the service works fine in testing but degrades severely under production load. The golden rule: inside an async def, NEVER call anything that blocks the OS thread without explicitly offloading it.

**The four blocking culprits:**
1. time.sleep() — use asyncio.sleep() instead
2. requests.get() / urllib.request.urlopen() — use aiohttp or httpx async client
3. open().read() on slow disks — use aiofiles
4. Any heavy CPU computation without yield points — use run_in_executor with ProcessPoolExecutor, or insert asyncio.sleep(0) checkpoints

---

## EXAMPLE 3.1 — Blocking vs Non-Blocking, and run_in_executor

Narration: This example shows the two cases side by side. The wrongWay coroutine calls time.sleep(1) directly — this blocks the OS thread for one full second, freezing every other coroutine in the process. The rightWay coroutine uses loop.run_in_executor: it hands the blocking function off to a ThreadPoolExecutor, which runs it on a worker thread, and awaits the result. From the event loop's perspective, the coroutine is suspended at the await — ready to be woken when the executor finishes — so other coroutines can run during that second. Running two rightWay coroutines concurrently with asyncio.gather completes in approximately one second because the two executor threads overlap their blocking sleeps. Running two wrongWay coroutines would take two seconds — they serialize.

```python
# Example 22.2
import asyncio                                         # line 1
import time                                            # line 2
import concurrent.futures                              # line 3

def blockingOperation(duration):                       # line 4
    time.sleep(duration)                               # line 5
    return f"Blocking result after {duration}s"        # line 6

async def wrongWay():                                  # line 7
    print("Wrong: calling blocking sleep directly")    # line 8
    time.sleep(1)                                      # line 9  BLOCKS EVENT LOOP
    print("Wrong: done")                               # line 10
    return "wrong result"                              # line 11

async def rightWay(loop, executor):                    # line 12
    print("Right: offloading blocking call to executor")  # line 13
    result = await loop.run_in_executor(               # line 14
        executor,                                      # line 15
        blockingOperation,                             # line 16
        1                                              # line 17
    )                                                  # line 18
    print(f"Right: got result: {result}")              # line 19
    return result                                      # line 20

async def main():                                      # line 21
    loop = asyncio.get_event_loop()                    # line 22
    with concurrent.futures.ThreadPoolExecutor() as executor:  # line 23
        results = await asyncio.gather(                # line 24
            rightWay(loop, executor),                  # line 25
            rightWay(loop, executor),                  # line 26
        )                                              # line 27
    print(f"Results: {results}")                       # line 28

asyncio.run(main())                                    # line 29
```

---

## CONCEPT 4.1 — asyncio.sleep(0): Yielding Control Without Waiting

**Problem it solves:** Some coroutines do legitimate CPU-bound work in a loop — processing a batch of records, computing transformations, validating a large dataset. These loops may run for hundreds of milliseconds without ever hitting an I/O await point. During that entire time, the event loop is frozen and every other coroutine is stalled.

**Why invented:** asyncio.sleep(0) is a cooperative yield point that costs essentially zero real time. It suspends the current coroutine for zero seconds — which means: return to the event loop, let other ready coroutines run for one cycle, then immediately resume this coroutine. It is the explicit "I am being cooperative" signal in a CPU-heavy loop.

**What happens without it:** A coroutine that processes one million items without a single await will hold the event loop for its entire duration. A background heartbeat task that should fire every 100ms will not fire at all during that time. Health checks fail. Timeouts trigger. The server appears hung.

**Industry use:** Batch processors, data transformation pipelines, validation loops, any loop that might run for more than a few milliseconds. The pattern is to insert "await asyncio.sleep(0)" every N iterations — where N is chosen so that each batch takes at most a few milliseconds — giving the event loop regular opportunities to handle I/O and run other tasks.

**Industry impact:** This pattern is used in high-throughput data pipeline services where the same process handles both batch computation and real-time I/O. Without periodic yields, the batch work would starve the real-time work.

---

## EXAMPLE 4.1 — Long Loop with Periodic Yield

Narration: We have two concurrent tasks: a long batch processor that multiplies five thousand numbers by two, and a heartbeat that prints a timestamp every 0.1 seconds to simulate a background health monitor. Without the asyncio.sleep(0) call inside the processor's loop, the heartbeat would be completely frozen during processing — you would see large gaps or no heartbeat output at all during the batch run. With the yield every one thousand items, the processor gives the event loop a chance to run the heartbeat between batches. The heartbeat prints regularly, the batch completes, and then the heartbeat task is cancelled cleanly. This demonstrates cooperative multitasking in its purest form: the programmer manually inserts cooperation points.

```python
# Example 22.3
import asyncio                                         # line 1
import time                                            # line 2

async def longBatchProcessor(items, batchSize=1000):  # line 3
    results = []                                       # line 4
    for index, item in enumerate(items):               # line 5
        results.append(item * 2)                       # line 6
        if index % batchSize == 0:                     # line 7
            await asyncio.sleep(0)                     # line 8  yield to event loop
            print(f"Processed {index}/{len(items)}")   # line 9
    return results                                     # line 10

async def heartbeat():                                 # line 11
    while True:                                        # line 12
        print(f"[heartbeat] {time.perf_counter():.1f}")  # line 13
        await asyncio.sleep(0.1)                       # line 14

async def main():                                      # line 15
    items = list(range(5000))                          # line 16
    heartbeatTask = asyncio.create_task(heartbeat())   # line 17
    results = await longBatchProcessor(items)          # line 18
    heartbeatTask.cancel()                             # line 19
    print(f"Processed {len(results)} items")           # line 20

asyncio.run(main())                                    # line 21
```

---

## CONCEPT 5.1 — Final Takeaway for Lecture 22

The event loop is a single-threaded scheduler running a tight three-step cycle: run all ready coroutines until they yield, poll the OS for completed I/O, schedule newly-ready coroutines back into the ready queue. Coroutines yield control voluntarily at every await expression — the event loop sees the yield and switches to the next ready task. A blocking call inside a coroutine freezes the entire event loop for its entire duration — use async library equivalents or loop.run_in_executor to offload blocking work to threads. asyncio.sleep(0) yields control for one event loop cycle without actually sleeping — it is the essential tool for keeping CPU-heavy loops cooperative. asyncio.run() is the correct and only recommended entry point for async programs: it creates a fresh event loop, runs the top-level coroutine to completion, then closes the loop and releases all resources. The model's power comes from its simplicity: single-threaded execution means no race conditions for shared state between any two await points in the same coroutine, and scaling means adding more I/O operations to one loop rather than adding more threads. Next lecture: Futures and Executors — the bridge between async code and the synchronous library world.
