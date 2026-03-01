# Lecture 18 — Concurrency Models Overview: I/O-Bound vs CPU-Bound Work
## Delivery Speech

---

For the past several lectures, we have been deep inside Python's object model. We studied descriptors — how Python resolves attribute access through a precise lookup chain. We studied abstract base classes — how Python enforces interface contracts at a structural level. We studied memory layout — slots, dictionaries, and how Python decides where to store your object's data. That work gave us a precise, internal view of how Python builds and manages things.

Now we turn the page entirely.

We are going to ask a completely different kind of question. Not "how does Python build a single object?" but "how does Python handle multiple things happening at once?" This is concurrency, and it is one of the most practically important topics in this entire course. Every production Python system — every web server, every data pipeline, every background worker — eventually faces concurrency questions. The engineers who answer those questions correctly, and answer them quickly, are the engineers that systems are built around.

But here is the thing about concurrency that most courses skip past too quickly. Before you write a single line of concurrent code, before you reach for any tool, you must answer one foundational question. What is your program actually waiting for?

Because the answer to that question determines everything.

---

Programs are slow because they wait. That is not a metaphor — it is literally true. And there are two fundamentally different kinds of waiting.

The first kind is called I/O-bound. Your program is waiting for something outside the CPU — a network response, a database query, a disk read, an API call from an external service. During this wait, your CPU is idle. It has finished its computation. It is sitting there, doing nothing, staring at the wall, waiting for bytes to arrive. If you have five API calls that each take one second and you run them one after another, your program takes five seconds — but the CPU was only doing meaningful work for a tiny fraction of that time. The rest was pure waiting. That is waste. And concurrency is the tool that eliminates that waste.

The second kind is called CPU-bound. Your program is crunching numbers — running machine learning inference, transforming large datasets, doing encryption, parsing huge files, running tight computation loops. The CPU is the bottleneck. It is running at full power. There is no idle waiting. The program is slow because the computation itself takes time, and you need more computation per second than one CPU core can deliver.

Now here is where it gets critical. These two categories require completely different solutions. Applying the wrong solution is not just ineffective — it is one of the most expensive engineering mistakes a team can make. A senior engineer's first instinct upon seeing a slow program is not to immediately grab a concurrency tool. It is to ask: I/O-bound or CPU-bound? The diagnosis comes before the prescription. Every time.

---

Let me give you a concrete picture of the I/O-bound problem.

Imagine five API calls, each taking one second to complete. You run them sequentially, one after another. Five seconds total. The CPU is idle for essentially all of that time — it launched the network request and then went to sleep, waiting. This is the baseline problem that I/O concurrency is designed to solve.

Now here is the powerful insight: those five waiting periods can overlap. If you start all five requests at the same time, they all wait in parallel. After roughly one second, all five are done. Five tasks, one second. That is the speedup that the right concurrency tool delivers for I/O-bound work.

But — and this is the crucial but — that same trick does nothing for CPU-bound work. If each task requires one second of intense computation, running five of them "at the same time" still requires five seconds of computation. You cannot make addition faster by doing it "concurrently." You need more CPU power, not more overlap. The nature of the wait determines the nature of the solution.

---

Python gives you three distinct concurrency models, and each one was built to solve a specific problem.

The first is threads, from the threading module. Multiple threads share the same process and the same memory. They can overlap their waiting periods. For I/O-bound work, this is extremely effective — the moment one thread starts waiting for I/O, another thread can run. The speedup is real and significant. Five threads doing five one-second I/O waits finish in one second instead of five.

But here is the catch, and it is a Python-specific catch: the Global Interpreter Lock — the GIL. The GIL is a mutex inside CPython that allows only one thread to execute Python bytecode at a time. This means that for CPU-bound work, adding threads in Python produces essentially no speedup. The threads take turns, one at a time, running through the interpreter. Ten threads on a ten-core machine still execute like a single thread for pure Python computation. The GIL is released during I/O system calls — which is why threads work for I/O — but it is never released during CPU computation.

The second model is async, from the asyncio module. This is single-threaded cooperative concurrency. One event loop, multiple tasks, and each task explicitly yields control when it is waiting for I/O. The overhead is minimal — no thread switching, no locking overhead — and this model scales to thousands of concurrent connections on a single thread. When you see Python web servers handling enormous traffic on modest hardware, async is usually the reason. FastAPI, aiohttp, Tornado — all built on this model.

The third model is multiprocessing. Each worker is a separate operating system process with its own Python interpreter and its own GIL. True parallel execution of Python code — multiple cores genuinely running simultaneously. The cost is real: process startup time is longer, inter-process communication has overhead, and each process consumes its own memory. But for genuinely CPU-bound work where you need multiple cores, this is the only Python model that actually works.

---

The decision framework is clean once you have the diagnosis.

I/O-bound with moderate concurrency — say, dozens to a few hundred simultaneous operations? Use threads. Straightforward, familiar, effective.

I/O-bound but you need very high concurrency — thousands of simultaneous network connections, a high-throughput API server? Use async. Threads would consume too much memory and create too much context-switching overhead. Async handles this at a fraction of the resource cost.

CPU-bound and you need genuine parallel computation across multiple cores? Use multiprocessing. It is the only path to bypassing the GIL and running Python CPU code in true parallel.

Every other combination is either wrong or suboptimal. Threads on CPU-bound work do nothing — the GIL blocks you. Multiprocessing on I/O-bound work adds process overhead for no benefit. Async on CPU-bound work is actively harmful — a coroutine that never yields will block the entire event loop and starve every other task. The framework is not complicated, but you have to use it correctly.

---

Python did not arrive at three concurrency models by design committee. Each one was added to solve a real problem that emerged at a specific point in computing history.

Python started single-threaded. The GIL was introduced early to make CPython's memory management thread-safe without the complexity of fine-grained locking across the entire runtime. Simple, effective, and — as it turned out — limiting for CPU parallelism.

Threads were added to allow I/O concurrency. Because the GIL releases during I/O system calls, threads achieved real concurrency for network and disk work. CPU parallelism was still unavailable.

Multiprocessing was added to address CPU-bound parallelism. Separate processes, separate interpreters, separate GILs — true parallel execution, with the tradeoffs that come with it.

Then the internet happened at scale. Web services needed to handle tens of thousands of simultaneous connections. One thread per connection does not scale — the memory and context-switching cost is too high. The async model, formalized with the asyncio module and async-await syntax in Python 3.5, solved this with single-threaded cooperative concurrency — inspired by Node.js and the reactor pattern. A single process, one event loop, thousands of tasks.

Understanding this history is not just interesting — it tells you why each model has its specific tradeoffs. They were each built for a specific problem, at a specific moment in time. None of them is universally correct.

---

Let me make the cost of getting this wrong concrete, because it is not abstract.

A data pipeline team builds an ML inference service. Inference is CPU-bound. They decide to parallelize with threads. They launch ten threads. They observe no speedup. They spend two weeks tuning thread counts, profiling thread overhead, adjusting batch sizes. Eventually, a senior engineer recognizes the pattern — the GIL is blocking all parallel CPU execution. The migration to multiprocessing takes another week. Three weeks of engineering time, and a delayed launch, because the diagnosis was wrong at the very beginning.

A different team builds a high-concurrency web API. Each request makes several downstream HTTP calls — I/O-bound. They use multiprocessing: one worker process per CPU core, sixteen workers. Each process loads the application, establishes connection pools, and consumes around two hundred megabytes of RAM. At one hundred workers, the service uses twenty gigabytes of RAM. A colleague rebuilds the same service with async — a single process handling thousands of concurrent connections with two gigabytes of RAM and lower latency. Months of over-provisioned infrastructure. A migration project. All because the diagnosis was wrong.

These are not hypothetical scenarios. They are the stories that circulate in engineering retrospectives across the industry. The cost is measured in weeks of engineering time, months of infrastructure spend, and the opportunity cost of the correct solution not being shipped sooner.

---

So here is the summary — sharp and simple.

The first question you ask about any concurrency problem is not "which module should I import?" It is: I/O-bound or CPU-bound?

If it is I/O-bound with moderate concurrency — threads. If it is I/O-bound with high concurrency — async. If it is CPU-bound and you need multiple cores — multiprocessing.

And before you decide anything, measure. Use cProfile to find where your time is being spent. Use time dot perf underscore counter for section-level timing. Watch your CPU utilization. Let the data tell you what kind of bottleneck you have. Do not guess.

In the next lecture, we go deep on threads — the threading module, the GIL in detail, thread safety, locks, and the patterns that make threaded I/O-bound code both correct and fast. Today we drew the map. Next lecture, we start walking the first path.

This is the concurrency chapter. Master it.
