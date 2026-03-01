# Slides — Lecture 6: Functions as Objects — First-Class Functions and Runtime Behavior

---

Slide 1, Title: Transition from Lecture 5 — Behavior Is Also Data
Point 1: In Lecture 5 we learned that variables are references — they point to objects in memory rather than containing them.
Point 2: Functions in Python follow the same rule — a function definition creates a function object and binds it to a name.
Point 3: Because functions are objects, they can be referenced, passed, stored, and returned just like lists and strings.
Point 4: This single insight enables an entirely new class of system design patterns.

---

Slide 2, Title: Functions Are Objects — What This Actually Means
Point 1: When Python executes a def statement, it creates a function object in memory and binds the function name as a reference to it.
Point 2: That name is just a variable — reassigning it or passing it elsewhere follows the same rules as any other variable.
Point 3: Function objects have attributes: dunder name stores the function's name, dunder doc stores the docstring.
Point 4: The motivation: when behavior is data, systems can choose what to do at runtime rather than having it fixed at write time.

---

Slide 3, Title: What First-Class Really Means in Practice
Point 1: First-class means a function can appear anywhere a value can appear — in a list, a dictionary, a parameter, or a return value.
Point 2: No special syntax is required — using a function name without parentheses treats it as a value rather than calling it.
Point 3: Route handlers in web frameworks, sort keys, and data pipeline steps are all functions stored and passed as values.
Point 4: Understanding this makes the architecture of major frameworks transparent — they are built on function-as-value patterns.

---

Slide 4, Title: Behavior Injection — Passing Functions as Arguments
Point 1: Behavior injection means passing a function as an argument so the caller controls what the receiving function does.
Point 2: The receiving function does not care which specific function it receives — it just calls it with the appropriate arguments.
Point 3: Like protocols enabling interchangeable objects, any function with the right signature can be injected as a behavior.
Point 4: Result: processing logic and business logic are decoupled — each can be changed, tested, and extended independently.

---

Slide 5, Title: Eliminating Conditional Logic with Behavior Injection
Point 1: Without behavior injection, every new case requires adding an elif branch to an existing function — the function grows and fragments.
Point 2: With behavior injection, the processing function never changes — new behavior means writing a new function and passing it in.
Point 3: This is the open-closed principle: the system is open to extension through new functions, closed to modification of core logic.
Point 4: Fewer branches means smaller functions, clearer responsibilities, and tests that do not break when new rules are added.

---

Slide 6, Title: Functions in Data Structures — Pipelines and Routing Tables
Point 1: Storing functions in a list creates a pipeline — each function transforms the data and passes it to the next step.
Point 2: Storing functions in a dictionary creates a routing table — keys map to behaviors that are looked up and invoked at runtime.
Point 3: Django middleware, data science preprocessing, and ETL workflows are all pipelines of functions stored in lists.
Point 4: The pipeline applier has one job — iterate and call — it never needs to know what specific steps the pipeline contains.

---

Slide 7, Title: Returning Functions — The Factory Pattern
Point 1: A function that creates and returns another function is a factory — it produces customized callables on demand.
Point 2: You call the factory with configuration parameters and receive back a function that encapsulates that configuration.
Point 3: Each returned function is independent — it can be named, stored, passed, and tested like any other function.
Point 4: This is the foundation of a broader pattern — closures in Lecture 7 will explain exactly how returned functions retain their configuration.

---

Slide 8, Title: Callables and __call__ — Objects That Behave Like Functions
Point 1: A callable is anything that can be invoked with parentheses — functions are callables, but they are not the only callables.
Point 2: Any class that implements the dunder call method produces instances that are also callable.
Point 3: A callable class instance can maintain state in its attributes while still being passed anywhere a function is expected.
Point 4: Python does not ask whether something is a function — it asks whether it is callable, making the type system highly flexible.

---

Slide 9, Title: Final Takeaway — Treating Behavior as Data
Point 1: The four patterns enabled by first-class functions are behavior injection, runtime strategy selection, pipelines, and factories.
Point 2: Each pattern reduces conditional logic, separates responsibilities, and makes systems extensible without modifying existing code.
Point 3: When you treat behavior as data, you write smaller functions that do one thing and compose them into larger systems.
Point 4: Lecture 7 extends this further with closures, where returned functions remember state from their creation context, and lambda for inline behavior.
