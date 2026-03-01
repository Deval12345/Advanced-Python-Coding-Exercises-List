# I/O-Bound Versus CPU-Bound Work and the GIL

## Lesson Opening

In the previous session we looked at memory layout and the special dunder method slots. From here we shift to concurrency. Before we write threaded or multiprocessing code, we need to understand when threads help and when they do not. That depends on whether the work is I/O-bound or CPU-bound, and on Python’s global interpreter lock, the GIL.

By the end of this lesson you will understand the difference between I/O-bound and CPU-bound work, what the GIL is in one sentence, and why that matters for choosing threads versus processes.

------------------------------------------------------------------------

## Section 1 — I/O-Bound Versus CPU-Bound

### CONCEPT 0.1

When your program spends most of its time waiting for something outside the CPU, we say the work is I/O-bound. Waiting for a network response, reading from disk, or waiting for a database query are I/O-bound. When your program spends most of its time doing calculations in the CPU, we say the work is CPU-bound. Crunching numbers, parsing large data, or running a tight loop are CPU-bound. The distinction matters because Python handles these two cases differently.

### CONCEPT 1.1

For I/O-bound work, the CPU is often idle. While one task waits for the network, another task could run. So overlapping many I/O operations can improve throughput without needing more CPU cores. Threads are useful here because when one thread is waiting, the interpreter can switch to another thread. For CPU-bound work, the CPU is busy. Adding more threads in Python does not give you more CPU time per core because of the GIL. So for CPU-bound work we need a different strategy: multiple processes.

------------------------------------------------------------------------

## Section 2 — The Global Interpreter Lock in One Sentence

### CONCEPT 2.1

The GIL is a lock inside the Python interpreter that ensures only one thread executes Python bytecode at a time. So at any moment, only one thread is actually running Python code. That simplifies memory management and keeps the implementation correct. But it means that if you have multiple threads doing heavy computation, they do not run in parallel on multiple cores. They take turns. So CPU-bound threads do not scale with the number of cores. I/O-bound threads still help, because when a thread is waiting for I/O, it releases the GIL and another thread can run.

### CONCEPT 2.2

So the rule of thumb: use threads when the work is I/O-bound and you want to overlap waiting. Use multiple processes when the work is CPU-bound and you need real parallelism. The GIL is why Python favors this split. We will see threading in the next lecture and multiprocessing after that.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 3.1

I/O-bound means most time is spent waiting for I/O; CPU-bound means most time is spent computing. Threads help with I/O-bound work by overlapping waiting; they do not give parallel CPU execution because of the GIL. The GIL allows only one thread to execute Python bytecode at a time. For CPU-bound parallelism use processes. Understanding this avoids the mistake of adding threads for CPU-heavy work and expecting speedup.
