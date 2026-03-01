# Slides — Lecture 33: Big Project Stage 1 — Pluggable Analytics Engine

---

## Slide 1 — Lecture Overview

**From Individual Tools to Integrated System**

- This course has built individual tools in isolation — now we integrate all of them
- The Pluggable Analytics Engine: a real streaming data processing system
- Uses data model, protocols, descriptors, generators, decorators, context managers, concurrency
- Architecture goal: every component independently testable, replaceable, and understandable
- Today: Source, Pipeline Stage, and Sink protocols — the system's three extension points

---

## Slide 2 — Why Architecture Matters

**The maintenance wall**

- A flat script works at small scale — every change touches every part at large scale
- Adding a new data source → must modify ingestion loop
- Adding a new transform → must modify processing code
- Adding a new output → must add a branch in the innermost loop
- Architecture solution: define clear component interfaces; let components be independently developed

---

## Slide 3 — The Three-Protocol Architecture

**Source → Pipeline Stages → Sink**

- **Source**: any class with a `stream()` generator method that yields records
- **Pipeline Stage**: any class with a `process(inputStream)` generator method
- **Sink**: any class with a `consume(stream)` method
- No inheritance required; duck typing enforced by clear documentation
- New component = new class implementing the protocol; no existing code changes

---

## Slide 4 — The Source Protocol: Streaming Without Loading

**Why generators are the right tool**

- Naive approach: load all data into memory → impossible for infinite live streams
- Generator approach: yield one record at a time → constant memory regardless of dataset size
- `CsvFileSource.stream()`: opens file, yields one dict per CSV row, closes on iteration end
- `SyntheticSource.stream()`: yields artificial readings indefinitely for testing
- Both implement identical interface — pipeline cannot tell them apart

---

## Slide 5 — Generator Composition: Lazy Pipelines

**How pipeline stages chain without materializing data**

- Each stage wraps the input stream in its own generator
- `source.stream()` → ThresholdFilter.process → NormalizeTransform.process → MovingAverageTransform.process
- Result: a single composed generator; no intermediate lists; constant memory regardless of pipeline depth
- `buildPipeline(source, stages)` formalizes this composition: loop assigns each stage's output as the next stage's input
- Scales to 100 stages as easily as 3

---

## Slide 6 — ThresholdFilter: First Defense Against Bad Data

**Eliminating outliers at the pipeline boundary**

- Sensor hardware faults, calibration errors, and transmission noise produce invalid readings
- ThresholdFilter yields only records where `minVal ≤ value ≤ maxVal`; silently drops outliers
- Early filtering saves computation in all downstream stages
- Production rule: validate data as close to the source as possible; downstream logic should not handle invalid data
- Configurable bounds: different sensors have different valid ranges

---

## Slide 7 — NormalizeTransform and MovingAverageTransform

**Making heterogeneous data comparable**

- **Normalize**: scale values to [0, 1] using configured min/max → readings from different sensors can be compared directly
- **MovingAverage**: maintain a `deque(maxlen=N)` window → yields the rolling mean alongside each record
- Both use `{**record, "value": new_value}` to spread the existing record and override one key
- State (the deque window) lives in the stage instance — the generator is stateful by design
- This pattern works because generators execute lazily in the caller's thread — no concurrency needed yet

---

## Slide 8 — The Sink Protocol: Flexible Output

**Separating "where results go" from "how they are computed"**

- Output destinations should be as pluggable as data sources
- `ConsoleSink.consume(stream)`: prints records — for debugging
- `FileSink.consume(stream)`: writes JSON lines to a file — for persistence
- `ThresholdAlertSink.consume(stream)`: watches values, triggers callbacks on threshold breach
- All three implement `consume(stream)` — the coordinator never knows which it's calling

---

## Slide 9 — The ETL Pattern

**Source → Transform → Load — universal data engineering**

- Extract (Source), Transform (Pipeline Stages), Load (Sink) = ETL
- The fundamental pattern in every data engineering system
- Apache Spark, AWS Glue, Apache Flink, Kafka Streams all implement variants of ETL
- Pure Python implementation scales to millions of records per day on a single machine
- Understanding this pattern at the protocol level makes all framework documentation readable

---

## Slide 10 — What Gets Added in Each Project Stage

**Building up the complete system**

- Stage 1 (today): Core protocols — Source, Pipeline Stage, Sink
- Stage 2: Production hardening — context managers, descriptors, retry decorators
- Stage 3: Concurrency — async ingestion, thread pools for sync libraries
- Stage 4: Measurement — profiling, latency metrics, memory tracking
- Stage 5: Caching — memoization for expensive computations, TTL-based invalidation
- Stages 6–8: Resilience, observability, advanced internals

---

## Slide 11 — Design Principle: Composition Over Inheritance

**Why protocols beat class hierarchies here**

- Inheritance-based design: `class CsvFileSource(AbstractSource)` → tight coupling to the framework
- Protocol-based design: `class CsvFileSource:` with a `stream()` method → zero coupling
- A Kafka client, a database cursor, an API paginator can all be Sources with zero code changes
- Python's duck typing: "if it has a stream method, it is a Source"
- Optional runtime check: `isinstance(source, Iterable)` or a custom `@runtime_checkable Protocol`

---

## Slide 12 — Lecture 33 Key Principles

**The foundation of the Pluggable Analytics Engine**

- Source protocol: `stream()` generator → constant memory for any dataset size
- Pipeline Stage: `process(inputStream)` generator → composable, lazy transformations
- Sink protocol: `consume(stream)` → pluggable output destinations
- `buildPipeline(source, stages)` composes all stages into one lazy generator chain
- No inheritance, no framework dependency, no global state
- Next lecture: resource lifecycle (context managers), validated config (descriptors), latency measurement (decorators)

---
