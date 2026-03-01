# Speech Source — Lecture 33: Big Project Stage 1 — Pluggable Analytics Engine

---

## CONCEPT 0.1 — Transition: From Individual Tools to Integrated System

Throughout this course we have built individual tools with precision: the Python data model, protocols and duck typing, descriptors for validated fields, closures and decorators, context managers, generators for memory-efficient iteration, the async event loop, thread pools, and process pools. Each of these was studied in isolation — a single concept, a clear example, a contained exercise. That is the right way to learn. But it is not the way real software works.

Real software is the intersection of all of these. A production data pipeline uses generators to stream records without loading everything into memory, context managers to ensure file handles and database connections are released, decorators to measure latency and retry on failure, descriptors to validate configuration values, protocols to allow new data sources to be plugged in without changing existing code, async to handle many concurrent network connections, and process pools to run CPU-bound transformations in parallel. None of these concepts is optional. Each solves a specific problem that arises at a specific scale.

Today we begin building a project that uses all of them — together, in a real architecture. We call it the Pluggable Analytics Engine. It is a streaming data processing system that ingests sensor readings from multiple sources, filters and transforms them through a configurable pipeline, computes aggregate statistics, and emits alerts when thresholds are crossed. By the end of the project arc, it will be a system that could actually run in production at a small scale.

---

## CONCEPT 1.1 — Project Architecture Overview

**Problem it solves:**
Any large software project risks becoming a pile of code — functions calling functions, mutable global state, configuration scattered through the codebase, no clear boundaries between responsibilities. The project becomes unmaintainable as it grows. The solution is architecture: deliberate decisions about which pieces are independent, what their interfaces are, and how they communicate.

**Why invented:**
Software architecture is not about following patterns for their own sake. It is about making every component independently testable, independently replaceable, and independently understandable. An architecture that uses protocols instead of inheritance for extension means a new data source can be added by writing one class, with no changes to any existing code. An architecture that uses context managers for resource lifecycle means that adding a new resource type never risks resource leaks — the protocol enforces cleanup by design.

**What happens without it:**
A flat script that processes data works fine at small scale. As requirements grow — new data sources, new transformation steps, new output formats — every change touches every part of the code. Adding a new data source requires modifying the ingestion loop. Adding a new transformation requires modifying the pipeline runner. Testing requires running the whole system. This is the maintenance wall that architecture prevents.

**Industry impact:**
The architecture we choose consists of four layers: Source (where data comes from), Pipeline (how data is transformed), Sink (where results go), and Coordinator (the top-level orchestration). Each layer is defined by a protocol — an informal interface that any class can implement without inheriting from a base class. This is Python's approach to extensibility: duck typing enforced by clear documentation and optional runtime checks through ABCs.

---

## CONCEPT 1.2 — The Source Protocol: Streaming Data Without Loading Everything

**Problem it solves:**
A naive analytics system reads all data into memory, processes it, and writes results. This works for a dataset of a thousand records. For a million records arriving continuously from a sensor network, there is no fixed "all of the data" — the stream is infinite or near-infinite. The system must process records as they arrive, maintaining only as much state as the computation requires.

**Why invented:**
Python's generator protocol — an object implementing the iterator protocol using yield — is the natural model for streaming data sources. A generator produces one record at a time, on demand, without holding all records in memory. Wrapping it in a class with a well-defined interface — the Source protocol — means any data origin (file, database, network socket, message queue) can be used interchangeably by the pipeline.

**What happens without it:**
Without a streaming interface, the system loads the entire dataset before processing begins. For real-time sensor data this is simply impossible — you cannot buffer infinity. For large batch datasets it means memory consumption proportional to dataset size rather than constant, which limits the system's capacity and forces expensive infrastructure scaling.

**Industry impact:**
Kafka consumers, database cursors, file readers, and API paginator clients all implement this streaming pattern. The Source protocol is the pipeline's entry point. By defining it as a protocol (an informal interface with expected method signatures), we allow any of these to be used without coupling the pipeline to a specific data source class.

---

## EXAMPLE 1.1 — Source Protocol: Base interface and first implementations

Narration: We define the Source protocol as a class with one required method: a generator called stream that yields records one at a time. We implement two concrete sources: CsvFileSource reads records from a CSV file using Python's csv module, yielding one dictionary per row — a memory-constant operation regardless of file size. SyntheticSource generates artificial sensor readings for testing — it yields records indefinitely, simulating a live sensor feed. Both classes implement the same generator interface; the pipeline can use either without knowing which it has.

```python
# Example 33.1
import csv                                             # line 1
import random                                          # line 2
import time                                            # line 3
from pathlib import Path                               # line 4

class CsvFileSource:                                   # line 5
    """Source protocol: read sensor records from a CSV file."""  # line 6
    def __init__(self, filePath, sensorId="unknown"):  # line 7
        self.filePath = Path(filePath)                 # line 8
        self.sensorId = sensorId                       # line 9

    def stream(self):                                  # line 10  generator method
        with open(self.filePath, newline="") as f:     # line 11
            reader = csv.DictReader(f)                 # line 12
            for row in reader:                         # line 13
                yield {                                # line 14
                    "sensorId": self.sensorId,         # line 15
                    "timestamp": float(row["timestamp"]),  # line 16
                    "value": float(row["value"]),       # line 17
                    "unit": row.get("unit", "unknown") # line 18
                }                                      # line 19

class SyntheticSource:                                 # line 20
    """Source protocol: generate synthetic sensor readings for testing."""  # line 21
    def __init__(self, sensorId, meanValue=50.0, stddev=10.0, numRecords=None):  # line 22
        self.sensorId = sensorId                       # line 23
        self.meanValue = meanValue                     # line 24
        self.stddev = stddev                           # line 25
        self.numRecords = numRecords                   # line 26  None = infinite

    def stream(self):                                  # line 27
        count = 0                                      # line 28
        while self.numRecords is None or count < self.numRecords:  # line 29
            yield {                                    # line 30
                "sensorId": self.sensorId,             # line 31
                "timestamp": time.time(),              # line 32
                "value": random.gauss(self.meanValue, self.stddev),  # line 33
                "unit": "celsius"                      # line 34
            }                                          # line 35
            count += 1                                 # line 36
            time.sleep(0.001)                          # line 37  simulate sensor polling rate

if __name__ == "__main__":                             # line 38
    print("--- CsvFileSource demonstration (first 3 records) ---")  # line 39
    # For demonstration, we use SyntheticSource to generate test data  # line 40
    synth = SyntheticSource(sensorId="TEMP_01", numRecords=5)  # line 41
    for record in synth.stream():                      # line 42
        print(f"  {record['sensorId']}: {record['value']:.2f}{record['unit']} at {record['timestamp']:.3f}")  # line 43
    print("Source protocol demonstration complete")    # line 44
```

---

## CONCEPT 2.1 — The Pipeline Stage Protocol: Composable Transformations

**Problem it solves:**
A real analytics pipeline is not a single transformation — it is a sequence of steps: filter outliers, normalize values, compute moving averages, apply business rules, enrich with metadata. If each step is hardcoded into a single function, changing one step requires modifying and retesting the whole pipeline. Adding a new step requires understanding the entire transformation sequence.

**Why invented:**
The pipeline stage protocol makes each transformation a self-contained component with a standard interface: a method that accepts a stream of records and yields a transformed stream. These stages can be chained: the output generator of stage one becomes the input generator of stage two, all lazily. The entire pipeline runs without materializing intermediate results — constant memory regardless of pipeline depth.

**What happens without it:**
Without a composable stage protocol, adding a new transformation step means either modifying the main processing loop or creating a new top-level function that receives the full dataset. Neither scales. With the protocol, a new stage is a new class implementing one method — and it composes with every other stage automatically.

**Industry impact:**
This is the transformation pipeline pattern used by Apache Beam, Apache Flink, pandas pipeline chaining, and every modern stream processing framework. Python's lazy generator composition makes it implementable without any framework — pure standard library.

---

## EXAMPLE 2.1 — Pipeline stages: ThresholdFilter, NormalizeTransform, and MovingAverageTransform

Narration: We implement three pipeline stages. ThresholdFilter accepts a minimum and maximum value and yields only records within that range, dropping outliers silently. NormalizeTransform scales values to a zero-to-one range using min and max bounds provided at construction time. MovingAverageTransform maintains a sliding window of the last N values and adds a rolling average to each record without modifying the original value. Each stage is a generator that consumes an input stream and produces an output stream. We compose them by nesting the stream calls, and we count how many records survive the filter to verify correctness.

```python
# Example 33.2
from collections import deque                         # line 1

class ThresholdFilter:                                 # line 2
    """Pipeline stage: drop records outside [minVal, maxVal]."""  # line 3
    def __init__(self, minVal, maxVal):                # line 4
        self.minVal = minVal                           # line 5
        self.maxVal = maxVal                           # line 6

    def process(self, inputStream):                    # line 7  generator method
        for record in inputStream:                     # line 8
            if self.minVal <= record["value"] <= self.maxVal:  # line 9
                yield record                           # line 10
            # else: silently drop the record

class NormalizeTransform:                              # line 11
    """Pipeline stage: scale values to [0, 1] range."""  # line 12
    def __init__(self, minVal, maxVal):                # line 13
        self.minVal = minVal                           # line 14
        self.range = maxVal - minVal                   # line 15

    def process(self, inputStream):                    # line 16
        for record in inputStream:                     # line 17
            normalized = (record["value"] - self.minVal) / self.range  # line 18
            yield {**record, "value": round(normalized, 6), "unit": "normalized"}  # line 19

class MovingAverageTransform:                          # line 20
    """Pipeline stage: add rolling average over last N records."""  # line 21
    def __init__(self, windowSize):                    # line 22
        self.windowSize = windowSize                   # line 23
        self.window = deque(maxlen=windowSize)         # line 24

    def process(self, inputStream):                    # line 25
        for record in inputStream:                     # line 26
            self.window.append(record["value"])        # line 27
            rollingAvg = sum(self.window) / len(self.window)  # line 28
            yield {**record, "rollingAvg": round(rollingAvg, 6)}  # line 29

def buildPipeline(source, stages):                     # line 30
    """Compose a source and a list of stages into a single lazy generator."""  # line 31
    stream = source.stream()                           # line 32
    for stage in stages:                               # line 33
        stream = stage.process(stream)                 # line 34
    return stream                                      # line 35

if __name__ == "__main__":                             # line 36
    source = SyntheticSource(sensorId="TEMP_01", numRecords=20)  # line 37
    stages = [                                         # line 38
        ThresholdFilter(minVal=30.0, maxVal=70.0),     # line 39
        NormalizeTransform(minVal=30.0, maxVal=70.0),  # line 40
        MovingAverageTransform(windowSize=5),          # line 41
    ]                                                  # line 42
    pipeline = buildPipeline(source, stages)           # line 43
    count = 0                                          # line 44
    for record in pipeline:                            # line 45
        count += 1                                     # line 46
        print(f"Record {count}: sensor={record['sensorId']}, value={record['value']:.4f}, rolling={record.get('rollingAvg', 'N/A')}")  # line 47
    print(f"Pipeline processed {count} records (after filtering)")  # line 48
```

---

## CONCEPT 3.1 — The Sink Protocol: Flexible Output

**Problem it solves:**
Where data goes after processing should be as pluggable as where it comes from. An analytics engine might write to a database, emit to a message queue, write to a file, or display on a dashboard — or all four simultaneously. Hardcoding the output destination into the pipeline means changing one output format requires touching the pipeline code.

**Why invented:**
The Sink protocol mirrors the Source: a class with one method that accepts a stream of records and consumes them. By keeping the Sink interface identical for all output types, the pipeline can be configured to output to any combination of sinks without changing pipeline code.

**What happens without it:**
Without a sink protocol, output is hardcoded in the pipeline's innermost loop. Adding a second output destination requires adding a branch inside the processing code. Adding a third requires another branch. Eventually the pipeline code becomes an unmaintainable tangle of output-specific logic mixed with transformation logic.

**Industry impact:**
This pattern — source, pipeline stages, sink — is called the ETL pattern: Extract, Transform, Load. It is the fundamental building block of every data engineering system. The implementation here is minimal and pure Python, but the architectural principle scales directly to Apache Spark, AWS Glue, and every enterprise data platform.

---

## CONCEPT 4.1 — Final Takeaway Lecture 33

Today we established the project's foundation: the Source protocol for streaming data ingestion without loading everything into memory, the Pipeline Stage protocol for composable lazy transformations, and the Sink protocol for flexible output. These three protocols together define the Pluggable Analytics Engine's extension boundary — everything inside is infrastructure; everything outside is a plugin. A new data source is a new class with a stream method. A new transformation is a new class with a process method. A new output destination is a new class with a consume method. None of these additions require changes to any existing code.

In the next lecture we add the infrastructure layer: resource lifecycle management using context managers, validated configuration using descriptors, and metrics collection using decorators — the production hardening that separates a working pipeline from a reliable one.
