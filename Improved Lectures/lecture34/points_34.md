# Key Points — Lecture 34: Big Project Stage 2 — Resource Lifecycle Architecture

---

**1. Resource leaks are guaranteed without explicit lifecycle management**
When a pipeline fails mid-execution, any resources opened by the Source — database connections, file handles, network sockets — remain open indefinitely if there is no cleanup mechanism. In a long-running system that restarts automatically, leaked resources accumulate. A database connection pool with 20 slots can be exhausted by 20 crash-restart cycles, each leaving one ghost connection behind. The `PipelineContext` class provides a structured lifecycle: `__enter__` opens and records state, `__exit__` always runs and always cleans up, regardless of whether the pipeline completed or crashed.

**2. The context manager protocol is a language-enforced contract**
The `with` statement guarantees that `__exit__` will be called even if an exception propagates through the block. This is not a convention or a best practice — it is enforced by the Python interpreter. `__exit__` receives the exception type, value, and traceback as arguments, allowing it to inspect the failure mode. Returning `False` from `__exit__` tells Python to re-raise the exception after cleanup completes; returning `True` suppresses it. In production pipelines, always return `False` — silently swallowing exceptions hides failures.

**3. Descriptors intercept attribute assignment at the class level**
A Python descriptor is any object that implements `__get__`, `__set__`, or `__delete__` and is assigned as a class attribute. When `self.attributeName = value` executes on an instance, Python's attribute lookup protocol checks the class for a descriptor at that name and, if found, calls the descriptor's `__set__` method instead of storing the value directly. This means validation runs at the moment of assignment, not at the moment of use — separating the error site from the cause by zero stack frames rather than dozens.

**4. `__set_name__` eliminates manual name wiring in descriptors**
Before Python 3.6, descriptors had to be told their own name explicitly, either through the constructor or through metaclass magic. `__set_name__(owner, name)` is called automatically by Python when the class body is processed — the descriptor receives the class it belongs to and the name it was assigned under. This allows the descriptor to compute a private storage name (conventionally `_name`) that does not collide with the descriptor's own name, avoiding infinite recursion in `__get__` and `__set__`.

**5. Fail-fast validation at the configuration boundary prevents diagnostic confusion**
The principle of fail-fast means that invalid state should raise an error as close to its source as possible. A `PipelineConfig` with descriptor-validated attributes raises `TypeError` or `ValueError` at the line where the invalid value is assigned — which is exactly where the developer's attention is. Without this, the same invalid value might propagate through multiple layers before causing a crash inside a statistics computation or a normalization formula, producing a stack trace that points at symptom code rather than root cause.

**6. Decorators on generator methods create non-invasive instrumentation**
A decorator applied to a generator method wraps the entire generator function. The wrapper can initialize state before the generator starts, intercept each yielded value as it passes through, and execute cleanup code after the generator is exhausted. This makes the decorator pattern ideal for pipeline metrics — each record that passes through the stage can be counted, timed, and logged without any changes to the stage's own logic. The `@functools.wraps` call preserves the original function's name and docstring in the wrapper.

**7. Retry logic is correctly implemented as a decorator, not inline logic**
Retry behavior is a cross-cutting concern that applies to many sources and stages. Implementing it inline inside each source creates duplication and makes the retry policy difficult to change consistently. A `@retryOnFailure(maxAttempts=3, delaySeconds=1.0)` decorator wraps the `stream()` method and catches `IOError` — the standard Python exception for network and I/O failures — retrying with a delay before re-raising if all attempts are exhausted. Applied as a decorator, the retry policy is visible at a glance and changeable in one place.

**8. The lazy generator chain means records only move when pulled**
`PipelineContext.run()` chains generators — `source.stream()`, then each `stage.process(stream)` — and the chain is entirely lazy. No records move until the final `for record in stream` loop executes. This means the context manager's record counter only increments when a record actually reaches the sink. If a stage raises an exception while producing a record, `_recordCount` reflects exactly how many records made it through, giving precise failure accounting.

**9. These three patterns compose without coupling**
The context manager, descriptor validators, and stage decorators are entirely independent of each other. `PipelineContext` does not know or care about `PipelineConfig`. `PipelineConfig` does not know about decorators. A `@measuredStage` decorator works equally well on stages used inside a `PipelineContext` or called directly in tests. This independence is a design property, not an accident — each pattern addresses a single concern at a single level of the system, and their combination produces a system that is both reliable and observable.
