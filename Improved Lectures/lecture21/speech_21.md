# Speech — Lecture 21: Async/Await Fundamentals — Cooperative Concurrency with a Single Thread

---

Welcome back, everyone. Last lecture we explored threading — spinning up multiple OS threads so that I/O operations can overlap. And threading works. It genuinely works. But I want you to think carefully about what a thread actually costs you.

Every OS thread needs a stack. On most modern systems, that stack is roughly one megabyte. One megabyte per thread. Now ask yourself: what happens when you are writing a web server that needs to handle ten thousand simultaneous connections? You do the math — that is ten gigabytes of RAM, just for thread stacks. Not for your application logic, not for your data — just for the overhead of having those threads exist. And on top of that, the operating system scheduler has to juggle all ten thousand of them. Context-switching overhead becomes measurable. Latency climbs even when your CPU is sitting mostly idle.

This is the scalability wall. And it is exactly what async/await was invented to break through.

Back in 2009, Node.js showed the world something surprising: a single-threaded event loop could handle fifty thousand concurrent connections, with a fraction of the memory that thread-per-connection servers require. Python's asyncio — introduced in version 3.4, with the clean async/await syntax arriving in Python 3.5 — brings that same architectural idea into Python. Today, every major async Python framework — FastAPI, aiohttp, Starlette, Sanic — is built on top of this model. If you are writing high-concurrency API servers, websocket servers, or data pipeline ingestion layers, async Python is the default architecture.

So — how does it work?

The core concept is the coroutine. A coroutine is defined with the "async def" keywords, and it looks almost exactly like a normal function. But there is one critical difference: it can pause. At any point where you write the "await" keyword, the coroutine voluntarily hands control back to the event loop and says — "I am waiting on something; go run something else." The event loop notes that this coroutine is paused, switches to the next ready coroutine, and comes back when the I/O finishes.

This is called cooperative concurrency. The coroutines cooperate — they yield control willingly. The event loop never forcibly interrupts them. That distinction matters enormously, as we will see.

Now, before async/await existed, the way to handle this was callbacks. You would register a function to be called when some I/O finished, and then that callback would register another callback, and so on. If you have ever heard of "callback hell" — this is what people mean. Your sequential logic gets shredded into dozens of tiny functions, and error handling becomes a nightmare. Async/await lets you write I/O-concurrent code that reads top-to-bottom, like ordinary sequential code. That is the real win.

Let's look at a concrete example. Imagine we have a coroutine called fetchData that simulates fetching from a remote source; it takes a source identifier and a delay, prints that it has started, then awaits a sleep to simulate network latency, and returns the result. Now — calling fetchData does not actually start the execution. It returns a coroutine object. Think of it like a recipe — calling the function gives you the recipe, but the event loop is the one who actually cooks it.

When we use asyncio.gather to run three of these fetches simultaneously — one that takes a full second, one that takes half a second, one that takes 0.8 seconds — something remarkable happens. All three start immediately. All three suspend at their await points almost instantly. The event loop watches all three. The 0.5-second one finishes first, then the 0.8-second one, then the 1-second one. Total elapsed time? Roughly one second — the length of the longest individual fetch. Not 2.3 seconds, which is what you would get running them sequentially. That is the power of concurrent I/O in a single thread.

Now let's talk about tasks; because sometimes you want more fine-grained control than gather gives you.

asyncio.create_task lets you schedule a coroutine to run in the background — immediately — and continue doing other work right now. Think of it like launching a thread and getting a handle back, except it is all still single-threaded. The task starts running the moment the event loop gets its next opportunity. You can check on it later, wait on it with a timeout, or cancel it if you no longer need the result.

Let's look at an example. We create three tasks immediately — one job that will take two seconds, one that takes half a second, one that takes one second. Then we use asyncio.wait with a timeout of 1.2 seconds. After 1.2 seconds, the two shorter jobs have finished; the long one is still pending. We cancel it. Here is the important detail: the cancellation is delivered at the nearest suspension point inside that task — the await inside the job. That is where the event loop raises a CancelledError. The task catches it, does any cleanup it needs, and then re-raises. Always re-raise CancelledError after cleanup; this is a firm rule. If you swallow it, the event loop cannot know the task actually stopped, and you will have a subtle, hard-to-find bug.

This pattern — create tasks, wait with a timeout, cancel stragglers — is used constantly in production. Health-check tasks, background metric flushers, periodic cleanup jobs: they are all independent tasks running alongside your main request-handling coroutines.

Now there is one more layer of the async model I want to cover, and this is the part that trips people up most often — async context managers and async iteration.

You already know the "with" statement: open a resource, use it, close it — guaranteed, even if an exception occurs. And you know the "for" loop: iterate through a sequence. Both of those are synchronous by design. But what if opening a database connection requires a network round-trip? What if iterating through a result set means fetching rows from the database one at a time? Doing that with the synchronous versions would block the event loop entirely.

That is why Python introduced async with and async for. The mental model is simple: they are exactly like their synchronous counterparts, except that the setup and teardown — and each step of iteration — are allowed to suspend the coroutine at the awaited steps.

Let's look at an example that brings this together. We build an AsyncDatabaseConnection class. When we enter the "async with" block, the event loop awaits the open operation — simulating a real network handshake. While that is happening, other coroutines can run. Inside the block, we use "async for" to iterate through rows — each row fetch awaits a small delay, simulating real database network I/O. Between every single row, the event loop gets a chance to do other work. When the "async with" block exits — whether normally or because of an exception — the close operation is awaited, cleanly tearing down the connection. No manual try/finally required.

This pattern is how every async database driver works: asyncpg, aiosqlite, motor for MongoDB. It is how async HTTP clients like aiohttp and httpx work. And async iteration specifically is what makes streaming possible — you can process gigabytes of data one chunk at a time, with constant memory usage, because you are never holding the whole dataset in memory at once.

So now the big question: when should you use async, and when should you use threads?

The answer is about the nature of your workload and your library ecosystem. Use async when you have high-concurrency I/O — thousands of simultaneous connections — and you are able to use async-compatible libraries throughout your stack. The single-threaded model means there are no race conditions for shared variables between await points; that is a genuine safety benefit. But — and this is critical — any blocking call freezes the entire event loop. If you call a synchronous database driver directly from a coroutine, every other coroutine in your server stalls until that call returns.

Use threads when you are working with synchronous-only libraries — legacy database drivers, encryption libraries, PDF generation. Threads are also fine for simpler concurrent I/O where you do not need to handle tens of thousands of simultaneous operations.

And here is the most important insight for modern production Python: you do not have to choose just one. You can use async for your request-handling layer and offload synchronous library calls to a thread pool using run_in_executor. That combination gives you the scalability of async and the compatibility of threads — the best of both worlds.

To summarize: async/await gives you cooperative concurrency in a single thread. Coroutines pause voluntarily at await expressions. gather runs a fixed set of coroutines concurrently and collects all their results. create_task schedules independent background work. async with and async for bring the familiar resource and iteration patterns into the async world. And the choice between async and threads comes down to your concurrency scale and your library ecosystem.

Next lecture, we open the engine compartment entirely — we will look at how the event loop itself works under the hood. See you there!
