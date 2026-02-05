
# When Speed Meets Structure — A Real System Example

> Speed without structure collapses  
> Structure without speed stagnates  
> Real systems need both

This document demonstrates a real engineering situation where:

- performance optimization alone fails
- architecture alone fails
- combining both creates a scalable design

We simulate a production analytics pipeline.

---

## Real-World Scenario: Log Analytics Backend

A company processes millions of log events per minute:

- real-time anomaly detection
- dashboard aggregation
- alert scoring

The system must ingest, transform, score, and store data continuously.

We build it in 3 stages:

1. Fast but messy  
2. Clean but slow  
3. Fast AND structured

---

# Stage 1: Speed Without Structure

Optimized for speed, but fragile.

```python
import multiprocessing as mp

def pipeline(data):
    results = []
    for x in data:
        y = x * 2
        if y % 7 == 0:
            score = y ** 2
        else:
            score = y + 5
        results.append(score)
    return results

if __name__ == "__main__":
    data = list(range(1_000_000))
    chunks = [data[i:i+100_000] for i in range(0, len(data), 100_000)]
    with mp.Pool() as pool:
        output = pool.map(pipeline, chunks)
```

Fast — but impossible to extend safely.

---

# Stage 2: Structure Without Speed

Clean architecture, poor performance.

```python
class Transformer:
    def apply(self, x):
        return x * 2

class Scorer:
    def score(self, x):
        if x % 7 == 0:
            return x ** 2
        return x + 5

class Pipeline:
    def __init__(self, transformer, scorer):
        self.transformer = transformer
        self.scorer = scorer

    def run(self, data):
        out = []
        for x in data:
            y = self.transformer.apply(x)
            out.append(self.scorer.score(y))
        return out

pipeline = Pipeline(Transformer(), Scorer())
pipeline.run(range(1_000_000))
```

Beautiful structure — unusable throughput.

---

# Stage 3: Structure + Speed

Structured AND parallel.

```python
import multiprocessing as mp

class Transformer:
    def apply(self, x):
        return x * 2

class Scorer:
    def score(self, x):
        if x % 7 == 0:
            return x ** 2
        return x + 5

class Worker:
    def __init__(self):
        self.transformer = Transformer()
        self.scorer = Scorer()

    def process(self, chunk):
        out = []
        for x in chunk:
            y = self.transformer.apply(x)
            out.append(self.scorer.score(y))
        return out

def run_worker(chunk):
    w = Worker()
    return w.process(chunk)

if __name__ == "__main__":
    data = list(range(1_000_000))
    chunks = [data[i:i+100_000] for i in range(0, len(data), 100_000)]

    with mp.Pool() as pool:
        result = pool.map(run_worker, chunks)
```

Now we get:

✔ modular components  
✔ multiprocessing speed  
✔ extensibility  
✔ safe scaling  

---

## Engineering Insight

Architecture enables scale.  
Performance enables survival.

High-performance systems require both.

Never choose only one.
