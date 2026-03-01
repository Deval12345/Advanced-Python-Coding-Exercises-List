# Speech — Lecture 36: Big Project Stage 3 — Concurrency Layer Part 2 (Process Pool for CPU Work)

---

Last lecture we solved one problem beautifully. Ten sensors, 100 milliseconds of network latency each — asyncio reads all ten simultaneously, the event loop switches between them at every `await`, and we get ten readings in 100 milliseconds instead of a thousand. Elegant. Efficient. Correct.

Now the pipeline has to do something with those records. And this is where we encounter the other kind of slowness — not the waiting kind, but the working kind.

Our pipeline stages need to normalize values. Compute rolling statistics. Extract features for downstream analytics. This is computation — real arithmetic on real data — and it takes CPU time. Not much per record. Maybe 10 microseconds. But with 50,000 records per second flowing through, that is 500 milliseconds of CPU work per second — which means the CPU is fully saturated, just keeping up.

And here is the problem: if you run that computation inside an asyncio coroutine, you have broken the event loop.

Let me be very specific about why. The event loop's concurrency model is cooperative. Coroutines take turns. A coroutine runs until it hits an `await` and then steps aside, letting the event loop run other coroutines. But a CPU-bound computation has no `await` points. It just runs — loops through records, does arithmetic, appends results — for its entire duration. The event loop cannot interrupt it. No other coroutine runs until it finishes. If your normalization stage takes 100 milliseconds to process a batch of records, the event loop is frozen for 100 milliseconds. During those 100 milliseconds, your sensors are sending readings that nobody is accepting. Buffers fill. Readings are dropped.

This is the fundamental incompatibility. Asyncio is designed for I/O-bound work — work that waits. CPU-bound work — work that computes — does not wait. It runs.

The solution is to take the CPU-bound work out of the event loop entirely and hand it to a process pool.

Now — thread pool, you might say. Couldn't we use a thread pool? Yes, but with a critical limitation. Python has a Global Interpreter Lock — the GIL. The GIL ensures that only one thread runs Python bytecode at a time. Threading in Python gives you concurrency — threads take turns running — but not parallelism. For CPU-bound work, all your threads compete for the same GIL, so you get no speedup from multiple threads. Four threads on four CPU cores running Python bytecode all wait for each other.

Processes are different. Each process has its own Python interpreter and its own GIL. Processes share nothing by default. Four processes on four CPU cores can all run Python bytecode simultaneously — true parallelism. A `ProcessPoolExecutor` with four workers gives you four-fold parallelism on CPU-bound work. That is the tool we need.

`concurrent.futures.ProcessPoolExecutor` creates a pool of worker processes. You submit work to it with `pool.submit(function, args)` or through the asyncio bridge `loop.run_in_executor(pool, function, args)`. The function and its arguments are serialized — pickled — sent to a worker process over IPC, executed, and the result is pickled back. From the asyncio side, this looks like any other `await` — a Future that resolves when the worker is done.

And that is the elegant part. From the event loop's perspective, `await loop.run_in_executor(pool, processRecordBatch, batch)` is identical to `await asyncio.sleep(0.1)`. It is a pending operation that will complete eventually. The event loop continues ingesting sensor readings. The process pool workers run the computation in parallel. The two layers operate independently, linked only by the Future mechanism.

There is one design choice that looks minor but has enormous performance implications: batching.

Each `run_in_executor` call involves IPC overhead. The function arguments are pickled, sent through a pipe to the worker, the worker unpickles them, runs the function, pickles the results, and sends them back. This round trip costs time — perhaps 0.5 milliseconds, perhaps more, depending on the data size. If you call `run_in_executor` once per record, you pay that overhead 50,000 times for 50,000 records. At 0.5ms per call, that is 25 seconds of IPC overhead for a computation that might take 1 second if done synchronously.

Batching amortizes this cost. Instead of calling `run_in_executor` with one record, call it with 20 records. You pay the IPC overhead once. The worker processes 20 records, returns 20 results. Your IPC overhead per record drops from 0.5ms to 0.025ms. A factor of 20 improvement in IPC efficiency, with the computation cost unchanged.

The batch size is a tuning parameter. Too small: IPC overhead dominates. Too large: fewer batches to distribute across workers, so workers sit idle waiting for the next batch. For 400 records and 4 workers, a batch size of 20 gives 20 batches — enough to keep all 4 workers busy with 5 batches each. The right batch size depends on the ratio of IPC overhead to computation time per record.

In the code, the pattern looks like this. We create all the records first. We slice them into batches. We create a Future for each batch with `loop.run_in_executor`. We `asyncio.gather` all the Futures — all batches submit to the pool simultaneously. The pool assigns batches to workers. Workers run in parallel. When all are done, we flatten the results.

The `with ProcessPoolExecutor(max_workers=4) as pool:` block is important. The `with` statement ensures that when the block exits — normally or via exception — all worker processes are terminated cleanly. Without this, worker processes can become orphaned — continuing to run after the main process has finished or crashed. Always use the context manager form.

There is something else to understand about the process boundary. Functions submitted to the pool must be picklable. This means they must be defined at the module top level, not as lambdas or nested functions. The arguments must be picklable — dicts are, most standard types are, but objects with file handles or sockets are not. If you try to send an unpicklable object to a worker, you get a `PicklingError` that can be confusing. The design principle is: process pool workers are stateless. They receive data, process it, return results. They do not hold open connections or maintain state between calls.

This is actually a good architectural constraint. Stateless workers are easy to reason about, easy to test, and easy to scale. Add more workers — the computation scales linearly. The state lives in the main process; the computation is distributed.

Let me put the two lectures together. Sensor data arrives — async ingestion, event loop, multiple sensors simultaneously. Records flow into an asyncio.Queue. The main process takes batches from the queue and submits them to the process pool. Workers process the batches in parallel. Results flow back. The pipeline is fully concurrent at both layers — I/O-bound work in the event loop, CPU-bound work in the process pool.

This is how production data pipelines handle the two flavors of slowness: waiting for the world, and computing about the world. Each kind of slowness has its own tool. The art is knowing which tool applies where, and connecting them cleanly.
