In the previous session we saw that I/O-bound work benefits from overlapping waiting and that the GIL allows only one thread to run Python bytecode at a time. So for I/O-bound tasks, threads are the right tool: when one thread blocks on I/O, it releases the GIL and another thread can run.

Today we look at how to use threads in Python. We will create threads, start them, and coordinate with joins. We will also see why shared state needs a lock and how to use one correctly.

By the end of this lesson you will be able to start multiple threads for I/O-bound work, use a lock to protect shared updates, and understand the basics of thread coordination.

We use threads when we want to overlap I/O. Python’s threading module provides a Thread class. You give it a target function and optional arguments. Calling start on the thread object runs that function in a new thread. The main thread continues. So you can start several threads and they run concurrently with the main program. When the work is waiting on I/O, the interpreter can switch between threads so that waiting time is overlapped.

To wait for a thread to finish, you call join on it. So the typical pattern is: create threads, start them, then join them. That way the main program does not exit before the worker threads complete. If you start a thread and do not join it, the program might exit while the thread is still running. So join is the way to synchronize the end of the main flow with the end of the workers.

Here we define a function named fetch that takes a parameter named urlId. It sleeps for a short time to simulate network delay, then returns a string. We create a list of threads, each with fetch as the target and a different urlId. We start each thread, then we join each thread. So we run several fetches concurrently. The total time is roughly the time of one fetch, not the sum of all fetches, because the waits overlap. This is the basic pattern for I/O-bound concurrency with threads.

When multiple threads update the same variable, the updates can interleave. For example, one thread reads a counter, adds one, and writes it back. Another thread does the same. Without coordination, the final value can be wrong because both read the same value before either wrote. So we need a lock. Only one thread can hold the lock at a time. The thread that holds the lock can update the shared state; others wait. When the thread releases the lock, another can proceed. That way the read-modify-write is atomic.

Here we define a shared counter and a lock. We define a function that increments the counter inside a with statement using the lock. So each increment is protected. We start several threads that each call this function a few times. After joining all threads, the counter has the correct total. Without the lock, the total could be wrong. So for any shared mutable state that is updated from multiple threads, use a lock around the critical section.

Use threads for I/O-bound work: many network calls, file reads, or database queries where most time is spent waiting. Do not use threads to speed up CPU-bound work; the GIL will prevent parallel execution. Keep the amount of shared state small and protect it with locks. Prefer passing data in and out of the thread rather than sharing mutable globals. That keeps the design simple and avoids subtle bugs.

Threads let you overlap I/O-bound work. Create a Thread with a target and args, call start, then join to wait for completion. Protect shared mutable state with a lock; use a with statement and the lock so that only one thread updates at a time. Use threads for I/O-bound concurrency; use processes for CPU-bound parallelism. This is the foundation for building responsive and efficient Python programs that wait on many I/O operations.
