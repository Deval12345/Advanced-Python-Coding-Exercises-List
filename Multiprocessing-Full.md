# Multiprocessing in Real Systems — Practical Teaching Notes

Multiprocessing is not just about speed.

It is about **system design**.

You are building a group of independent workers that cooperate safely.

Good multiprocessing problems share 3 traits:

- tasks are independent
- computation is heavy
- communication is minimal

When these are true → multiprocessing scales beautifully.

---

## Image Processing as a Parallel Problem

Image processing is embarrassingly parallel.

Each pixel can be processed independently.

### Real-world scenario

A photo editing app applies a heavy artistic filter to a 4K image.

Sequential processing freezes the interface.

Users think the app crashed.

We split the image into rows and process them in parallel.

```python
from multiprocessing import Pool
import numpy as np

def artistic_filter(pixel):
    x = pixel
    for _ in range(2000):
        x = (x * x + 7) % 255
    return x

def process_row(row):
    return [artistic_filter(p) for p in row]

if __name__ == "__main__":
    image = np.random.randint(0, 255, (1200, 1200))
    with Pool() as pool:
        result = pool.map(process_row, image)
    result = np.array(result)
```

Architecture insight:

Image rows become natural work units.

No communication needed between workers.

Perfect scaling.

---

## Randomness Pitfall in Parallel Systems

Parallel simulations can silently break if workers share the same random seed.

### Real-world failure

A Monte Carlo financial risk engine reused seeds.

All simulation paths were identical.

The company trusted fake risk estimates.

### Rule

Each worker must have its own random stream.

```python
import random
import os
from multiprocessing import Pool

def simulate(_):
    random.seed(os.getpid())
    return random.random()

if __name__ == "__main__":
    with Pool(4) as pool:
        print(pool.map(simulate, range(4)))
```

---

## Shared Memory for Large NumPy Data

Copying giant arrays kills performance.

Memory bandwidth becomes the bottleneck.

### Real-world scenario

Satellite imagery pipeline processes gigabyte-scale matrices.

Copying per process is impossible.

Shared memory solves this.

```python
import numpy as np
from multiprocessing import Process, shared_memory

def worker(name, shape):
    shm = shared_memory.SharedMemory(name=name)
    arr = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)
    arr += 1
    shm.close()

if __name__ == "__main__":
    data = np.ones((1000, 1000))
    shm = shared_memory.SharedMemory(create=True, size=data.nbytes)
    shared = np.ndarray(data.shape, dtype=data.dtype, buffer=shm.buf)
    shared[:] = data[:]

    p = Process(target=worker, args=(shm.name, data.shape))
    p.start()
    p.join()

    shm.close()
    shm.unlink()
```

---

## Queue-Based Worker Architecture

Producer-consumer architecture.

Main process produces tasks.

Workers consume tasks.

They never talk to each other.

### Real-world use

Analytics pipelines  
job schedulers  
background processing systems

```python
from multiprocessing import Process, Queue
import math

def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

def worker(tasks, results):
    while not tasks.empty():
        n = tasks.get()
        results.put((n, is_prime(n)))

if __name__ == "__main__":
    tasks = Queue()
    results = Queue()

    for n in range(100000, 100100):
        tasks.put(n)

    workers = [Process(target=worker, args=(tasks, results)) for _ in range(4)]

    for w in workers: w.start()
    for w in workers: w.join()

    while not results.empty():
        print(results.get())
```

---

## Pools — High-Level Interface

Pool hides process management.

Most production multiprocessing uses Pool.

```python
from multiprocessing import Pool

def clean_row(row):
    return row.strip().lower()

if __name__ == "__main__":
    rows = [" Data ", " Science ", " Python "]

    with Pool(3) as pool:
        cleaned = pool.map(clean_row, rows)

    print(cleaned)
```

---

## Synchronization & Shared State

Race conditions corrupt data.

Two workers update shared memory simultaneously.

Result becomes unpredictable.

### Failure example

```python
from multiprocessing import Process, Value

def worker(counter):
    for _ in range(100000):
        counter.value += 1
```

Incorrect result.

### Lock fix

```python
from multiprocessing import Process, Value, Lock

def worker(counter, lock):
    for _ in range(100000):
        with lock:
            counter.value += 1
```

### Real-world scenario

Distributed metrics collector.

Dashboards must be correct.

Locks protect shared totals.

---

## Safe File Writing in Multiprocessing

Disk is a shared resource.

Multiple writers corrupt files.

### Production rule

One writer at a time.

### Lock version

```python
from multiprocessing import Process, Lock

def worker(lock, i):
    with lock:
        with open("log.txt", "a") as f:
            f.write(f"Worker {i} finished\n")
```

### Production pattern: dedicated writer

```python
from multiprocessing import Process, Queue

def writer(queue):
    with open("log.txt", "a") as f:
        while True:
            msg = queue.get()
            if msg == "STOP":
                break
            f.write(msg + "\n")
```

Workers send messages → writer owns disk.

---

## Debugging Multiprocessing

Parallel bugs are timing-dependent.

They appear randomly.

### Make bugs loud

```python
import os
print("Process", os.getpid())
```

### Stress test

Run program repeatedly.

Failures appear under load.

### Deadlock rule

Always acquire locks in the same order.

Never mix ordering.

### Professional workflow

- run single-process first
- add logging
- remove shared state
- replay queues
- stress test
- simplify architecture

Multiprocessing debugging = debugging a distributed system.

---

## Final Design Guidelines

- prefer independent tasks
- minimize shared state
- protect critical sections
- centralize IO
- test under stress
- design architecture first, speed second

Multiprocessing is not magic.

It is disciplined engineering.
