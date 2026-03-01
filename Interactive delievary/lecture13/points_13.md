Slide 0
Title: Previous Lecture Takeaway
Point 1: Threads for I/O-bound with locks; for CPU-bound we need processes and IPC.

Slide 1
Title: Why Processes for CPU-Bound Work
Point 1: Processes have separate memory and separate GIL; multiple processes run Python in parallel on different cores.
Point 2: Use a *Pool* to maintain worker processes and submit work; results are collected.

Slide 2
Title: Running Work in a Pool
Point 1: *with Pool()* then *pool.map(func, args)*; each argument runs in a worker; results returned in order; no shared state so no locks in the function.
Point 2: CPU-bound work runs in parallel; total time can be much less than sequential.

Slide 3
Title: The Cost of IPC
Point 1: Data and results are serialized (pickled); IPC has a cost; use process pools for chunky work, not tiny tasks.

Slide 4
Title: Final Takeaways
Point 1: Multiprocessing for CPU-bound parallelism; *pool.map* for independent tasks; IPC cost means substantial work per task; threads for I/O, processes for CPU.
