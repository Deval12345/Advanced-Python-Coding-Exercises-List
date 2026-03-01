# Lecture 20: The Global Interpreter Lock — Why CPython Is Single-Threaded by Design
## Spoken Instructor Narrative

---

In the last lecture, we talked about threading — ThreadPoolExecutor, race conditions, locks, queues. And I mentioned, almost in passing, something called the GIL. I said it was the reason threads don't help with CPU-heavy work. A few of you asked about it afterward. So today we go deep. Today we pull the curtain back on one of the most misunderstood design decisions in all of Python — the Global Interpreter Lock.

Here is the question I want you to sit with for a moment. If you have four CPU cores, and you spin up four Python threads, how many of those threads are running Python code at any given instant? Your instinct says four. The correct answer is one. Always one. And that single fact changes everything about how you architect Python programs.

---

So what actually is the GIL? The full name is the Global Interpreter Lock — and the name tells you exactly what it is. It is a single mutex, a single lock, that any thread must acquire before it can execute Python bytecode. One thread holds the lock. Every other thread waits. When the first thread finishes its turn, it releases the lock, and one of the waiting threads picks it up.

Why would anyone design a language runtime this way? The answer is memory management. Python's interpreter maintains a tremendous amount of internal state — reference counts on every object, internal data structures for the interpreter itself, object mutation in progress. If two threads could freely run at the same time, they could both reach into Python's internals simultaneously, and the result would be chaos. Memory corruption, crashes, and the worst category of bug in software — behavior that is wrong but doesn't immediately fail.

The designers of CPython faced a choice. They could put a fine-grained lock on every single object, every reference count, every data structure inside the interpreter — thousands of locks. Or they could use one coarse lock around the whole interpreter. The coarse lock won. It was simpler to implement, easier to reason about, and nearly impossible to get wrong. The cost was parallelism. The benefit was correctness and simplicity.

---

Now let's talk about what the GIL is specifically protecting. The core mechanism is reference counting. Every Python object has a counter that tracks how many variables, containers, and function arguments currently point to it. The moment that counter reaches zero, Python immediately frees the object's memory. No garbage collector needed to schedule a cleanup pass — the memory is reclaimed right now, at the instant the last reference disappears.

This is elegant. It gives Python deterministic cleanup — when you close a file object, the file handle is released immediately, not at some later GC pause. It gives you predictable memory behavior. But here is the catch — updating a reference count is a read-modify-write operation. Read the count, add one or subtract one, write it back. That sequence is not atomic. Without protection, two threads could both read the same count, both modify it, and write conflicting results.

Imagine two threads both pointing at the same object. Thread one is about to release its reference — it reads the count, sees one, subtracts one, sees zero, and begins freeing the memory. But before it can finish, thread two reads the count, also sees one, subtracts one, also sees zero, and also tries to free the same memory. You have now freed the same memory block twice. This is a double-free — one of the most dangerous bugs in systems programming. The GIL prevents this from ever happening.

We can actually watch reference counting in action, and I want to show you that before we move on. Python exposes a function in the sys module — getrefcount — that returns the current reference count of any object. We'll create a list, create aliases to it, put it inside a container, then remove those references one by one. Every step, the count changes in a completely predictable way. The one surprise is that getrefcount itself temporarily adds one extra reference as its argument, so every count you see is one higher than you might expect.

---

Now, I need to introduce the most important practical consequence of the GIL, and I want you to feel the intuitive wrongness of it before I explain the reasoning. Ready?

Adding more threads to a CPU-bound task — something like counting primes, summing a billion numbers in a pure Python loop, any tight computation — makes your program either no faster, or actually slower. Four threads, four cores, and you get the performance of one.

Here is why. The GIL means only one thread can run Python bytecode at any moment. Your four threads are not four workers digging in parallel — they are four workers where only one is allowed to hold the shovel at a time, and they spend constant energy passing the shovel back and forth. The operating system still context-switches between them. It still saves and restores thread state. You pay all the overhead of concurrency; you receive none of the benefit. On CPU-bound work, threads with the GIL can actually be measurably slower than a single sequential run.

Let's look at an example that proves this empirically. We'll define a function that does 40 million additions in a pure Python loop — that is deliberately CPU-intensive and never touches I/O. We'll time it running sequentially, then with four threads, then with four processes. The threading result will surprise you if you expect parallelism. The multiprocessing result will be what your intuition predicted.

The fix is exactly what you saw in that benchmark — multiprocessing. When you spawn a separate process, you get a separate Python interpreter with its own GIL. Four processes on four cores means four interpreters, four GILs, all running simultaneously. True parallelism. Or the alternative — offload your computation to a C extension library like NumPy, which releases the GIL during array operations, allowing threads to run NumPy computations in parallel. This is exactly how scientific Python code achieves multi-core performance without multiprocessing overhead.

---

Here is the part that saves threads entirely — and it is critical that you understand this. The GIL releases. Every time a thread makes a system call — asking the operating system to read from a socket, write to a file, wait for a timer — it tells Python "I'm going to be idle; I don't need the interpreter right now." Python releases the GIL. Another thread immediately picks it up and starts executing Python bytecode. When the first thread's I/O completes, it queues up to reacquire the GIL.

Think about what this means for a web scraper fetching ten pages. Thread one asks the network for page one and releases the GIL. Thread two immediately grabs it and asks for page two. Thread three asks for page three. All ten threads have their requests in flight simultaneously, all waiting on the network, and as each network response comes back, each thread picks up the GIL briefly to process the response. The total time approaches the time for the single slowest request — not ten times the average request time.

Any operation that calls the operating system for I/O releases the GIL. Socket reads and writes. File reads and writes. Database queries. HTTP requests. Even time dot sleep — sleeping releases the GIL so other threads can run during the wait. Pure Python computation? Never. That is the line.

This is why threading genuinely works for web scrapers, API clients, database polling, and concurrent download managers. Every blocking operation is an opportunity for other threads to make progress. The GIL is not a wall; it is a revolving door.

---

So now you have the mental model, and let me sharpen it into the exact question you should ask every time you reach for a concurrency tool. The question is not "How do I get around the GIL?" The question is — is this workload I/O-bound, or is it CPU-bound?

I/O-bound work spends most of its time waiting for the operating system. Network calls, disk access, database queries. For I/O-bound work, threads work beautifully — the GIL releases at every system call and other threads fill the waiting time. Async and await works even better — single-threaded cooperative scheduling that scales to hundreds of thousands of concurrent I/O operations. We cover that next lecture.

CPU-bound work spends most of its time executing Python bytecode. Mathematical computation, string processing, in-memory data transformation. For CPU-bound work, threads are useless or harmful due to the GIL. Reach for multiprocessing — separate processes, each with its own interpreter and its own GIL. Or reach for a C extension like NumPy that releases the GIL internally. Or use a task queue like Celery that runs separate worker processes.

When you see a web server handling thousands of connections — I/O-bound — threads and async are the right tools. When you see an ML preprocessing pipeline crunching through gigabytes of feature data — CPU-bound — ProcessPoolExecutor or NumPy is the right tool. Get that classification right and the architecture follows naturally.

---

Let me leave you with the honest truth about the GIL. It is not a flaw that Python developers tolerate despite the language being otherwise excellent. It is a deliberate design commitment — an explicit trade-off of multi-core CPU parallelism in exchange for simplicity, single-threaded performance, and an enormous ecosystem of C extensions that can be written without thinking about thread safety.

PEP 703 in Python 3.13 offers an experimental no-GIL mode that eliminates the lock using biased reference counting and atomic operations. Early results show 10 to 40 percent slower single-threaded performance — you pay for thread-safety even when you have only one thread. And hundreds of thousands of C extension packages must be audited before that mode can be trusted in production. It will come, eventually, but it will take years.

In the meantime, the GIL is the reality — and understanding it makes you a better Python engineer than most. You know when threads help. You know when they don't. You know exactly why. And you know exactly what to reach for instead.

Next time, we dive into async and await — the single-threaded concurrency model that scales I/O to scales that threads can't match, without any of the overhead of context switching. See you then.
