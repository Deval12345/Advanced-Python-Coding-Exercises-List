# Lecture 27: Coordination Speed Trade-off in Multiprocessing
## speech_27.md — Instructor Spoken Narrative

---

Last lecture we measured the true cost of IPC — three data copies, serialization overhead, the three failure scenarios. Today we go one level deeper. Today we talk about coordination — how multiple worker processes signal each other, how they stop each other, and why the design of that coordination determines whether your parallel system is fast or slow.

Let me start with a scenario you will encounter constantly in real systems. You have a worker pool searching for something — the first fraud signal in a transaction log, the first matching record in a distributed database, the first prime in a numerical range. Any one of your workers might find it. The moment one does, the others should stop. Not gracefully over the next few seconds. Immediately. Because every extra millisecond those workers run is wasted CPU time that costs money.

This is the cooperative early-exit problem, and the solution is a shared flag.

But before we get to flags, let us talk about shutdown — the poison-pill pattern. Imagine you have four worker processes, all pulling tasks off a Queue. When you are done — no more tasks — how do you tell them to stop? The naive approach is worker.terminate(), which kills the process immediately. That is fine for stateless workers, but it loses any in-progress task and can corrupt shared state. The correct approach is the poison-pill pattern.

You put a sentinel value — None — into the Queue once per worker. Each worker reads tasks in a loop; when it reads None, it exits cleanly. The Queue itself carries the shutdown signal, in the correct order, delivered to exactly the right number of workers. No Manager. No shared flag. No extra synchronization. The data channel delivers the control signal.

This appears in nearly every production queue consumer you will ever use. Celery workers, Kafka consumers, Redis queue processors — all of them use some version of this pattern. The reason is ordering: a separate shutdown socket or flag might arrive before the last task in the Queue. The poison-pill pattern is immune to that race because it uses the same ordered channel as the data.

Now let us move to the shared flag pattern for cooperative early-exit. Here is the design. You create a multiprocessing.Value with type code b for boolean, initialized to False. Every worker process receives this Value. Each worker checks the flag at the start of each iteration. If the flag is True, the worker returns immediately. If a worker finds the answer, it acquires the Value's internal lock, checks the flag again inside the lock, and only if it is still False does it set the flag to True and record the result. That double-check inside the lock is critical.

Why the double-check? Without it, two workers could both read flag as False, both find results simultaneously, and both try to write. With the double-check, only the first one through the lock actually sets the flag. The second arrives, sees the flag is already True, and returns without writing. This is the standard acquire-check-act-release pattern for shared state, and it prevents result corruption.

Now here is the key question: how often should a worker check the flag? This is not a trivial design decision — it is a performance-critical one. A worker that checks every single iteration pays the lock acquisition cost every iteration. A worker that checks every thousand iterations pays that cost every thousand iterations, amortized across 1,000 units of computation.

The tradeoff is between responsiveness and throughput. If each iteration costs 10 milliseconds, checking every iteration is fine — the lock cost is negligible compared to the computation. But if each iteration costs 1 microsecond, checking every iteration means you are spending as much time on coordination as on computation. Four workers each acquiring a lock one million times per second generates four million lock operations per second. That is measurable. That is real overhead.

Example 27.2 makes this concrete. Two worker implementations do identical prime-counting work; one checks the flag every ten iterations, the other every thousand. We time both with four workers. The difference reveals the coordination overhead. Your exact numbers will depend on what each iteration costs relative to the lock acquisition cost on your hardware.

The general rule is: check frequency should be inversely proportional to the cost of each iteration. Cheap iterations — check rarely. Expensive iterations — check freely.

Now I want to make sure you understand the full cost spectrum of shared-state mechanisms, because this is where many developers make expensive mistakes.

On one end of the spectrum: Manager.dict and Manager.list. These are Python objects living in a separate manager process. Every single access — every read, every write, every method call — is a full IPC round-trip. The manager process serializes the call, executes it, serializes the result, and sends it back. This adds one to five milliseconds of latency per access. For complex, flexible, infrequently-updated shared state — a configuration dictionary, an output collection written once per task — Manager objects are appropriate. For a flag checked thousands of times per second per worker, they are catastrophic.

On the other end: multiprocessing.Value and multiprocessing.Array. These live in shared memory — a memory-mapped region that the OS exposes to all participating processes. Reading a Value is a memory access, possibly with a memory barrier, costing nanoseconds. Writing a Value with the lock costs microseconds. This is orders of magnitude faster than a Manager access.

The decision framework: use Manager for data that is complex, variable-shape, or needs to hold arbitrary Python objects, and that is accessed infrequently — say, once per worker per task. Use Value for single scalars checked frequently — flags, counters, thresholds. Use Array for fixed-size numeric data that multiple workers write in-place. Use shared_memory — next lecture — for NumPy arrays where you need a zero-copy view of large data.

The fraud detection analogy brings this together. Imagine a system where four workers analyze incoming transactions in parallel. Worker one finds a fraud signal. It sets the shared flag. Workers two, three, and four — each at some point in their current-transaction analysis — check the flag, see it is set, abandon their current analysis, and move to the next transaction. The system processes the fraud signal quickly and immediately recycles all four workers to new transactions. The shared flag is the coordination mechanism; its check frequency determines how quickly the system responds to a found signal. Too infrequent, and workers waste cycles finishing analyses that do not matter. Too frequent, and the coordination cost eats into the throughput of the analysis itself.

This is the coordination speed trade-off. There is no universal right answer. There is only the correct answer for your specific iteration cost and your specific responsiveness requirement. Measure both. Choose accordingly.

The discipline to carry forward: design your coordination mechanism to match your access pattern. Poison-pill for clean shutdown via the data channel. Shared Value flag for high-frequency cooperative early-exit. Manager objects for flexible but infrequently-updated state. And always tune check frequency to match the cost of what you are checking it inside.
