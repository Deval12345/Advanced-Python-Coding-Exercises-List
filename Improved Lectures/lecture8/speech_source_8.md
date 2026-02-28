# Speech Source — Lecture 8: Iterators and Iterables — The Iteration Protocol

---

## CONCEPT 0.1 — Transition from Closures to Iteration

In our previous lecture, we studied closures — functions that remember the environment in which they were created. We saw how a function could carry captured state with it, holding onto a variable from an outer scope long after that outer scope had finished executing.

Today we return to the Python data model. But we go much deeper than we did in Lectures 1 and 2. We already saw that for loops and the `__iter__` method are connected. We already saw that implementing `__iter__` on a class is what makes your object iterable. But we never asked the deeper question: what exactly is an iterator? How is it different from an iterable? And why does that distinction matter so profoundly?

Today we answer those questions precisely. We will understand the iteration protocol from first principles. And we will see how a simple misunderstanding between an iterable and an iterator creates one of the sneakiest silent bugs in all of Python.

---

## CONCEPT 1.1 — How Python's For Loop Really Works

Let us begin at the foundation. When you write a for loop in Python, you might imagine that Python simply moves through a collection one item at a time. But that mental model is incomplete. Python executes a precise two-phase protocol every single time a for loop runs.

Here is what actually happens. When Python encounters the line `for item in collection`, it does not begin moving through the collection immediately. It first calls `__iter__` on the collection. That call returns a completely separate object — an iterator. Then, Python calls `__next__` on that iterator, again and again, until `__next__` raises a `StopIteration` exception. When that exception is raised, the loop ends cleanly.

The collection and the iterator are potentially two entirely different objects. This is the key architectural insight. Python deliberately separates the thing you iterate over from the object that actually tracks where you are in the iteration.

Why design it this way? Imagine you want to run two separate for loops over the same list at the same time — perhaps in a nested loop comparing every pair of elements. Each loop needs its own independent progress counter. If the list itself tracked the current position, two nested loops would collide and corrupt each other's state. By returning a fresh, independent iterator object each time `__iter__` is called, Python makes this safe, clean, and predictable.

---

## CONCEPT 1.2 — The Iterable Protocol

An iterable is any object that implements `__iter__` and returns a new, independent iterator each time `__iter__` is called.

Lists are iterables. Tuples are iterables. Strings are iterables. Dictionaries are iterables. When you call `iter()` on a list, Python returns a `list_iterator` object — a separate, lightweight object whose only job is to track the current position in that list. The list itself has no idea where you are in the iteration. Only the iterator does.

This design gives you something powerful: an iterable is reusable. You can call `iter()` on a list one hundred times and get one hundred independent iterators, each starting from the beginning, each independent of the others.

The rule for an iterable is simple: implement `__iter__`, and return a fresh iterator each time it is called.

---

## CONCEPT 2.1 — The Iterator Protocol

An iterator is a different kind of object. An iterator implements two methods: `__iter__` and `__next__`.

The `__next__` method is the core of an iterator. Each call to `__next__` returns the next value in the sequence. When there are no more values, `__next__` raises `StopIteration`. That exception is not an error. It is a signal. It is the normal, expected way an iterator communicates that it is finished.

The `__iter__` method on an iterator is deliberately simple: it returns `self`. An iterator is its own iterator. This might seem circular, but it has a practical purpose: it means you can pass an iterator directly into a for loop, and the for loop will work correctly, because it will call `__iter__` on the iterator, get the iterator itself back, and then call `__next__` on it.

The critical consequence of this design: iterators are single-use. Once `__next__` has raised `StopIteration`, the iterator is exhausted. Calling `__next__` again should continue raising `StopIteration`. The iterator cannot reset itself. It represents exactly one complete pass through a sequence of values.

---

## EXAMPLE 2.1 — SequenceIterator: A Custom Iterator from Scratch

Here we define a class named `SequenceIterator`.

Inside the special dunder method `__init__`, we store a list of values inside a variable named `dataStore` and set a position variable named `currentIndex` to zero.

We implement the special dunder method `__iter__` to return `self`. The iterator is its own iterator.

We implement the special dunder method `__next__`. If `currentIndex` is greater than or equal to the length of `dataStore`, we raise `StopIteration`. Otherwise, we retrieve the item at `currentIndex`, increment `currentIndex` by one, and return the item.

This is a complete, functioning iterator. It tracks its own position in `currentIndex`. It returns values one at a time through `__next__`. It signals exhaustion by raising `StopIteration`. And it identifies itself as its own iterator by returning `self` from `__iter__`. Every piece of Python's iteration protocol is satisfied.

---

## CONCEPT 2.2 — Iterators Are Exhaustible

This single-use nature of iterators is not a flaw. It is a deliberate design choice that enables Python to process data that is far too large to hold in memory.

Consider iterating over a file. The file might be ten gigabytes. You cannot load ten gigabytes into a list. But you can read it one line at a time. The file object acts as an iterator: each call to `__next__` reads and returns the next line. When the file ends, `StopIteration` is raised. The entire file was processed without ever holding more than one line in memory.

This same design powers `range()`. When you write `range(1_000_000)`, Python does not create a list of one million numbers. It creates a small object that knows its start, stop, and step. Each call to `__next__` on a `range_iterator` computes the next number on the fly. The memory usage is essentially zero regardless of the range size.

This is why `range()` is an iterable, not an iterator. Every time you use `range(5)` in a for loop, Python creates a fresh `range_iterator`. But if you manually create an iterator from a list by calling `iter(myList)`, and then try to iterate over it twice, the second pass produces nothing. The iterator was already exhausted by the first pass.

---

## CONCEPT 3.1 — The Silent Bug: Self-Referencing Iterables

Now we come to one of the most important and most dangerous patterns in Python. This is a bug that produces no error message, no traceback, no warning of any kind. Data simply disappears.

Here is the scenario. You build a class to represent a collection of data — let us say a `TransactionBatch`. You give it an `__init__` that stores transaction data. You give it an `__iter__` method. But inside `__iter__`, instead of returning a fresh iterator over your data, you store `iter(self.data)` as an instance variable and return `self`.

What have you done? You have made your `TransactionBatch` an iterator, not an iterable. It has internal state that tracks position. The first time someone iterates over it, everything works. But the iterator is now exhausted. The second time someone iterates over it, they see nothing. No error. No exception. Just an empty sequence.

In a financial system, this is catastrophic. Imagine a batch of ten thousand transactions. The first consumer iterates through all of them to compute totals. The totals look correct. Then the second consumer iterates through the same batch to generate a report. The report is empty. The second consumer has no idea why. The batch object looks perfectly normal. No exception was raised. The data simply did not appear.

This is the silent bug. It happens precisely when an iterable returns itself — or an exhausted iterator — from `__iter__`.

---

## EXAMPLE 3.1 — Broken and Fixed TransactionBatch

We will look at two versions of a `TransactionBatch` class side by side.

In the broken version, `__init__` stores the data as `iter(self.data)` — immediately converting the data to an iterator and storing that iterator as an instance variable. The `__iter__` method returns `self`. The result: the object is an iterator. It can be consumed exactly once. Any second consumer sees an empty sequence.

In the fixed version, `__init__` stores the raw data list in a variable named `transactionList`. The `__iter__` method returns `iter(self.transactionList)` — creating a brand new iterator from the raw data on every single call. The result: the object is a proper iterable. Each consumer gets their own fresh, independent iterator. All consumers see the complete data.

The fix is two characters different in terms of what it returns. The consequence is the difference between a correct system and a silent data loss bug.

---

## CONCEPT 4.1 — Building a Practical Custom Iterable

With the distinction between iterables and iterators now clear, let us build a well-designed custom iterable from scratch.

The pattern is simple: store the raw data. Return a fresh iterator from `__iter__`. Never track position inside the iterable itself.

This is the exact same design that Python's built-in types use. A list stores raw data and returns a fresh `list_iterator` from `__iter__`. A tuple stores raw data and returns a fresh `tuple_iterator`. A string stores raw data and returns a fresh `str_ascii_iterator`. The pattern is universal.

---

## EXAMPLE 4.1 — DataPage: The Correct Custom Iterable Pattern

Here we define a class named `DataPage`.

Inside the special dunder method `__init__`, we store a list of records in a variable named `recordList`.

The special dunder method `__len__` returns the length of `recordList`.

The special dunder method `__iter__` returns `iter(recordList)` — a brand new iterator each time it is called.

Now multiple consumers can independently iterate over the same `DataPage` without any interference. Consumer one starts a for loop, gets a fresh iterator at position zero. Consumer two simultaneously starts their own for loop, gets a completely separate fresh iterator also at position zero. Neither affects the other. This is the correct design.

---

## CONCEPT 5.1 — Industrial Applications of the Iteration Protocol

The iteration protocol is not an academic exercise. It is the foundation of how Python processes data at scale in production systems around the world.

Database cursors in Django's ORM and SQLAlchemy expose query results through the iteration protocol. When you write a for loop over a query result, you are not loading all rows into memory. You are calling `__next__` on a database cursor, which fetches rows from the database one batch at a time. The consuming code uses a plain for loop and never needs to know whether it is iterating a list or querying a live database. The protocol abstracts the source completely.

Large file processing in data engineering relies on this protocol constantly. Reading a ten-gigabyte CSV file line by line is only possible because file objects implement the iterator protocol. The file object is itself an iterator — it returns `self` from `__iter__` — which means it can only be iterated once. If you need to re-read a file, you must seek back to the start or open a new file handle.

API pagination in network clients uses the protocol to hide complexity. A `PaginatedResults` object can implement `__next__` to transparently fetch the next page from an API endpoint. The caller writes a for loop. The iterator handles authentication, HTTP requests, response parsing, and page transitions. The caller never sees any of that complexity. They just iterate.

IoT and real-time sensor data streams use the same pattern. A sensor stream iterator calls `__next__`, which reads the latest measurement from a hardware interface or a message queue. The calling code iterates. The hardware complexity is hidden behind two methods.

---

## CONCEPT 6.1 — Final Takeaway

Let us be precise about what we learned today.

An iterable implements `__iter__` and returns a fresh, independent iterator each time. It can be iterated any number of times. It stores raw data. It does not track position.

An iterator implements both `__iter__` and `__next__`. Its `__iter__` returns `self`. Its `__next__` returns the next value and raises `StopIteration` when exhausted. It tracks position. It is single-use.

The for loop calls `__iter__` once to get an iterator, then calls `__next__` repeatedly until `StopIteration` is raised.

The silent bug occurs when an iterable returns itself or an already-created iterator from `__iter__`. The first consumer exhausts it. The second consumer sees nothing, with no error.

The fix is always the same: store raw data, return `iter(self.data)` from `__iter__`, never store the iterator as instance state.

In our next lecture, we will see how generators make implementing iterators dramatically simpler. Instead of writing a class with `__iter__` and `__next__` and manually managing `currentIndex`, you write a function with a single `yield` keyword, and Python handles all the state management automatically. But now that you understand the protocol precisely, you will understand exactly what a generator is doing under the hood.
