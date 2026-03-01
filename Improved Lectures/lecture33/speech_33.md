# Lecture 33: Big Project Stage 1 — Pluggable Analytics Engine
## speech_33.md — Instructor Spoken Narrative

---

We have spent this entire course building individual tools — each one powerful, each one focused. You have learned to write classes that feel native to Python using the data model. You have built protocols that let your code be extended without modification. You have written descriptors that enforce business rules at the attribute level. You have crafted decorators that add behaviors without changing functions. You have built generators that process infinite streams with constant memory. You have mastered async for high-concurrency I/O and process pools for CPU-bound parallelism.

Now comes the moment when all of those individual tools need to work together.

Today we begin building the Pluggable Analytics Engine — a streaming data processing system that incorporates every major concept from this course into a single coherent architecture. I want to be clear about why we are doing this. It is not just to show that all the pieces connect. It is to demonstrate that good Python architecture makes large systems manageable. Every design decision we make today will make some future change trivially easy — because we will design for extension from the start.

Let me describe what this system does. It reads sensor data from various sources — temperature sensors, pressure gauges, flow meters. It passes those readings through a configurable pipeline of transformations: filter out invalid values, normalize readings to a common scale, compute moving averages. Then it emits the processed data to output destinations: a file, a monitoring dashboard, an alert system. The entire thing runs as a continuous stream — records flow through one at a time, memory consumption stays constant regardless of dataset size.

Here is the most important architectural decision of the whole project: every major component is defined by a protocol, not by a base class.

What does that mean in practice? A Source is any class with a stream method that yields records. A Pipeline Stage is any class with a process method that accepts an input stream and yields a transformed stream. A Sink is any class with a consume method that accepts a stream and reads from it. That is the entire contract. Any class that satisfies these method signatures works in the system. No inheritance required; no registration required. This is Python's duck typing applied systematically to system design.

Why does this matter? Because it makes every component independently pluggable. A new data source — say, a Kafka consumer — is a class with a stream method. It drops in without any changes to the pipeline code. A new transformation — say, an anomaly detector using a machine learning model — is a class with a process method. It slots into the pipeline at any position. A new output — say, a webhook that sends alerts to Slack — is a class with a consume method. The pipeline never knows or cares what the specific implementation is.

Let us trace through the first concrete piece: the Source protocol. The problem it solves is fundamental. A naive system reads all data into memory first, then processes it. For a live sensor feed, there is no "all of the data" — the stream is potentially infinite. For a large historical dataset, loading everything means memory consumption grows proportionally with dataset size — which means expensive infrastructure scaling rather than efficient engineering.

The generator protocol is Python's answer. Our CsvFileSource opens a file and yields one record at a time using Python's csv.DictReader. The file stays open only as long as the stream is being consumed — no memory spike regardless of file size. Our SyntheticSource generates artificial readings on demand for testing. Both classes implement the same stream interface. The pipeline cannot tell them apart, and it does not need to.

Example 33.1 shows both sources. Look at how the generators are structured: they yield one record at a time, and the caller drives the iteration pace. The pipeline processes each record and immediately discards it — there is no accumulation.

Now the pipeline stages. These are the transformation steps that records pass through. Each stage has one job: accept a stream, transform each record, yield the result. ThresholdFilter drops records outside a valid range — this is your first line of defense against sensor noise and hardware faults. NormalizeTransform scales values to a common zero-to-one range so that readings from different sensors — measured in different units at different scales — can be compared meaningfully. MovingAverageTransform adds a rolling average over the last N records, which smooths out transient spikes and reveals underlying trends.

The crucial observation in Example 33.2 is the buildPipeline function. It takes a source and a list of stages. It calls source.stream() to get the initial generator, then for each stage it wraps that generator with stage.process, which is itself a generator. The output is a single composed generator. No record is processed until you iterate over the result. No intermediate list exists. The entire multi-stage pipeline runs with constant memory because Python generators are lazy — they produce values only when demanded.

This is generator composition. It scales to a hundred stages just as easily as three. Each stage adds one layer of lazy wrapping. The memory cost is the stack of generator frames, not the data itself.

The Sink protocol completes the trio. Where data goes after processing should be as configurable as where it comes from. A ConsoleSink prints records for debugging. A FileSink writes to a log file. A ThresholdAlertSink watches values and triggers alerts when thresholds are crossed. All three implement the same consume method. The pipeline coordinator calls consume on whatever sinks are configured — it does not know whether it is writing to a file or sending alerts.

This three-protocol architecture — Source, Pipeline Stage, Sink — is not unique to Python. You will recognize it as the ETL pattern: Extract, Transform, Load. It is the fundamental building block of every data engineering system from small Python scripts to Apache Spark running on thousands of machines. The principle scales because the protocol is simple.

In the next lecture we add production hardening: context managers to ensure every resource is properly released even when exceptions occur, descriptors to validate configuration values at assignment time, and decorators to measure latency and retry on transient failures. The architecture stays the same; the reliability improves dramatically.

