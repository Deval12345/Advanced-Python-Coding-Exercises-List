# Slides — Lecture 32: Importance of Speed in Real Systems

---

## Slide 1 — Lecture Overview

**Performance Engineering: Measurement, Intuition, and Discipline**

- Why speed matters in production — latency and throughput as system requirements
- The measurement-first philosophy: Amdahl's Law and the profiling discipline
- timeit for microbenchmarks: quantifying individual operation costs
- cProfile for pipeline profiling: finding the actual bottleneck
- Python's performance characteristics: what is fast, what is slow
- Latency vs. throughput: two different optimization targets

---

## Slide 2 — When Slowness Becomes Failure

**At production scale, performance is a correctness issue**

Scenario: pipeline processes 50,000 records in 30 seconds; input arrives every 10 seconds

```
Time 0:  50,000 records arrive → pipeline starts (takes 30s)
Time 10: 50,000 more arrive   → waiting (pipeline still running)
Time 30: first batch done     → 2nd batch starts (already 20s stale)
Time 60: 4 batches queued     → data is 60s stale, queue growing
Time ∞:  memory exhaustion or 60+ minute stale data
```

- **Slowness = functional failure** — not just degraded quality
- Business impact: Amazon quantified +100ms latency → -1% revenue
- Google: +500ms search delay → -20% traffic
- Engineering impact: stale data → incorrect decisions; dropped records → incomplete analytics

---

## Slide 3 — Why Intuitive Optimization Is Wrong

**Amdahl's Law: speedup is bounded by the fraction of time the optimized component occupies**

| Component | Current time % | Optimized to | System speedup |
|-----------|---------------|-------------|----------------|
| Normalization | 8% | 0 (perfect) | 8.7% |
| Feature computation | 70% | 50% faster | 40% |

- Engineers optimize code they understand well, not code that is actually slow
- The 70% component looks "simple" — a loop; the 8% component looks "complex" — math
- Profile first. Always. Every time.
- **Rule: Never write optimization code until you have profiled and identified the hotspot**

---

## Slide 4 — Example 32.1: timeit Microbenchmarks

**Quantify actual costs — not expected costs**

```python
# String construction: three approaches for 5000 elements
def stringConcatLoop(n):    # O(N²): each += creates new string
    s = ""
    for i in range(n): s += str(i)

def stringJoinList(n):      # O(N): list.append then "".join()
    parts = []
    for i in range(n): parts.append(str(i))
    return "".join(parts)
```

Typical results for N=5000:
- `stringConcatLoop`:  ~8–15 ms per call
- `stringJoinList`:    ~0.8–1.2 ms per call
- **Speedup: 8–12×**

Membership test: `target in list` vs `target in set` for 10,000 elements:
- List: O(N) scan → ~150–300 µs
- Set: O(1) hash → ~0.03–0.08 µs
- **Speedup: 300–500×**

---

## Slide 5 — Example 32.2: cProfile on a Three-Stage Pipeline

**The profiler reveals what intuition hides**

```python
pr = cProfile.Profile()
pr.enable()
result = runPipeline(10000)   # filter → normalize → feature
pr.disable()
ps = pstats.Stats(pr, stream=stream)
ps.sort_stats("cumulative")
ps.print_stats(10)
```

Profiler output (approximate):
```
ncalls  tottime  cumtime  function
 10000    1.843    2.104   featureStage
 10000    0.031    0.031   normalizeStage
 10000    0.012    0.012   filterStage
```

- **featureStage dominates** — calls `sum(window)` every record (10,000 × 10 = 100,000 additions)
- normalizeStage appears complex but is vectorizable and fast
- Insight: roll the window mean incrementally, not recomputed from scratch

---

## Slide 6 — Python's Performance Characteristics

**What is fast, what is slow — and the O(1) vs O(N) distinction**

| Operation | Cost | Prefer Instead |
|-----------|------|---------------|
| `x in list` | O(N) | `x in set` — O(1) |
| `s += str(i)` in loop | O(N²) | `"".join(parts)` — O(N) |
| Python loop over 1M floats | ~slow | NumPy vectorized — 100× faster |
| `object.__dict__` (10M objects) | ~200–300 B each | `__slots__` — ~50–80 B |
| Recompute expensive result | O(cost) each call | `lru_cache` — O(1) after first |
| Materialize N-element list | O(N) memory | Generator — O(1) memory |

- Python constant factor ~10–100× slower than C per operation
- Asymptotic complexity still dominates at scale
- NumPy, SciPy: C-level speed in Python API — first-class tool for numerical work

---

## Slide 7 — The Measure → Analyze → Fix → Re-Measure Cycle

**Performance work is not guessing — it is scientific**

```
Step 1: Measure end-to-end wall clock time
        → establishes baseline and confirms there is a problem

Step 2: Profile (cProfile) to find the hotspot
        → which function takes the most cumulative time?

Step 3: Analyze WHY it is slow
        → O(N²) algorithm? Repeated work? Wrong data structure?

Step 4: Apply targeted fix
        → one change at a time, not multiple simultaneous changes

Step 5: Re-measure end-to-end
        → confirm improvement; check for regressions elsewhere
```

- Never optimize without profiling (you will optimize the wrong thing)
- Never profile without re-measuring after the fix (you will not know if it worked)
- Re-measure is also a regression check

---

## Slide 8 — Latency vs. Throughput: Two Different Optimization Targets

**Optimizing for one does not optimize for the other**

| Metric | Definition | Optimization Strategy |
|--------|-----------|----------------------|
| **Latency** | Time for one operation to complete (p50, p99, p99.9) | Reduce computation per request; minimize blocking |
| **Throughput** | Operations completed per second | Batching; parallelism; amortize fixed costs |

**Batching tension:**
- Increases throughput: one IPC round-trip for 50 records vs. 50 round-trips
- Increases latency: record 1 waits for records 2–50 before processing starts
- Real-time sensor alert: low latency required → small batches or no batching
- Overnight batch pipeline: high throughput required → large batches

---

## Slide 9 — Little's Law: Connecting Latency and Throughput

**For a stable system: Throughput = Concurrency / Latency**

```
L = λW
(avg items in system) = (arrival rate) × (avg time in system)

Rearranged:
Throughput (λ) = Concurrency (L) / Latency (W)
```

Practical implication:
- If latency is fixed at 100ms and you want 1000 req/s throughput → need 100 concurrent workers
- If you reduce latency to 50ms → need only 50 concurrent workers for same throughput
- **Reducing latency multiplies throughput capacity** at the same concurrency level

---

## Slide 10 — Python Performance Wins: The Standard Playbook

**Targeted, high-ROI optimizations confirmed by the Python community**

1. **Replace Python numerical loops with NumPy vectorization** — 10–100× speedup for array math
2. **Use generators instead of list comprehensions** — constant memory instead of O(N)
3. **Use `functools.lru_cache` for expensive repeated calls** — amortize cost across calls
4. **Use `__slots__` for objects created millions of times** — 3–5× memory reduction
5. **Use `set` for membership, `dict` for key lookup** — O(1) instead of O(N)
6. **Use `"".join(parts)` for string construction** — linear instead of quadratic
7. **Batch IPC calls** — amortize serialization cost; profile to find optimal batch size
8. **Avoid global state lookups in hot loops** — bind to local name before loop

---

## Slide 11 — Summary: The Performance Engineering Mindset

**What to carry into the big project**

- Performance is a system requirement — not a luxury, not "phase 2"
- At scale, slow = broken (pipeline cannot keep up → data is stale or lost)
- Measure first, always — `timeit` for microbenchmarks, `cProfile` for system hotspots
- Amdahl's Law: optimize the 70%, not the 8%
- Python's fast path: sets, generators, NumPy, `__slots__`, `lru_cache`, join
- Latency and throughput are different problems: define which you are optimizing for
- Next: the big project — all concepts combined in a real streaming analytics pipeline

---
