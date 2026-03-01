# Lecture 31: Concurrency Architecture Patterns
## speech_31.md — Instructor Spoken Narrative

---

By now you understand the individual concurrency tools — asyncio for I/O, thread pools for synchronous libraries, process pools for CPU-bound computation. You have seen each one in isolation; you have seen how the GIL shapes your choices; you have seen how race conditions and deadlock arise when you coordinate threads carelessly.

Today we step up one level. Not tools — patterns. The structural templates that appear again and again in production concurrent systems, regardless of framework or industry.

Let me start with a question that senior engineers answer differently than junior ones. Someone comes to you and says: "We need to make this service faster. Should we use async or threading?" What do you say?

The wrong answer is to pick one and justify it. The right answer is: it depends on the nature of the work — and there are three different cases, each with its own correct answer.

---

The first case: the work is I/O-bound, and the libraries you use support async. Network calls with httpx, database queries with asyncpg, file reads through asyncio — all of these can be awaited. The event loop suspends the coroutine at the await point, runs other coroutines, and resumes when the operation completes. One thread handles hundreds of concurrent operations. No thread pool needed. Async is the correct and most efficient tool.

The second case: the work is I/O-bound, but the libraries you use are synchronous. Legacy database drivers, old HTTP clients, any library that blocks. You cannot await a blocking call. If you try, you freeze the event loop for the duration. The correct tool here is a thread pool — `ThreadPoolExecutor` — and the bridge is `loop.run_in_executor`. The blocking call runs in a thread; from the event loop's perspective, it looks like any other awaitable.

The third case: the work is CPU-bound. Real computation — parsing, compression, numerical transforms, machine learning inference. Threading cannot help here; the GIL ensures that at most one thread runs Python bytecode at any moment. The correct tool is a `ProcessPoolExecutor`. Each worker process has its own interpreter and its own GIL. True parallelism. Four processes on four cores — four times the throughput.

This is the three-layer model. Each layer handles a specific workload type. The sophistication is knowing which layer applies to which part of your system — and never mixing them incorrectly.

In Example 31.1, you see all three layers working together. An async gateway handles concurrent incoming requests. Each request needs two things: a legacy synchronous library call — sent to the thread pool — and a CPU-heavy computation — sent to the process pool. The async handler orchestrates both with `run_in_executor`, staying unblocked throughout. Eight requests, all handled simultaneously, with thread pool and process pool running in parallel on their respective work.

Without this architecture, you are forced into bad choices. A developer who only knows threads uses threads for everything — the GIL limits CPU parallelism to single-thread performance, and they wonder why their server does not scale. A developer who loves async tries to run CPU computation inline in coroutines — the event loop freezes, and they wonder why response times spike under load. The three-layer model prevents these category errors by giving each kind of work its correct home.

---

The second pattern: fan-out, fan-in.

Think about how a distributed search engine handles a query. Your search does not go to one server. The query fans out to dozens of shards — each shard searches its portion of the index. The results fan back in — aggregated, ranked, returned. Total latency is not the sum of all shard latencies. It is the maximum, plus a small coordination cost.

Fan-out is the simultaneous dispatch: one driver issues N concurrent operations. Fan-in is the aggregation: one collector merges N results into a unified answer. The mathematical improvement is significant. Eight shards, 200 milliseconds each, queried sequentially: 1600 milliseconds. Queried simultaneously: 200 milliseconds. An eight-fold latency reduction.

`asyncio.gather` is the standard fan-out primitive. You create N tasks; gather runs them all simultaneously; it returns when all complete. But there is a subtlety that matters enormously in production: what happens when one backend is slow?

If one of eight shards takes two seconds while the others respond in 200 milliseconds, gather waits for the slow one. Your response time is two seconds — dominated by a single outlier. This is the tail latency problem. It is not hypothetical; in distributed systems with many backends, the probability that at least one is slow at any given moment is surprisingly high.

The solution is `asyncio.wait` with a timeout. You set a deadline — say, 300 milliseconds. Wait dispatches all tasks simultaneously. When 300 milliseconds pass, it returns two sets: the tasks that completed, and the tasks that are still pending. You take the completed results, cancel the pending tasks, and assemble a partial answer. The query returns in 300 milliseconds, with most shards' data, rather than two seconds with all data.

The engineering tradeoff is explicit: you accept an incomplete result in exchange for a predictable, fast response. Google's engineering literature calls this "good enough." In web search, returning 7 shards' results in 300ms is far better than returning 8 shards' results in 2 seconds. The user experience is better; the system load is better; the slow shard gets time to recover.

Example 31.2 demonstrates this exactly. Eight simulated shards, each with random latency between 50 and 400 milliseconds. A 300ms deadline. The code shows which shards responded, how many timed out, and the final record count from the partial answer.

---

The third pattern: the circuit breaker.

Here is the failure scenario. Your service calls a downstream microservice for every request. The downstream service starts degrading — it is slow, timing out after 5 seconds. Your service has 100 simultaneous requests in flight. Each one is blocked on the downstream call. Each one holds resources — a connection slot, a thread or coroutine, memory. The 100 requests are consuming resources for 5 seconds each. New requests arrive. They also call the downstream service. They also block for 5 seconds. Resources are exhausted. Your service can no longer handle new requests — even for operations that do not touch the downstream service. One degraded dependency has cascaded into a complete system failure.

This is the cascading failure pattern. It is one of the most common failure modes in microservice architectures. And the circuit breaker is the solution.

The circuit breaker tracks failure rates. When failures exceed a threshold — say, 5 failures in 10 seconds — it opens the circuit. Subsequent requests fail fast: they do not contact the downstream service at all; they immediately return an error or a fallback response. The downstream service is no longer being flooded with requests from clients that cannot be served anyway. It gets time to recover.

After a reset interval, the circuit enters the half-open state: it allows one probe request through. If the probe succeeds, the circuit closes — normal operation resumes. If the probe fails, the circuit opens again and the timer resets. Three states, one safety mechanism.

The origin of this name is literal: electrical circuit breakers work exactly the same way. Too much current? The breaker opens the circuit, preventing damage. The current stops; the element cools; the breaker resets. Software circuit breakers are named for this analogy by Michael Nygard in his 2007 book "Release It!" — still the canonical reference on production software resilience.

---

Let me close with the decision framework — the four questions you ask when designing a concurrent system.

First: is this work I/O-bound or CPU-bound? I/O-bound means waiting for the world; CPU-bound means computing about it. If it is I/O-bound, you need concurrency. If it is CPU-bound, you need parallelism.

Second: for I/O-bound work, can you use async-compatible libraries? Yes — use asyncio directly. No — use a thread pool through run_in_executor.

Third: at what scale? Thousands of simultaneous connections — async is the only practical choice; threads cannot scale to thousands. Dozens of parallel CPU operations — a process pool with worker count equal to CPU count.

Fourth: what are the failure modes? Async services need circuit breakers and timeout handling for slow or failed dependencies. Process pools need to handle worker crashes; the work must be retried if a worker dies. Thread pools need to avoid lock contention; the bottleneck shifts from computation to coordination at high contention.

These four questions are not arbitrary. They are the crystallized experience of thousands of production systems that got the answers wrong and paid for it. The decision framework is the reason senior engineers can evaluate a concurrent architecture quickly — not because they have memorized rules, but because they understand why each rule exists.

---

The three concurrency layers. Fan-out and fan-in with timeout. The circuit breaker for dependency resilience. The decision framework for choosing correctly under any workload.

In the next part of the course, we begin the big project — a real analytics pipeline that uses all of these concepts together. What you have learned about concurrency, you will apply. What you have learned about data models, protocols, decorators, generators, and context managers — you will apply those too. The project is the course, assembled.

