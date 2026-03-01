# Speech — Lecture 34: Big Project Stage 2 — Resource Lifecycle Architecture

---

Last time, we built the skeleton. We defined a Source protocol with a `stream()` generator, a Stage protocol with a `process()` generator, and a Sink with a `consume()` method. We composed them into a lazy pipeline that read sensor data, filtered it, transformed it, and sent it somewhere. It was clean, elegant, and completely unfit for production use — because we never thought about what happens when things go wrong.

Today we think about that.

Here is the scenario. Your pipeline is running. It has opened a database connection to fetch calibration data. It has opened a file handle to log output. It has a network socket to a sensor array. Everything is going beautifully — 50,000 records processed, 51,000, and then at record 52,417, the sensor array drops offline. An exception propagates up the call stack. Your pipeline crashes.

What happened to that database connection? What happened to that file handle? What happened to that socket? If you have no cleanup mechanism, the answer is: they are still open. The operating system is holding them for you, faithfully, waiting for you to close them. In a system that restarts automatically and runs continuously, you now have ghost connections accumulating. Run this long enough, and the database connection pool is exhausted. New connections are refused. Your entire service degrades — not because of the sensor failure, which was transient, but because of resource leakage, which is permanent until you restart the whole system.

This is not a hypothetical. This is Tuesday in production.

The solution is Python's context manager protocol, and it is one of the most important patterns in the entire language. The idea is deceptively simple: give the pipeline a structured entry point and a guaranteed exit point. When the `with` block is entered, `__enter__` runs — you open connections, start timers, initialize state. When the `with` block exits — for any reason, including an exception — `__exit__` runs. No matter what. You cannot forget to call it. You cannot skip it. The language guarantees it.

Our `PipelineContext` class is that structure. It wraps the entire pipeline run — source, stages, and sink — inside a managed block. It records when we started, how many records we processed, and whether we exited cleanly or crashed. The `__exit__` method receives the exception type, value, and traceback if something went wrong — and it returns `False`, which is crucial. `False` means "I have noted the exception, I have done my cleanup, but I am not suppressing it — let it propagate to the caller." We log what happened, we release what we held, and we get out of the way.

Notice that `run()` is the method where the lazy pipeline chain gets executed. The source produces a generator. Each stage wraps that generator in another generator. The final `for` loop is what actually pulls records through — nothing moves until that loop runs. We increment `_recordCount` on every record before passing it to the sink. This means if the sink fails on record 1,000, we know exactly how many records made it through.

Now, once we have a working pipeline, we need to configure it. And configuration without validation is a bug waiting to happen.

Consider this: what is the valid range for a sampling rate? It must be positive. It must be a number. What is the valid range for a window size? Probably between 1 and some reasonable maximum. What happens if someone passes `samplingRateHz = -5`? Without validation, nothing happens at construction time. The value gets stored. Later, some computation divides by the sampling rate, produces a negative number, and the pipeline outputs nonsense. Or it crashes with a `ZeroDivisionError` when the window size is zero. The stack trace points at the computation, not at the configuration, and you spend an hour wondering why your statistics code is broken when the problem was the value you passed in twenty lines ago.

Python's descriptor protocol solves this at the architecture level. A descriptor is an object that lives as a class attribute and intercepts attribute access on instances of that class. When you write `self.samplingRateHz = value`, Python checks whether the class has a descriptor at `samplingRateHz` — and if it does, it calls that descriptor's `__set__` method instead of storing the value directly.

This is how `property` works internally. But `property` is defined per-attribute, inline. Descriptors are classes — reusable, composable, nameable. We define `PositiveFloat` once and apply it to `samplingRateHz`, `filterMin`, and `filterMax`. We define `BoundedInt` once with a configurable range and apply it to `windowSize`. Every one of those attributes now validates at assignment time, with a clear, specific error message that tells you exactly what was wrong.

`__set_name__` is a Python 3.6 addition that makes descriptors ergonomic. It is called automatically when the descriptor is assigned as a class body attribute — so the descriptor knows its own name without you having to pass it explicitly. We use that name to store the actual value under a private mangled name on the instance, which avoids the infinite recursion you would get if the descriptor tried to store the value under the same name it intercepts.

The effect is profound. `config.windowSize = 0` raises a `ValueError` immediately, with the message "windowSize must be in [1, 10000]". `config.samplingRateHz = "fast"` raises a `TypeError` immediately, with the message "samplingRateHz must be numeric, got str". These errors happen at the boundary where invalid data enters the system — not deep inside the pipeline where the symptom eventually surfaces.

Now we come to metrics — and this is where the decorator pattern earns its place in production engineering.

Without measurement, you are flying blind. Your pipeline processes records — but how fast? Which stage is slowest? Is the filter passing through 90% of records or 10%? Where is the memory pressure coming from? Without answers to these questions, performance optimization is guesswork. You might spend a week improving an algorithm that contributes 5% of total runtime while the actual bottleneck — maybe a database query inside a normalization stage — sits completely untouched.

A `@measuredStage` decorator solves this without touching the stage itself. The decorator wraps the `process()` generator method. When the wrapper runs, it initializes a metrics object, starts a timer, and begins iterating through the inner generator. Each time a record comes through — each `yield` — it records the time. When the generator is exhausted, it calculates total time, throughput, and mean per-record latency, and prints or stores the result.

The key insight is that the stage being measured does not change at all. `MeasuredThresholdFilter` applies `@measuredStage("ThresholdFilter")` to its `process` method and the rest of the class is identical to the unmeasured version. The metrics are entirely additive. If you want to remove them, you remove the decorator. The business logic is untouched.

And then there is retry. Networks fail. Sensors go offline. APIs timeout. If a source fails on the first read, should the entire pipeline crash? Usually not — the failure might be transient. A `@retry_on_failure` decorator wraps a source's `stream()` method and catches `IOError` — which covers network-related failures — retrying up to a configured number of times with a delay between attempts. If all retries are exhausted, the exception propagates. But a transient failure that recovers on the second attempt never reaches the caller at all.

These three patterns — context managers for lifecycle, descriptors for configuration, decorators for metrics and retry — are not advanced tricks. They are foundational production engineering. They are the difference between a pipeline that works in a demo and a pipeline that runs reliably for months without manual intervention.

And they compose. The `PipelineContext` manages the lifecycle. Inside that context, `PipelineConfig` with its descriptors ensures no invalid configuration reaches the pipeline. The measured stages report on every execution. The retry decorator handles transient failures before they become crashes. Each layer adds reliability without complicating any other layer.

That is the architecture we are building. Not just correct code — resilient infrastructure.
