# Lecture 18 — Concurrency Models Overview: I/O-Bound vs CPU-Bound Work
## Speech Source File (Concept/Example Synchronization)

---

## CONCEPT 0.1 — Transition: From Object Model to Concurrency

For the past several lectures, we have been living deep inside Python's object model. We studied descriptors and how attribute access is resolved. We studied abstract base classes and how Python enforces interface contracts. We studied memory layout — slots, `__dict__`, and how Python allocates space for object data. All of that work gave us a precise understanding of how Python builds and manages individual objects.

Now we turn the page. We are going to ask a completely different kind of question. Not "how does Python build a single object?" but "how does Python handle multiple things at once?" This is concurrency — the art of making a program do more than one thing, or at least appear to.

But here is the critical thing about concurrency: before you choose a tool, before you write a single line of concurrent code, you must answer one foundational question. What is your program actually waiting for? Because the answer to that question determines everything. It determines which of Python's three concurrency models you should use. It determines whether you will get a 5x speedup or zero speedup. It determines whether your production system uses 2GB of RAM or 20GB.

Concurrency is not a single switch you flip. It is a diagnosis, followed by a prescription. Get the diagnosis wrong, and the prescription does nothing — or makes things worse.

---

## CONCEPT 1.1 — The Fundamental Question: I/O-Bound vs CPU-Bound

Every slow program is slow because it is waiting for something. The question is: what is it waiting for?

There are two categories, and they are mutually exclusive.

The first category is I/O-bound. Your program is waiting for something outside the CPU — a network response, a disk read, a database query, an API call. During this wait, the CPU is essentially idle. It has finished its computation. It is sitting, doing nothing, waiting for bytes to arrive. If you have five API calls that each take one second, and you run them sequentially, your program takes five seconds. But the CPU was only doing meaningful work for a tiny fraction of that time. The rest was waiting. This is waste, and concurrency is the tool to eliminate that waste.

The second category is CPU-bound. Your program is performing computation — crunching numbers, transforming data, running machine learning inference, parsing large files, doing encryption. The CPU is the bottleneck. It is running at full utilization. There is no idle waiting. The program is slow because the computation itself takes time.

These two categories require completely different solutions. Applying the wrong solution is not just ineffective — it is one of the most expensive engineering mistakes a team can make. A senior engineer's first instinct upon seeing a slow program is not to reach for a concurrency tool. It is to ask: I/O-bound or CPU-bound? The diagnosis precedes the treatment.

The reason this distinction matters so deeply in Python — more than in almost any other language — is the Global Interpreter Lock. The GIL is a mutex in CPython that allows only one thread to execute Python bytecode at a time. For I/O-bound work, the GIL is released during the wait, so multiple threads can overlap their waiting periods and produce a real speedup. For CPU-bound work, the GIL prevents multiple threads from running Python code simultaneously, so adding threads produces no speedup at all. You can add ten threads to a CPU-bound program and run it on a ten-core machine, and it will take exactly as long as running it on a single thread. The GIL makes this diagnosis not just a best practice, but a hard technical requirement.

---

## CONCEPT 1.2 — Measuring to Know: Profiling Before Deciding

Do not guess whether your program is I/O-bound or CPU-bound. Measure it.

Python provides several tools for this. The `cProfile` module is the standard profiler — it tells you which functions are consuming the most time, and how many times each was called. If the top entries in your profile are I/O functions — socket reads, file reads, HTTP calls — you are I/O-bound. If the top entries are tight computation loops with no I/O, you are CPU-bound. The `time.perf_counter` function gives you high-resolution wall-clock timing for specific sections of code, which is useful for quick before-and-after comparisons.

Beyond Python's built-in tools, system-level tools like `top`, `htop`, and `strace` on Linux tell you whether your process is spending time in user space (CPU) or in kernel wait states (I/O). A process pegged at 100% CPU is CPU-bound. A process showing low CPU with frequent system call waits is I/O-bound.

Industry teams waste weeks optimizing the wrong bottleneck. A team might spend two weeks tuning thread counts, adjusting connection pools, and profiling thread overhead — only to discover that their program was CPU-bound the entire time, and threads were never going to help due to the GIL. The cost of that mistake is not just two weeks of engineering time. It is also the opportunity cost of not having shipped the correct solution. A senior engineer's first action upon inheriting a slow system is always to profile. Not to guess. Not to assume. To measure.

---

## EXAMPLE 1.1 — Sequential I/O-Bound Demo

To make the I/O-bound problem concrete, let's look at a simple simulation. We have five API calls. Each one takes one second to complete. We run them one after another. The result is that our program takes five seconds — and the CPU is idle for essentially all of that time. This is the baseline problem.

```python
# Example 18.1 — Sequential I/O simulation
import time                                          # 1: standard library time module

def simulateFetch(taskId):                           # 2: simulates one API call
    time.sleep(1)                                    # 3: sleep(1) represents waiting for I/O
    return f"result_{taskId}"                        # 4: return a dummy result

def runSequential(count):                            # 5: runs all fetches in sequence
    start = time.perf_counter()                      # 6: high-resolution start timestamp
    results = [simulateFetch(i) for i in range(count)]  # 7: list comp runs fetches one at a time
    elapsed = time.perf_counter() - start            # 8: compute total wall time
    print(f"Sequential: {len(results)} tasks in {elapsed:.2f}s")  # 9: report results
    return results                                   # 10: return results list

runSequential(5)                                     # 11: expect ~5.0s
```

The output will show approximately 5.00 seconds. There is nothing wrong with this code in isolation. It is correct. It is just slow in a way that is entirely avoidable — because the CPU has nothing to do while the sleep is running. This is exactly the kind of bottleneck that threading and async are designed to address.

---

## CONCEPT 2.1 — The Three Python Concurrency Models

Python offers three distinct concurrency models, each designed to solve a specific kind of problem.

The first is threads, provided by the `threading` module. Multiple threads share the same process and the same memory space. This means they can read and write the same variables, which is both convenient and dangerous — we will cover thread safety in detail in the next lecture. Because threads share memory, they have very low startup overhead. The GIL prevents true parallel CPU execution, but for I/O-bound work, the GIL is released during system calls and wait states, allowing multiple threads to overlap their waiting periods. This is the key insight: threads do not help CPU-bound Python code, but they dramatically help I/O-bound code.

The second model is async/await, provided by the `asyncio` module. This is single-threaded cooperative concurrency. There is one event loop, and multiple coroutines take turns running. A coroutine runs until it encounters an I/O operation, at which point it explicitly yields control back to the event loop, which can then run another coroutine. When the I/O completes, the original coroutine is resumed. This model has extremely low overhead — no thread switching, no locking — and scales to thousands or tens of thousands of concurrent connections on a single thread. It is the model behind high-performance Python web servers like FastAPI and aiohttp.

The third model is multiprocessing, provided by the `multiprocessing` module. Each worker is a separate operating system process with its own Python interpreter and its own GIL. This means true parallel execution of Python bytecode — multiple cores running CPU-bound Python code simultaneously. The cost is inter-process communication overhead, higher memory usage, and longer startup time. But for genuinely CPU-bound work where you need true parallelism, this is the correct tool.

---

## CONCEPT 2.2 — The Decision Framework

The decision between the three models follows a clear framework.

If your work is I/O-bound and you have a moderate number of concurrent operations — say, dozens to a few hundred — use threads. The threading model is straightforward, familiar to most programmers, and handles this case cleanly.

If your work is I/O-bound but you need very high concurrency — thousands of simultaneous network connections, a high-throughput web service handling many requests at once — use async. The overhead of creating thousands of threads is significant in memory and context-switching cost. Async handles this case at a fraction of the resource cost.

If your work is CPU-bound and you need true parallelism across multiple cores — data processing, numerical computation, ML inference, image processing — use multiprocessing. It is the only Python model that bypasses the GIL and achieves genuine concurrent CPU execution.

Every other combination is either wrong or suboptimal. Threads on CPU-bound work gains you nothing due to the GIL. Multiprocessing on I/O-bound work adds process overhead for no benefit. Async on CPU-bound work is actively harmful — a coroutine that never yields blocks the entire event loop. Understanding this decision framework is more valuable than knowing the API of any individual module.

---

## EXAMPLE 2.1 — Three Models Side by Side: Sequential vs Threaded for I/O

Now we will directly compare sequential and threaded execution for an I/O-bound workload. The difference in wall time makes the problem — and the solution — immediately visible.

```python
# Example 18.2 — Threaded I/O simulation
import time                                          # 1: standard library time module
import threading                                     # 2: standard library threading module

results = []                                         # 3: shared list for all thread results
lock = threading.Lock()                              # 4: mutex to protect the shared list

def fetchWithThread(taskId):                         # 5: function each thread will execute
    time.sleep(1)                                    # 6: simulate 1-second I/O wait
    with lock:                                       # 7: acquire lock before writing to results
        results.append(f"result_{taskId}")           # 8: safely append to shared list

def runThreaded(count):                              # 9: launch count threads concurrently
    global results                                   # 10: reference the module-level list
    results = []                                     # 11: reset for a fresh run
    start = time.perf_counter()                      # 12: high-resolution start timestamp
    threads = [threading.Thread(target=fetchWithThread, args=(i,)) for i in range(count)]  # 13: create thread objects
    for t in threads:                                # 14: start all threads
        t.start()
    for t in threads:                                # 15: wait for all threads to finish
        t.join()
    elapsed = time.perf_counter() - start            # 16: compute total wall time
    print(f"Threaded: {len(results)} tasks in {elapsed:.2f}s")  # 17: report results
    return results                                   # 18: return collected results

runThreaded(5)                                       # 19: expect ~1.0s (5x speedup)
```

The sequential version takes approximately 5 seconds. The threaded version takes approximately 1 second. All five threads start simultaneously, each sleeps for one second, and they all finish around the same time. The GIL is released during `time.sleep` because sleep is a system call, not Python bytecode execution. This is the I/O-bound speedup that threading is designed to deliver.

---

## CONCEPT 3.1 — Why Python Has All Three (Historical and Design Context)

Python did not arrive at three concurrency models by committee design. Each model was added to solve a real problem that emerged at a specific point in computing history.

Python began as a single-threaded language. The GIL was introduced in CPython early in its history to make memory management thread-safe without the complexity of fine-grained locking across the entire runtime. The GIL is a single lock around the interpreter — simple, effective at preventing data corruption, but limiting.

Threads were added to allow Python programs to do I/O concurrently — to handle multiple network connections or file reads without blocking. Because the GIL is released during I/O system calls, threads achieved real concurrency for I/O-bound workloads. But CPU-bound parallelism remained unavailable.

The `multiprocessing` module was added to address CPU-bound parallelism. By spawning separate processes, each with its own interpreter and GIL, true parallel execution became possible. The tradeoff is the cost of inter-process communication and the memory overhead of separate processes. But for workloads that genuinely need multiple cores, this was the breakthrough.

The rise of internet services in the 2000s and 2010s created a new problem: servers needed to handle tens of thousands of simultaneous network connections. Creating one thread per connection does not scale — each thread consumes memory, and context switching thousands of threads has measurable overhead. The async model, eventually formalized with the `asyncio` module and the `async`/`await` syntax in Python 3.5, solved this by providing single-threaded concurrency with an event loop, inspired by Node.js and the reactor pattern. The result was a model that could handle enormous I/O concurrency at minimal resource cost.

Understanding this history means understanding why each model exists — and why none of them is simply "the right answer." They each solve a different problem.

---

## CONCEPT 4.1 — The Cost of Getting It Wrong

Let's be concrete about what happens when teams get the diagnosis wrong.

A data pipeline team is building an ML inference service. Inference is CPU-bound — the model performs heavy numerical computation for each request. The team decides to use threading to parallelize across multiple requests. They launch ten threads. They observe no speedup. They spend two weeks tuning thread counts, adjusting batch sizes, profiling thread overhead. Eventually, a senior engineer recognizes the pattern: the GIL is preventing any parallel CPU execution. The fix requires migrating to multiprocessing — which itself takes another week. The total cost: three weeks of engineering time, and a service that launched late because the diagnosis was wrong at the start.

A different team is building a high-concurrency web API. Each request makes several downstream HTTP calls to external services. This is I/O-bound. The team decides to use multiprocessing to handle concurrent requests, launching one worker process per CPU core — say, 16 workers. Each worker process loads the application, establishes its own connection pools, and occupies approximately 200MB of RAM. At 100 workers for higher load, the service consumes 20GB of RAM. A colleague rebuilds the same service using async — a single process with an asyncio event loop handles thousands of concurrent connections with 2GB of RAM and lower latency. The cost: months of over-provisioned infrastructure and a migration project.

These are not hypothetical. They are the kinds of stories that circulate in engineering retrospectives across the industry. The cost of the wrong diagnosis is not measured in lines of code — it is measured in weeks of engineering time, months of infrastructure cost, and the opportunity cost of the correct solution not being shipped sooner.

---

## CONCEPT 5.1 — Final Takeaway and What Comes Next

The first question you ask about any concurrency problem is not "which module should I use?" It is: "is this work I/O-bound or CPU-bound?"

If it is I/O-bound with moderate concurrency — threads. If it is I/O-bound with high concurrency — async. If it is CPU-bound with a need for true parallelism — multiprocessing. Measure before you decide. Use `cProfile`, use `time.perf_counter`, observe your CPU and I/O utilization, and let the data tell you what kind of bottleneck you have.

In the next lecture, we go deep on threads — the `threading` module, the GIL in detail, thread safety, locks, and the patterns that make threaded I/O-bound code correct and fast. Today's lecture gave you the map. The next lectures will walk you through each path on that map, one at a time.

This is the concurrency chapter. It is one of the most practically important chapters in this course — because every production Python system eventually faces these questions. The engineers who answer them correctly, and answer them quickly, are the engineers that systems are built around.
