# Speech — Lecture 22: The Async Event Loop — How Cooperative Concurrency Works Under the Hood

---

Welcome back, everyone. Last lecture, we learned how to write async code — coroutines, await expressions, gather, create task, async with, async for. You can use the machinery. You know the syntax. But here is what I want to ask you today: do you know what is actually happening when your code hits an await? Do you know what the event loop is doing, step by step, while your coroutines are suspended? Do you know what happens if you violate the model's one fundamental rule?

Today, we open the engine compartment.

Let me start with the problem that made all of this necessary.

Imagine you are running a web server. You expect ten thousand simultaneous connections. The old approach — the pre-async approach — is straightforward: give every connection its own thread. One thread, one connection. The thread blocks on the network read, waits for the client to send data, handles the request, sends a response, and exits. Simple, intuitive — and completely unscalable.

Why? Because every OS thread needs a stack. On most modern systems, that stack is roughly one megabyte. So ten thousand threads means ten gigabytes of RAM — just for thread stacks. Not for your application data, not for your caches, not for your business logic. Just for the overhead of those threads existing. And on top of that, the operating system has to context-switch between ten thousand threads constantly, spending CPU time purely on scheduling. The server saturates long before it runs out of network bandwidth.

This is the wall that async I/O was invented to break through.

And the key insight is deceptively simple: most of what a web server does is waiting. A thread that is blocked on a network read is doing absolutely nothing — it is consuming a megabyte of stack and a scheduler slot, producing zero value. What if, instead of one thread per connection, you had one thread that managed all the connections — and it was smart enough to never stand still?

That is the event loop.

I want you to imagine a restaurant. Not a huge restaurant — one waiter. But this waiter is extraordinary; they are the most efficient server you have ever seen. When a customer places an order, the waiter takes the order to the kitchen and immediately walks to the next table. They do not stand at the kitchen window waiting for the food. They keep moving — table to table, taking orders, delivering drinks, checking in. When the kitchen calls out that a dish is ready, the waiter picks it up and delivers it. One person; every table served. No standing around.

That is the event loop. And each table is a coroutine.

Now let's talk about the mechanics — because understanding the mechanics is what separates a developer who can write async code from one who can debug it when it goes wrong.

The event loop maintains two data structures. First: a ready queue — a list of coroutines that are ready to run right now. Second: an I/O registry — a list of coroutines that are suspended, each paired with the file descriptor or timer they are waiting on.

Here is the three-step cycle the event loop runs continuously. Step one: run every coroutine in the ready queue until each one hits an await and voluntarily suspends. Step two: call the operating system's I/O multiplexer — that is epoll on Linux, kqueue on macOS, select on Windows — with all the file descriptors the suspended coroutines are waiting on. This is a single blocking call that tells the OS: "Wake me when anything is ready." Step three: the OS wakes the event loop and says which I/O operations completed; the event loop moves those coroutines from the I/O registry back into the ready queue. Then repeat.

There is a word in there that is doing enormous work: voluntarily. The event loop never forces a coroutine to pause. It cannot. The coroutine must reach an await expression and hand control back willingly. That is what "cooperative" in cooperative concurrency means. The coroutines cooperate. They yield. And when they yield, everything else can run.

Let's trace through what happens in our first example. We have three tasks, all launched concurrently with gather. Each one does a short CPU-intensive busy-wait loop, then awaits an asyncio sleep to simulate I/O. Watch the timestamps when you run it — task A starts first, runs its CPU loop for a tenth of a second monopolizing the event loop, then finally yields at its await. Only then does task B get to start. Then task C. But from the moment all three hit their await sleeps, they are all suspended in the I/O registry simultaneously — and the event loop is free to run other work. The one with the shortest sleep wakes up first. You are seeing the cycle in action.

Now — I want to talk about the most dangerous mistake you can make in async code. And I do not use the word dangerous lightly.

If a coroutine calls a blocking function — time.sleep, requests.get, a synchronous file read on a slow disk — it does not just pause that coroutine. It freezes the entire event loop. Every single other coroutine in your program stops making progress. The waiter has decided to stand at the kitchen window after all — and while they stand there, every table in the restaurant goes unserved.

Here is what makes this particularly insidious: the bug does not crash your server. The server just becomes mysteriously slow under load. Request latency spikes. Health checks start timing out. You look at CPU utilization — it is fine. You look at memory — fine. But your server is responding at a tenth of its normal speed, and the cause is a single blocking call in one coroutine that runs infrequently during local testing but gets hit constantly in production.

The rule — and this is a rule you should burn into memory — is: inside an async function, NEVER call anything that blocks the OS thread. Never call time.sleep; use asyncio.sleep instead. Never call requests.get; use aiohttp or httpx. Never read a large file synchronously; use aiofiles. And if you are working with a legacy library that has no async equivalent, use loop.run_in_executor.

Let's trace through what happens in our second example. The wrong version calls time.sleep directly inside a coroutine — one second of the event loop frozen solid. The right version uses run in executor: it hands the blocking function off to a thread pool, and the coroutine awaits the result. From the event loop's perspective, the coroutine is simply suspended — like it was waiting on any other I/O. The thread pool handles the blocking call on a separate thread, where it can block without affecting the event loop at all. When the thread finishes, it wakes the coroutine. Run two of those concurrently — they overlap, and the total time is one second instead of two.

Now there is one more pattern I want to teach you today, and it is subtle but important.

What about a coroutine that does legitimate CPU work in a loop — processing a batch of records, validating a dataset, transforming ten thousand items? There is no I/O, no await, no suspension point. And if that loop runs for half a second, the event loop is frozen for half a second.

The solution is a special pattern: asyncio.sleep with a delay of zero.

asyncio.sleep(0) costs essentially no real time — it tells the event loop: "Suspend me for zero seconds." That means: return to the event loop right now, let other ready coroutines run for one cycle, and then resume this coroutine immediately. It is a voluntary yield point inside CPU-heavy work. Insert it every thousand iterations, every ten milliseconds of computation — whatever granularity keeps the loop responsive.

Let's trace through what happens in our third example. We have two concurrent tasks: a long batch processor working through five thousand items, and a heartbeat that prints a timestamp every tenth of a second to simulate a background health monitor. Without the yield, the heartbeat is completely silenced during the batch — the event loop is frozen, and those hundred-millisecond heartbeat intervals simply do not fire. With the yield every thousand items, the processor cooperates — the heartbeat prints regularly, the batch completes, and then the heartbeat task is cancelled. This is cooperative multitasking in its most direct form; the programmer manually inserts the cooperation points because the code has no natural I/O boundaries to yield at.

So let me tie this all together.

The event loop is a single-threaded scheduler. It runs ready coroutines until they yield, polls the OS for completed I/O, and resumes the ones that are ready. That is the whole cycle — run, poll, resume, repeat. Coroutines yield at await expressions; they never yield anywhere else. A blocking call inside a coroutine freezes everything — use async library equivalents or run in executor. asyncio.sleep(0) yields without waiting — it is your cooperative checkpoint for long loops.

And the single-threaded nature is both the model's greatest strength and its one key constraint. Its strength: between any two await points in the same coroutine, you have the thread to yourself. No other coroutine will interleave with your code during that window. No race conditions, no locks required for local variables. Its constraint: one blocking call, anywhere, freezes the whole system. Respect the model, and it will serve you well.

Next lecture, we go deeper into Futures and Executors — the bridge between async code and the synchronous library world. See you there.
