# Lecture 18 — Concurrency Models Overview: I/O-Bound vs CPU-Bound Work
## Key Takeaway Points

---

- **Diagnosis before tool**: Before choosing any concurrency model, determine whether your program is I/O-bound or CPU-bound — this single question determines the correct model, the achievable speedup, and the resource cost.

- **I/O-bound definition**: A program is I/O-bound when the CPU is idle while waiting for external operations — network requests, disk reads, database queries, or API calls. The bottleneck is the wait, not the computation.

- **CPU-bound definition**: A program is CPU-bound when the CPU is running at full utilization — performing computation such as ML inference, numerical processing, encryption, or tight loops. The bottleneck is the computation itself.

- **The GIL makes the distinction a hard technical requirement**: CPython's Global Interpreter Lock allows only one thread to execute Python bytecode at a time. The GIL is released during I/O system calls, so threads can overlap I/O waits. The GIL is never released during CPU computation, so threads cannot parallelize CPU-bound Python code.

- **Threads are correct for I/O-bound work**: The `threading` module allows multiple threads to share a process and overlap their I/O waiting periods. Five threads each waiting one second for I/O finish in one second total — a real, measurable speedup.

- **Async is correct for high-concurrency I/O**: The `asyncio` module provides single-threaded cooperative concurrency via an event loop. Coroutines yield control when waiting for I/O. This model scales to thousands of concurrent connections at minimal memory and CPU cost.

- **Multiprocessing is correct for CPU-bound work**: The `multiprocessing` module spawns separate processes, each with its own Python interpreter and GIL. This achieves true parallel CPU execution at the cost of higher memory usage and inter-process communication overhead.

- **Measure, never guess**: Use `cProfile` to identify which functions consume the most time, `time.perf_counter` for section-level timing, and system tools like `top` or `htop` to observe CPU utilization vs wait states. Profiling before deciding prevents weeks of wasted effort.

- **Wrong combinations have real costs**: Using threads for CPU-bound work produces zero speedup due to the GIL. Using multiprocessing for I/O-bound work adds process overhead with no performance benefit and dramatically increases memory consumption.

- **Historical context explains the tradeoffs**: Threads were added for I/O concurrency; multiprocessing was added for CPU parallelism; asyncio was added for high-concurrency I/O at scale. Each model was built for a specific problem — none is universally correct, and understanding why each exists explains its specific tradeoffs.

- **The decision framework is the most valuable output of this lecture**: I/O-bound + moderate concurrency → threads; I/O-bound + high concurrency → async; CPU-bound + true parallelism → multiprocessing. Internalizing this framework prevents the most common and costly concurrency mistakes in Python.
