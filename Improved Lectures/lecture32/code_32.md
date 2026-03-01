# Code — Lecture 32: Importance of Speed in Real Systems

---

## Example 32.1 — timeit: Microbenchmarking Individual Operations

```python
# Example 32.1
import timeit

def stringConcatLoop(n):
    s = ""
    for i in range(n):
        s += str(i)
    return s

def stringJoinList(n):
    parts = []
    for i in range(n):
        parts.append(str(i))
    return "".join(parts)

def stringJoinComprehension(n):
    return "".join(str(i) for i in range(n))

N = 5000
repeats = 200

t1 = timeit.timeit(lambda: stringConcatLoop(N), number=repeats)
t2 = timeit.timeit(lambda: stringJoinList(N), number=repeats)
t3 = timeit.timeit(lambda: stringJoinComprehension(N), number=repeats)

print(f"String concat loop:      {t1/repeats*1000:.3f} ms per call")
print(f"List + join:             {t2/repeats*1000:.3f} ms per call")
print(f"Comprehension + join:    {t3/repeats*1000:.3f} ms per call")
print(f"Speedup (concat→join):   {t1/t2:.1f}x")

listData = list(range(10000))
setData = set(range(10000))
target = 9999

t_list = timeit.timeit(lambda: target in listData, number=100000)
t_set  = timeit.timeit(lambda: target in setData,  number=100000)
print(f"\nMembership test in list: {t_list/100000*1e6:.2f} µs")
print(f"Membership test in set:  {t_set/100000*1e6:.2f} µs")
print(f"Speedup (list→set):      {t_list/t_set:.1f}x")
```

### Line-by-Line Explanation

**`def stringConcatLoop(n):`**
The naive string-building approach. Each iteration evaluates `s += str(i)`, which in Python (for strings) allocates a new string of length `len(s) + len(str(i))`, copies both strings into it, and discards the old string. For N=5000, this creates 5000 temporary strings of lengths 0, 1, 2, ..., up to ~18000 characters. Total characters copied is 0+1+2+...+~18000 ≈ O(N²). This is the "string concatenation in a loop" antipattern.

**`def stringJoinList(n):`**
The correct approach. `parts.append(str(i))` is O(1) amortized (list resizing is amortized constant time). `"".join(parts)` makes a single pass over all parts, computes the total length once, allocates one output string, copies all parts in. Total cost: O(N). The speedup over the naive approach is typically 8–15× for N=5000.

**`def stringJoinComprehension(n):`**
Generator expression version of the same pattern. `str(i) for i in range(n)` is a lazy generator — it does not materialize all strings into a list. `"".join(...)` accepts any iterable, so it can consume the generator directly. The performance is very similar to `stringJoinList` — join internally buffers the iterable anyway. Use whichever is more readable.

**`t1 = timeit.timeit(lambda: stringConcatLoop(N), number=repeats)`**
`timeit.timeit` runs the callable `number` times and returns the total elapsed time in seconds. The lambda wraps the call so `N` is captured from the enclosing scope. `number=200` gives statistically stable results while keeping measurement time reasonable. Dividing by `repeats` gives time per call; multiplying by 1000 converts to milliseconds.

**`listData = list(range(10000))`**
Creates a list with 10,000 integers. `target in listData` must scan sequentially from index 0 until it finds 9999 — the last element. Expected scan length: 10,000 comparisons per lookup.

**`setData = set(range(10000))`**
Creates a set with the same 10,000 integers. `target in setData` computes `hash(target)`, finds the corresponding hash bucket, and does at most a few equality comparisons. Expected scan length: O(1) — typically 1 comparison, rarely 2–3 due to hash collisions.

**`t_list = timeit.timeit(lambda: target in listData, number=100000)`**
100,000 repetitions for the list lookup — needed because one lookup is so fast (~150µs) that fewer repetitions would give noisy results. The total time divided by 100,000 gives the per-lookup cost.

---

## Example 32.2 — Profiling a Slow Pipeline Stage with cProfile

```python
# Example 32.2
import cProfile
import pstats
import io
import random
import math
from collections import deque

def generateRecords(n):
    return [{"id": i, "value": random.gauss(50, 15)} for i in range(n)]

def filterStage(records, minVal, maxVal):
    return [r for r in records if minVal <= r["value"] <= maxVal]

def normalizeStage(records, minVal, maxVal):
    span = maxVal - minVal
    return [{**r, "norm": (r["value"] - minVal) / span} for r in records]

def featureStage(records, windowSize=10):
    window = deque(maxlen=windowSize)
    out = []
    for r in records:
        window.append(r["norm"])
        mean = sum(window) / len(window)
        variance = sum((x - mean) ** 2 for x in window) / len(window)
        out.append({**r, "mean": mean, "var": variance})
    return out

def runPipeline(n):
    records = generateRecords(n)
    filtered = filterStage(records, 20.0, 80.0)
    normalized = normalizeStage(filtered, 20.0, 80.0)
    featured = featureStage(normalized)
    return featured

if __name__ == "__main__":
    pr = cProfile.Profile()
    pr.enable()
    result = runPipeline(10000)
    pr.disable()

    stream = io.StringIO()
    ps = pstats.Stats(pr, stream=stream)
    ps.sort_stats("cumulative")
    ps.print_stats(10)
    print(stream.getvalue())
    print(f"Pipeline output: {len(result)} records")
```

### Line-by-Line Explanation

**`def generateRecords(n):`**
Creates N records as dictionaries. `random.gauss(50, 15)` generates values from a normal distribution with mean 50 and standard deviation 15. Approximately 99.7% of values fall between 5 and 95. The list comprehension creates all N records at once and holds them in memory simultaneously.

**`def filterStage(records, minVal, maxVal):`**
Filters records to the range [20, 80]. Uses a list comprehension — creates a new list of qualifying records. Approximately 83% of normally-distributed values with mean 50 and std 15 fall within [20, 80], so the filtered output is about 83% of input size.

**`def normalizeStage(records, minVal, maxVal):`**
Adds a "norm" field: `(value - minVal) / span` maps the value from [20, 80] to [0, 1]. `{**r, "norm": ...}` creates a new dict with all existing keys plus "norm". This is an O(N) operation with small constant: one arithmetic operation and one dict copy per record.

**`def featureStage(records, windowSize=10):`**
The performance hotspot. `window = deque(maxlen=windowSize)` creates a fixed-size sliding window. For each record, `window.append(r["norm"])` takes O(1). But `mean = sum(window) / len(window)` iterates over the entire window — up to 10 elements — for every record. For 10,000 records with window size 10, this is 100,000 additions just for the mean. The variance computation does another 10 additions per record. Total: ~200,000 additions.

**The fix (not shown in profiling example, but revealed by profiler):** Maintain a running sum: `runningSum += newValue - droppedValue`. The mean becomes `runningSum / len(window)` — two arithmetic operations instead of 10 per record. The profiler makes this inefficiency visible; without profiling, you would not know to look here.

**`pr = cProfile.Profile()`**
Creates a profiler instance. cProfile is implemented in C — faster than the pure-Python `profile` module — but still adds 2–10x runtime overhead during profiling. Run it offline, not in production.

**`pr.enable()` / `pr.disable()`**
Narrow the profiling window to only the code you care about. Code between `enable()` and `disable()` is measured. Setup code (generating test data, creating the profiler) is excluded. Keep the profiled section as tight as possible.

**`ps.sort_stats("cumulative")`**
Sorts the profiler output by cumulative time — the total time spent in a function plus all functions it calls. This shows which function represents the most expensive call chain. Alternative sort keys: `"tottime"` (time in the function itself, excluding callees) for identifying which function is intrinsically slow; `"calls"` for identifying which function is called most frequently.

**`ps.print_stats(10)`**
Prints the top 10 functions by the sort key. The output columns: `ncalls` (how many times called), `tottime` (time in this function excluding callees), `percall` (tottime / ncalls), `cumtime` (total including callees), `percall` (cumtime / ncalls), `filename:lineno(function)`. Focus on `cumtime` for identifying bottlenecks.

---
