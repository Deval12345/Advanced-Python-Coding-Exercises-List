Slide 0
Title: Previous Lecture Takeaway
Point 1: Memory layout and *slots* reduce per-instance overhead; we now shift to concurrency and when to use threads vs processes.

Slide 1
Title: I/O-Bound Versus CPU-Bound
Point 1: I/O-bound: most time waiting for network, disk, DB; CPU-bound: most time computing.
Point 2: Overlapping I/O improves throughput; threads help for I/O-bound; for CPU-bound we need multiple processes.

Slide 2
Title: The GIL in One Sentence
Point 1: The GIL ensures only one thread executes Python bytecode at a time; CPU-bound threads do not run in parallel; I/O-bound threads release the GIL while waiting.
Point 2: Use threads for I/O-bound; use processes for CPU-bound parallelism.

Slide 3
Title: Final Takeaways
Point 1: I/O-bound vs CPU-bound dictates threads vs processes; GIL limits parallel bytecode execution; use processes for CPU-bound speedup.
