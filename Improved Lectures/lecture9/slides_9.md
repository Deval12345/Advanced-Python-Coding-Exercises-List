# Slides — Lecture 9: Generators — Lazy Evaluation and Producer-Consumer Architecture

---

## Slide 1 — From Manual Iterators to Generators

**The Problem with Manual Iterators (L8 Recap)**

Writing a custom iterator requires:
- A full class with `__iter__` and `__next__`
- A state variable to track position
- Explicit `raise StopIteration`
- All state management done by hand

**The Generator Solution**

Any function with a `yield` statement is a generator function.
Python handles the entire iterator protocol automatically.

Key behavior:
- Calling a generator function returns a **generator object** (function body does NOT run yet)
- The body runs only when you begin iterating
- Generator objects implement both `__iter__` and `__next__` automatically

---

## Slide 2 — How yield Pauses and Resumes Execution

**Execution Flow**

1. You call the generator function → Python returns a generator object
2. You call `next()` (or enter a `for` loop) → function body begins running
3. Generator hits `yield someValue` → produces `someValue`, **suspends execution**
4. Local variables, position in code, all state is frozen in place
5. You call `next()` again → generator **resumes from the line after yield**
6. Function body finishes or hits `return` → `StopIteration` is raised automatically

**Compared to a Manual Iterator**

| Manual Iterator (L8) | Generator |
|---|---|
| Class with `__iter__` / `__next__` | Plain function with `yield` |
| State stored in `self` attributes | State stored in function's local scope |
| Explicit `raise StopIteration` | Automatic when function ends |
| Complex for non-trivial logic | Reads as natural sequential code |

---

## Slide 3 — Generator Expressions and Lazy Evaluation

**List Comprehension vs Generator Expression**

```
squares_list = [x * x for x in range(1_000_000)]   # computes ALL values immediately
squares_gen  = (x * x for x in range(1_000_000))   # computes NOTHING yet
```

Brackets → list (eager, all values in memory now)
Parentheses → generator (lazy, one value at a time on demand)

**Memory Comparison**

| Approach | 20 values | 20 million values |
|---|---|---|
| List comprehension | tiny | hundreds of MB |
| Generator expression | tiny | ~200 bytes (constant) |

**Lazy Evaluation Definition**

A value is not computed until the moment it is requested.
The generator holds only the current state, not all results.

---

## Slide 4 — Processing Large Data with Generators

**The Industrial Problem**

- Log file: 50 million lines → loading all into memory crashes the system
- Real-time sensor feed → data arrives continuously, cannot buffer everything
- Financial market events → millions per day, need kilobytes not gigabytes of RAM

**The Generator Solution**

Define a generator function that processes one record at a time:
- Receives the raw data source (file handle, list, API stream)
- Yields one parsed record per iteration
- Caller processes each record immediately

**Key Properties**

- Memory usage is bounded regardless of data size
- Producer (generator) and consumer (for loop) interleave execution
- Caller can stop early — generator pauses immediately, remaining values never computed

---

## Slide 5 — Producer-Consumer Architecture and Generator Pipelines

**Decoupling Producer from Consumer**

Producer: knows how to produce values (file, database, API, simulation)
Consumer: knows how to process values (aggregate, filter, write)
Neither knows the other's implementation → swap either side freely

**Three-Stage Pipeline**

```
readRecords(source)
    → filterValidRecords(records)
        → computeTotals(validRecords, currency)
```

Each stage:
- Receives a generator as input
- Yields processed values as output
- Processes one item at a time

**Pipeline Behavior**

- No stage waits for the previous to finish completely
- One value propagates through all stages per `next()` call
- Memory stays constant regardless of source size
- Each stage is independently testable with mock input

---

## Slide 6 — Lazy Evaluation in Industrial Systems

**Where This Pattern Appears at Scale**

**Django ORM**
- `Model.objects.filter(...)` builds a query plan, no SQL runs
- Query executes only when you iterate the QuerySet
- Chaining `.filter().exclude().order_by()` is free — still lazy

**Apache Spark**
- Each transformation (`map`, `filter`, `groupBy`) adds to a plan
- Plan executes only when you call an action (`collect`, `count`, `write`)
- Enables optimization across the entire pipeline before any data moves

**PyTorch DataLoader**
- Loads one batch at a time using an iterator
- Never holds the full dataset in GPU memory
- Training loop asks for the next batch; DataLoader produces it on demand

**The Unifying Principle**

Describe WHAT to compute. Execute only WHEN needed. Execute only AS MUCH as needed.

---

## Slide 7 — When to Use Generators vs Lists

**Use a Generator When**

- Processing data sequentially (one pass)
- Data is large or potentially infinite
- Memory efficiency is important
- Building a pipeline with multiple stages
- Data arrives as a stream (files, network, sensors)

**Use a List When**

- You need random access by index
- You need to iterate over the data more than once
- You need to know the length upfront
- You need to sort, search, or reverse the data
- The dataset is small and fixed

**The Practical Rule**

Start with a generator for sequential processing.
Convert to a list only when you discover you need list capabilities.

---

## Slide 8 — Summary and What's Next

**What Generators Give You**

- Full iterator protocol with no boilerplate class
- Lazy evaluation: values computed only when requested
- Constant memory regardless of data volume
- Natural sequential code that reads like a loop
- Composable pipeline stages via generator chaining

**The Consequence for Real Systems**

Data engineering pipelines, streaming processors, lazy query builders, and batch ML training all use this exact architecture — producer yields, consumer iterates, stages are decoupled.

**What's Next: Decorators**

Decorators use the closure and first-class function concepts from L6-L7 to inject behavior into functions non-invasively. A decorator wraps a function, adds behavior before and after it, and returns a new function — all without modifying the original function's code. This is how Python implements logging, authentication, caching, and timing in production systems.
