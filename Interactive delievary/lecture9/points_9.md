Slide 0
Title: Previous Lecture Takeaway
Point 1: EAFP and exceptions keep the happy path linear; today we return to the data model for iteration and lazy sequences.

Slide 1
Title: Iterable Versus Iterator
Point 1: Iterable supplies an iterator via *iter*; iterator produces values via *next*; *for* calls *iter* to get an iterator.
Point 2: *iter* must return a new iterator each time so multiple consumers do not share an exhausted iterator.

Slide 2
Title: Implementing an Iterable
Point 1: Return iter over data from *iter* to get a fresh iterator so *for*, *sum*, *list* all work.

Slide 3
Title: Generators and yield
Point 1: A generator function uses *yield*; calling it returns a generator (iterator); *next* runs until the next *yield* and produces one value lazily.
Point 2: *yield* turns a function into a lazy iterator without building a full list in memory.

Slide 4
Title: Why Lazy Matters
Point 1: Yielding one line at a time streams data; building a list loads everything; generators enable streaming pipelines.

Slide 5
Title: Final Takeaways
Point 1: Iterable has *iter*, iterator has *next*; *iter* returns fresh iterator; generators and *yield* give lazy iteration and bounded memory.
