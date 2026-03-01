# Slides — Lecture 8: Iterators and Iterables — The Iteration Protocol

---

## Slide 1
**Title: From Closures to Iteration**
- Closures capture state at the function level
- Iteration is about state at the data model level
- Today: the precise difference between an iterable and an iterator
- And the silent bug that lives in that distinction

---

## Slide 2
**Title: How Python's For Loop Really Works**
- Step 1: Python calls `__iter__` on the collection — returns an iterator object
- Step 2: Python calls `__next__` on that iterator repeatedly
- Step 3: When `__next__` raises `StopIteration`, the loop ends
- The collection and the iterator are potentially two different objects

---

## Slide 3
**Title: Why Two Separate Objects?**
- Two nested loops over the same list need independent progress counters
- If the list tracked position, nested loops would corrupt each other
- Each call to `__iter__` returns a fresh, independent iterator
- Fresh iterator = independent position counter = safe concurrent iteration

---

## Slide 4
**Title: The Iterable Protocol**
- An iterable implements `__iter__` and returns a NEW iterator each call
- Lists, tuples, strings, dicts are all iterables
- `iter(myList)` returns a `list_iterator` — a separate object
- The list stores data; the iterator tracks position
- Iterables are reusable: `__iter__` can be called any number of times

---

## Slide 5
**Title: The Iterator Protocol**
- An iterator implements BOTH `__iter__` and `__next__`
- `__next__` returns the next value; raises `StopIteration` when done
- `__iter__` returns `self` — an iterator is its own iterator
- Iterators are single-use: once exhausted, they stay exhausted
- StopIteration is a signal, not an error

---

## Slide 6
**Title: Example 8.1 — SequenceIterator**
- Class stores values in `dataStore`, position in `currentIndex`
- `__iter__` returns `self`
- `__next__` checks `currentIndex >= len(dataStore)` → raises `StopIteration`
- Otherwise: retrieves item, increments `currentIndex`, returns item
- Complete iterator: tracks position, signals exhaustion, self-referential

---

## Slide 7
**Title: Iterators Are Exhaustible — By Design**
- Python 2.2 introduced the iterator protocol — before that, all data had to be loaded first
- File iterator: reads one line per `__next__` call — ten gigabytes, nearly zero memory
- `range(1_000_000)` computes values on the fly — nearly zero memory
- `range` is an iterable: each loop creates a fresh `range_iterator`
- A bare list iterator is single-use: second loop over it produces nothing

---

## Slide 8
**Title: The Silent Bug — Self-Referencing Iterables**
- Dangerous pattern: `__iter__` returns `self`, with position tracked internally
- This makes the object an iterator, not an iterable
- First consumer: iterates fully and exhausts the object
- Second consumer: sees an empty sequence — no error, no warning
- Data silently disappears

---

## Slide 9
**Title: Example 8.2 and 8.3 — Broken vs Fixed TransactionBatch**
- Broken: `__init__` stores `iter(data)` as instance variable; `__iter__` returns `self`
- First loop works; second loop produces nothing
- Fixed: `__init__` stores raw `transactionList`; `__iter__` returns `iter(self.transactionList)`
- Each consumer gets a fresh iterator — all consumers see complete data
- The fix: never store the iterator; store the data

---

## Slide 10
**Title: Example 8.4 — DataPage: The Correct Iterable Pattern**
- Store raw records in `recordList`
- `__len__` returns `len(recordList)`
- `__iter__` returns `iter(recordList)` — fresh iterator every call
- Multiple consumers iterate independently with no interference
- This is the same pattern used by Python's built-in types

---

## Slide 11
**Title: Industrial Applications**
- Database cursors: Django ORM fetches rows via `__next__` — for loop abstracts the source
- Large file processing: file objects are iterators — single-use, stream line by line
- API pagination: `__next__` fetches next page transparently — caller just loops
- IoT sensor streams: `__next__` reads live hardware measurements on demand
- The protocol decouples the consumer from the data source entirely

---

## Slide 12
**Title: Final Takeaway — The Precise Distinction**
- Iterable: implements `__iter__`, returns fresh iterator each call, stores raw data, reusable
- Iterator: implements `__iter__` and `__next__`, returns `self`, tracks position, single-use
- For loop: calls `__iter__` once, calls `__next__` until `StopIteration`
- Silent bug: returning `self` (or stored iterator) from an iterable's `__iter__`
- Fix: always return `iter(self.data)` — never store the iterator as state
- Next lecture: generators — implementing iterators without the boilerplate

---
