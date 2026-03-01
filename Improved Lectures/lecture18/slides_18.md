# Lecture 18 — Concurrency Models Overview: I/O-Bound vs CPU-Bound Work
## Slides

---

## Slide 1
**Title:** From Object Model to Concurrency

- Previous chapter: how Python builds and manages individual objects
- Descriptors, ABCs, memory layout, slots — internal structure
- This chapter: how Python handles multiple things at once
- Concurrency is not one tool — it is a diagnosis followed by a prescription

---

## Slide 2
**Title:** The Only Question That Matters First

- Before choosing any concurrency tool, answer one question
- What is your program actually waiting for?
- The answer determines the model, the speedup, and the resource cost
- Getting the diagnosis wrong is one of the most expensive engineering mistakes

---

## Slide 3
**Title:** I/O-Bound Work — Waiting for the Outside World

- CPU is idle while waiting: network, disk, database, external APIs
- Example: 5 API calls × 1 second each = 5 seconds sequential
- CPU does almost no work during this time — it is pure waiting
- Key insight: these waiting periods can overlap — that is the speedup opportunity

---

## Slide 4
**Title:** CPU-Bound Work — The CPU Is the Bottleneck

- CPU is running at full utilization: computation, ML inference, encryption, parsing
- No idle waiting — the computation itself is the bottleneck
- Adding concurrency without more CPU power does not help
- Example: 4 heavy computations × 1 second each = 4 seconds, with or without threads (in Python)

---

## Slide 5
**Title:** The Global Interpreter Lock (GIL) — Why Diagnosis Matters in Python

- CPython's GIL: only one thread executes Python bytecode at a time
- GIL is released during I/O system calls — threads can overlap I/O waits
- GIL is NOT released during CPU computation — threads cannot parallelize CPU work
- This makes the I/O-bound vs CPU-bound distinction a hard technical requirement in Python

---

## Slide 6
**Title:** Measuring to Know — Profiling Before Deciding

- Never guess your bottleneck — measure it
- `cProfile`: shows which functions consume the most time
- `time.perf_counter`: section-level wall-clock timing for before/after comparisons
- System tools (`top`, `htop`): is CPU at 100% (CPU-bound) or low with wait states (I/O-bound)?
- Teams waste weeks optimizing the wrong bottleneck — measurement is the prevention

---

## Slide 7
**Title:** Python's Three Concurrency Models

- **Threads** (`threading`): shared memory, good for I/O-bound, limited by GIL for CPU work
- **Async/await** (`asyncio`): single-threaded event loop, cooperative, scales to thousands of I/O tasks
- **Multiprocessing** (`multiprocessing`): separate processes, separate GILs, true CPU parallelism
- Each model was invented to solve a different problem — none is universally correct

---

## Slide 8
**Title:** The Decision Framework

- I/O-bound + moderate concurrency (dozens–hundreds) → **threads**
- I/O-bound + high concurrency (thousands of connections) → **async**
- CPU-bound + need for true parallelism across cores → **multiprocessing**
- Wrong combinations: threads on CPU-bound (GIL blocks), multiprocessing on I/O-bound (overhead, no gain), async on CPU-bound (blocks event loop)

---

## Slide 9
**Title:** Sequential vs Threaded I/O — The Speedup Is Real

- Sequential: 5 tasks × 1s each = ~5.0s total
- Threaded: 5 tasks launched simultaneously, all wait in parallel = ~1.0s total
- 5x speedup for the same 5 tasks — the GIL releases during sleep/I/O
- This speedup does NOT apply to CPU-bound work — threads on CPU-bound take the same time

---

## Slide 10
**Title:** Why Python Has Three Models — Historical Context

- Python started single-threaded; GIL added for thread-safe memory management
- Threads added for I/O concurrency; GIL released during system calls
- Multiprocessing added for CPU parallelism (bypasses GIL with separate processes)
- asyncio added (Python 3.5) for high-concurrency I/O — one event loop, thousands of tasks, minimal overhead

---

## Slide 11
**Title:** The Cost of Getting the Diagnosis Wrong

- Threads on CPU-bound ML inference: zero speedup, two weeks of tuning, late launch
- Multiprocessing on I/O-bound web API: 20GB RAM vs 2GB with async — months of over-provisioning
- Cost is not just wrong code — it is weeks of engineering time and infrastructure spend
- First question always: I/O-bound or CPU-bound? Then measure to confirm

---

## Slide 12
**Title:** Summary and What Comes Next

- Diagnosis first: I/O-bound or CPU-bound?
- I/O + moderate concurrency → threads; I/O + high concurrency → async; CPU-bound → multiprocessing
- Measure with `cProfile` and `time.perf_counter` before deciding
- Next lecture: `threading` in depth — GIL mechanics, thread safety, locks, and I/O-bound patterns
