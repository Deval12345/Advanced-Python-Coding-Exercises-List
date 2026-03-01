# Speech Source — Lecture 9: Generators — Lazy Evaluation and Producer-Consumer Architecture

---

## CONCEPT 0.1 — Transition from L8

In the previous lecture we built custom iterators from scratch, implementing dunder iter and dunder next manually. We saw that this works, but it requires careful state management. You had to write a class, track a position variable, handle StopIteration by hand, and make sure dunder iter returned self while dunder next advanced the state correctly.

Python offers a far simpler way to create iterators: generator functions. With a single keyword — yield — Python handles all the iterator machinery for you. No class required. No manual state tracking. No explicit StopIteration. Just a function that uses yield wherever it wants to produce a value.

---

## CONCEPT 1.1 — What yield Does

A generator function is any function that contains a yield statement. The presence of yield changes everything about how Python treats the function.

When you call a generator function, Python does not run the function body immediately. Instead, it creates a generator object and returns it. The function body only runs when you start iterating over the generator object — when you call next on it or use it in a for loop.

Each time the generator executes a yield statement, two things happen: it produces a value for the caller, and it pauses execution. The function's local variables, its position in the code, and everything about its current state is preserved exactly as it was at the moment of pausing. The generator is suspended mid-execution.

When the consumer asks for the next value — by calling next again or continuing the for loop — the generator resumes exactly where it paused. It picks up from the line after the yield statement and continues running until it hits the next yield or reaches the end of the function.

When the function body finishes — either by running off the end or hitting a return statement — Python automatically raises StopIteration. The generator knows it is done, and it signals this to the caller without you writing any explicit error handling.

This pause-and-resume behavior is what makes generators powerful. The generator function's execution is interleaved with the caller's execution. They take turns: the generator produces a value, hands control back to the caller, the caller processes it, then asks for the next value, and control returns to the generator.

---

## EXAMPLE 1.1 — Simple Countdown Generator

Here we define a generator function named countdown that receives a parameter named startValue.

Inside, we use a while loop. While startValue is greater than zero, we yield startValue and subtract one from startValue.

When we iterate over countdown with ten, we get values ten, nine, eight, down to one.

Python automatically implements the full iterator protocol. The countdown call returns a generator object with both dunder iter and dunder next already implemented. You get a fully functioning iterator from a function that looks almost like a plain loop.

---

## CONCEPT 1.2 — The Memory Advantage

Consider what happens when you build a list versus a generator for a large range of numbers.

A list comprehension computes all values immediately and stores them all in memory at once. For a million numbers, that means a million integer objects sitting in RAM before you have processed a single one.

A generator expression computes values one at a time, on demand. For a million numbers, it holds only the current value and enough state to compute the next one. The memory usage is constant regardless of how many values the generator will eventually produce.

For twenty values, both approaches behave identically from the caller's perspective. The results are the same. For twenty million values, the list could use hundreds of megabytes, while the generator uses a fixed, tiny amount — often just a few hundred bytes.

This is the meaning of lazy evaluation. The values are not computed until they are needed. You describe what to produce, and the generator produces each value only at the moment it is requested.

---

## CONCEPT 2.1 — Generator Expressions vs List Comprehensions

You already know list comprehensions: square brackets containing an expression and a for clause. Generator expressions look identical except they use parentheses instead of brackets.

A list comprehension eagerly produces all values and stores them in a list object. A generator expression produces a generator object. No values are computed until you iterate.

When you pass a generator expression directly to a function like sum or max, you do not need the outer parentheses — the function call's own parentheses serve double duty. This is a common Python idiom.

The practical rule: if you need all values at once, or you need to iterate multiple times, use a list. If you are doing a single pass, use a generator expression. The syntax difference — brackets versus parentheses — is small. The memory difference can be enormous.

---

## EXAMPLE 2.1 — Processing Large Transaction Log

Here we define a generator function named readTransactions that receives a parameter named logLines.

For each line in logLines, we strip whitespace, parse the amount and currency from the line, and yield a dictionary containing those values.

Whether logLines contains ten records or ten million, this generator processes one at a time. It does not load the entire log into memory. It does not build a list of dictionaries upfront.

The caller consumes values as fast as it can process them. If the caller slows down, the generator pauses. If the caller is fast, the generator produces the next value immediately. Memory stays bounded at all times.

This pattern maps directly to reading large files: you open the file, iterate line by line using a generator, and process each line without ever holding the entire file in memory.

---

## CONCEPT 3.1 — Producer-Consumer Architecture

Generators enable a clean separation between the producer of data and the consumer of data.

The producer is the generator function. It knows how to produce values — from a file, from an API, from a database, from a computation. It yields each value and waits.

The consumer is whatever iterates the generator. It knows how to process values — aggregate them, filter them, write them somewhere. It asks for the next value and processes it.

Neither the producer nor the consumer knows about the other's implementation. The producer does not know whether its values are being summed, filtered, or written to a file. The consumer does not know whether values are coming from a file, an API, or a simulation.

This decoupling is the core of the producer-consumer pattern. You can swap the producer without touching the consumer. You can swap the consumer without touching the producer. You can insert new processing stages between them. Each stage is independently testable.

---

## EXAMPLE 3.1 — Generator Pipeline

Here we define three generator functions that form a pipeline.

The first function is named readRecords. It receives a parameter named sourceData. For each item in sourceData, it yields the item. This is the producer stage — it could be reading from a file, a database query, or any data source.

The second function is named filterValidRecords. It receives a parameter named records. For each record in records, it checks whether the amount is greater than zero. If so, it yields the record. Records with non-positive amounts are silently dropped. This is the filter stage.

The third function is named computeTotals. It receives a parameter named validRecords and a parameter named currency. For each record in validRecords that matches the given currency, it yields the computed total value. This is the transformation stage.

We chain them by passing each generator's output as the input to the next: readRecords feeds into filterValidRecords, which feeds into computeTotals. The final stage's output is what we iterate over.

Each stage processes one value at a time. No stage waits for the previous stage to finish completely. When the consumer asks for a value, computeTotals asks filterValidRecords for a valid record, which asks readRecords for a record. The value propagates through the pipeline one step at a time. The pipeline runs with constant memory, regardless of how large the source data is.

---

## CONCEPT 4.1 — Lazy Evaluation in Industrial Systems

The generator pattern is not just a Python trick. It underlies major systems and frameworks.

Django's QuerySet is lazy. When you write a filter or exclude on a QuerySet, no database query is executed. The query runs only when you iterate over the QuerySet, or when you call a method that forces evaluation like len or list. This is exactly the generator principle: describe the computation first, execute it only when needed.

Pandas operations are vectorized and can be chained. Newer data libraries like Polars use explicit lazy evaluation — you build a computation graph and call collect only when you want results.

Apache Spark processes massive datasets across clusters using a lazy evaluation model. Each transformation — filter, map, groupBy — builds a plan. The plan executes only when you call an action like collect or count.

PyTorch's DataLoader uses a generator-like iterator to load batches lazily during training. It does not load the entire dataset into GPU memory. It loads one batch at a time, processes it, discards it, and loads the next.

The key insight across all these systems: lazy evaluation means you describe what to compute before how and when. You build a computation plan. You execute it only when needed, and only as much as needed. This is efficient and elegant.

---

## CONCEPT 5.1 — When Generators Are Not Appropriate

Generators are not always the right tool. Understanding their limitations is as important as understanding their strengths.

Generators are sequential and one-pass. Once a value has been yielded and consumed, it is gone. You cannot go back. If you need to re-read earlier values, you must either store them yourself or restart the generator from the beginning.

Generators do not support indexing. You cannot ask a generator for the fifth value without consuming the first four. If you need random access — retrieving values by position — use a list.

Generators do not know their own length until fully consumed. If you need to know the count upfront, you must consume the generator first, which defeats the memory advantage.

Use generators for sequential processing of large or potentially infinite data, for one-pass pipelines, for streaming data where values arrive over time, and for situations where memory efficiency matters.

Use lists for random access, for data you will iterate multiple times, for known fixed small datasets, and for operations that require the entire collection — like sorting or binary search.

The practical heuristic: start with a generator if you are processing data sequentially. Convert to a list only when you discover you need capabilities that generators cannot provide.

---

## CONCEPT 6.1 — Final Takeaway

Generators turn complex iterator state machines into simple functions with yield. Where the L8 pattern required a full class with dunder iter and dunder next and explicit state management, a generator function requires none of that. The iterator protocol is implemented automatically.

They enable lazy evaluation, which keeps memory bounded even for enormous datasets. A log file with fifty million lines, a real-time market feed with millions of events per day, a machine learning dataset with millions of images — all of these can be processed one item at a time with a generator, using only constant memory.

Generator pipelines decouple producers from consumers, enabling each stage to be developed and tested independently. Each stage is a pure function from an input stream to an output stream. You can test readRecords with mock data. You can test filterValidRecords independently. You can compose them in any order.

The consequence for real systems: data engineering pipelines in production use exactly this architecture. Apache Beam, Luigi, Airflow — all build on the idea of composable data transformations. The generator is the simplest possible implementation of the same idea.

In the next lecture, we will explore decorators, which use the closure and first-class function concepts from our earlier lectures to inject behavior into functions non-invasively. You will see how a decorator wraps a function, adds behavior before and after it, and returns a new function — all without modifying the original function's code.

