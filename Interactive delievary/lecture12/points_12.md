Slide 0
Title: Previous Lecture Takeaway
Point 1: I/O-bound vs CPU-bound and GIL; threads help for I/O-bound; today we use the threading module.

Slide 1
Title: Starting Threads
Point 1: Create a *Thread* with target and args; call *start()*; threads run concurrently; use *join()* to wait for completion.
Point 2: Pattern: create threads, start them, join them so the main program does not exit early.

Slide 2
Title: Shared State and Locks
Point 1: Multiple threads updating the same variable can interleave; use a lock so only one thread updates at a time (atomic read-modify-write).
Point 2: Use *with lock:* around the critical section; after joining, shared state has the correct total.

Slide 3
Title: When to Use Threads
Point 1: Use for I/O-bound work (network, file, DB); not for CPU-bound; keep shared state small and lock-protected.

Slide 4
Title: Final Takeaways
Point 1: Threads for I/O overlap; start then join; protect shared state with a lock; use processes for CPU-bound parallelism.
