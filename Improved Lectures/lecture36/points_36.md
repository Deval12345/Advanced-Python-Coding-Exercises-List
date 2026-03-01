# Key Points — Lecture 36: Big Project Stage 3 — Concurrency Layer Part 2 (Process Pool for CPU Work)

---

**1. CPU-bound work inside asyncio coroutines freezes the event loop**
The asyncio event loop depends on coroutines yielding control at `await` points. A CPU-bound computation — a loop over records doing arithmetic — has no `await` points. It runs to completion before the event loop can schedule anything else. If normalization of a 1,000-record batch takes 50ms, the event loop is frozen for 50ms: sensor ingestion stops, queue drains stop, heartbeats stop. Any pipeline that mixes I/O-bound ingestion (asyncio) with CPU-bound computation (Python arithmetic) must offload the CPU work to prevent this freeze.

**2. Python's GIL makes thread pools ineffective for CPU-bound work**
The Global Interpreter Lock ensures that only one OS thread executes Python bytecode at any moment. A `ThreadPoolExecutor` with 4 workers avoids blocking the event loop (each thread takes the GIL for a quantum and releases it), but the four threads cannot run Python code simultaneously — they serialize on the GIL. For I/O-bound work (threads spend time waiting, not computing), threading provides concurrency. For CPU-bound work (threads spend time computing), threading provides almost no speedup. The GIL is why CPython cannot scale CPU-bound work with threads.

**3. ProcessPoolExecutor achieves true parallelism by bypassing the GIL**
Each worker process is a separate Python interpreter with its own GIL. Four worker processes on four CPU cores can each hold their own GIL and run Python bytecode simultaneously — true parallelism. `concurrent.futures.ProcessPoolExecutor(max_workers=N)` creates N workers. For CPU-bound work, set `max_workers` to the number of physical CPU cores, not logical cores — hyperthreading provides limited benefit for CPU-bound Python. The pool is managed as a context manager (`with` block) to ensure clean worker termination.

**4. `loop.run_in_executor` is the asyncio-to-process-pool bridge**
`loop.run_in_executor(pool, function, *args)` submits `function(args)` to the pool and immediately returns an `asyncio.Future`. The event loop registers this future and continues running other coroutines — no blocking. When a worker process finishes executing `function`, it marks the future as done. The `await loop.run_in_executor(...)` expression suspends the calling coroutine until that moment, then resumes with the result. From the event loop's perspective, waiting for a process pool worker is identical to waiting for a network response — both are pending futures.

**5. IPC serialization overhead makes per-record `run_in_executor` calls prohibitively expensive**
Every `run_in_executor` call serializes (pickles) its arguments, sends them through an OS pipe to the worker process, executes the function, pickles the result, and sends it back. This round trip costs approximately 0.5-2 milliseconds regardless of how little work the function does. Calling `run_in_executor` once per record for 50,000 records incurs 25-100 seconds of IPC overhead — far more than the computation itself. Batching amortizes this cost: 50,000 records in batches of 25 means 2,000 IPC calls instead of 50,000, reducing IPC overhead by 25x.

**6. Batch size is a performance tuning parameter with opposing forces**
Smaller batches: more IPC calls (higher overhead per record), but more tasks to distribute across workers (better load balancing, workers less idle). Larger batches: fewer IPC calls (lower overhead per record), but fewer tasks (workers may be idle waiting for a large batch to be submitted). The optimal batch size is where each `run_in_executor` call takes 5-50ms — enough that IPC overhead is a small fraction of total work, but small enough that the pool is well-utilized. Profile with different batch sizes and measure total throughput.

**7. Process pool functions must be module-level and fully picklable**
Python's `multiprocessing` module serializes function objects and their arguments using `pickle`. Functions defined at module top level have a stable reference (their module name and function name) that can be pickled. Lambda functions, nested functions, and closures often cannot be pickled — they reference enclosing scope state that does not survive serialization. Arguments must also be picklable: dicts, lists, numbers, strings, and most standard types are; objects with open file handles, network sockets, or threading locks are not. Design process pool functions to be pure: data in, data out, no side effects, no external state.

**8. The queue bridge decouples async ingestion rate from process pool throughput**
The async ingestion layer may produce records faster or slower than the process pool can consume them. An `asyncio.Queue` between the two layers absorbs this rate mismatch. The ingestion coroutines put records into the queue; a separate coroutine reads from the queue, assembles batches, and submits them to the pool. The queue's `maxsize` parameter provides backpressure: if the queue is full (pool is slow), producers pause. This prevents unbounded memory growth when the pipeline is running hot.

**9. The full pipeline architecture separates concerns by execution model**
The async event loop handles all I/O-bound work: sensor reads, queue operations, result routing. The process pool handles all CPU-bound work: normalization, statistics, feature extraction. The `asyncio.Queue` is the explicit boundary between these two execution models. This separation has testing benefits: the CPU-bound functions can be tested as plain synchronous Python without any async infrastructure. The async ingestion can be tested with mock sources and queues. Each layer is independently testable, independently deployable, and independently scalable.
