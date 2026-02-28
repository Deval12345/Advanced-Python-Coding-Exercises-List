# Speech — Lecture 19: Threading for I/O-Bound Work — Thread Pools, Safety, and Practical Patterns

---

Welcome back. In our last session we drew a line between two kinds of work: I/O-bound work, where your program spends most of its time waiting for external responses — network, disk, database — and CPU-bound work, where your program is actively computing the whole time. We also introduced three concurrency models Python offers: threading, multiprocessing, and async/await. Today we go deep on the first one: threading. By the end of this lecture you will understand why threading is the right fit for I/O-bound tasks, how to use Python's ThreadPoolExecutor correctly, what can go wrong with shared state, and how to fix those problems using locks and queues.

---

Let's start with the fundamental question: why does threading help for I/O-bound work at all?

Imagine you are building a web scraper. You have 100 URLs to fetch and each one takes about one second to respond. If you fetch them one at a time, sequentially, you are looking at 100 seconds of runtime. But here's the thing — during almost all of that 100 seconds, your program is not computing anything. It is blocked, waiting for a server on the other side of the internet to reply. Your CPU is completely idle.

Threading solves this. A thread that is waiting for a network response does something very important in Python: it releases the GIL. The GIL — the Global Interpreter Lock — is Python's mechanism for making sure only one thread runs Python bytecode at a time. But when a thread is blocked waiting for I/O, it voluntarily releases that lock. This means another thread can immediately step in and start its own work. With 10 threads, all 10 can be simultaneously waiting for server responses. Your 100-second scraper becomes a 10-second scraper, not because you are computing faster, but because you are stacking the waits on top of each other instead of putting them end to end.

Now, threads themselves have a cost. The OS creates a real thread — what your operating system's scheduler manages — and that takes somewhere between 50 and 100 microseconds. If you're creating a new thread for every URL you fetch, and you have thousands of URLs, that overhead adds up and you also end up with thousands of simultaneously active threads, which wastes memory. Each thread has a stack of about one megabyte. A thousand threads is a gigabyte of stack memory before you've done any real work.

The solution is a thread pool.

---

Python's `concurrent.futures.ThreadPoolExecutor` manages a fixed pool of reusable worker threads. You decide up front how many workers you want — say, 10 — and those 10 threads are created once and reused for every task you submit. When a task finishes, the worker thread picks up the next one from the queue rather than being destroyed and recreated.

You submit work using `executor.submit()`, which takes a callable and its arguments, adds the task to an internal queue, and immediately returns a `Future` object. A Future is a handle to a result that does not exist yet. When the worker thread finishes the task, it stores the result in the Future. You can call `future.result()` to block until the result is ready, or you can use `concurrent.futures.as_completed()` to iterate over futures as they finish — whichever comes first.

For I/O-bound work, how many workers should you use? More than you might think. A common formula is `min(32, os.cpu_count() + 4)` — because threads are mostly waiting, not computing, you can have many more threads than CPU cores without causing harm. The limit you're looking for is the point where adding more threads stops improving throughput. That's usually constrained by the external resource — the rate limit on the API you're calling, the connection pool limit on your database, or your network bandwidth.

One critical point: the Executor used as a context manager with the `with` statement guarantees that all submitted tasks complete before the `with` block exits. You won't accidentally move on while work is still in progress.

Let's look at a concrete example. We have five URLs to fetch from httpbin.org, a public HTTP testing service. We define a `fetchPage` function that opens the URL, reads the response, measures how long it took, and returns a tuple of the URL, content size, and elapsed time. We create a ThreadPoolExecutor with 5 workers. We submit all five URLs at once using a dictionary comprehension that maps each Future to its URL. Then we use `as_completed` to process results as they arrive. The whole operation finishes in roughly the time of the slowest single request, not the sum of all requests.

---

Now for the part that trips up almost everyone who starts working with threads: shared state.

Threads share memory. There is no copying, no serialization — thread A and thread B both have direct access to the same Python objects. This is efficient, but it is also dangerous. Let me show you exactly why.

Consider a counter. Ten threads all want to increment the same counter. You might expect the final value to be 10. But the `+=` operation is not atomic. Under the hood, Python compiles `self.count += 1` into multiple bytecode instructions: load the current value of `count`, add 1 to it, store the new value back. The OS scheduler can interrupt a thread between any two of those instructions. If thread A loads the value — let's say it's 5 — then gets preempted, and thread B also loads 5, then thread B stores 6, then thread A stores 6 — you've lost one increment. Both threads incremented but the count only went up by one.

This is called a race condition. Race conditions are non-deterministic. Your program might produce the right answer 99 times out of 100, then fail in production under load. They are notoriously difficult to reproduce in testing because the exact interleaving of threads varies from run to run.

The standard fix is `threading.Lock`. A Lock is an object with exactly two states: locked and unlocked. When a thread calls `lock.acquire()`, it either immediately holds the lock if it was free, or it blocks and waits until the thread that holds the lock calls `lock.release()`. Only one thread can hold the lock at a time. Using the lock as a context manager — `with self._lock:` — ensures it is released even if an exception occurs inside the block.

The principle to follow is: any time multiple threads read and write the same mutable variable, that variable must be protected by a lock. Keep the critical section — the code between acquire and release — as small as possible. Doing heavy computation inside a lock forces other threads to wait unnecessarily.

---

For communication between threads, there is an even better tool: `queue.Queue`.

The producer-consumer pattern is one of the most fundamental patterns in concurrent programming. One or more threads generate work — they are producers. One or more threads process that work — they are consumers. The queue sits between them as a buffer. Producers put items on the queue. Consumers take items off and process them.

`queue.Queue` has internal locking built in. Every `put()` and `get()` is thread-safe without you writing a single `acquire()` or `release()`. It also handles backpressure: if you create the Queue with a `maxsize`, any call to `put()` will block when the queue is full, preventing the producer from generating work faster than the consumers can handle it.

The sentinel pattern is the clean way to signal shutdown. The producer, when it is done generating work, puts a special sentinel value — `None` is the standard choice — onto the queue. Each consumer, when it dequeues `None`, re-puts `None` so the next consumer sees it too, then breaks out of its loop. This way you don't need any external flag or event — the sentinel propagates automatically.

Our example shows this: a single producer generating five tasks, three consumers processing them concurrently, results collected in a shared list protected by its own Lock, and the sentinel-based shutdown coordination.

---

Let me leave you with the key principles for today.

Threading is appropriate for I/O-bound work — network calls, file I/O, database queries — because threads release the GIL while waiting. ThreadPoolExecutor is the right abstraction: it manages thread lifecycle, handles exceptions, and cleans up automatically. Never share mutable state between threads without synchronization. Use `threading.Lock` to protect individual shared variables. Use `queue.Queue` for communication between threads — it encapsulates correct locking and backpressure in a clean, tested API.

In the next lecture we're going to look at the GIL itself in depth: what it is, why CPython needs it, and what its actual impact is on different kinds of work. Understanding the GIL properly will help you make good architectural decisions — knowing when threads are the right choice and when you need something else.

---
