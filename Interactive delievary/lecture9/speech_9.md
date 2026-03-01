In the previous session we used EAFP and exceptions to keep the happy path clear. Today we return to the data model. We already saw that a for loop over an object uses the special dunder method iter. Now we go deeper: what is the difference between an iterable and an iterator, and how does yield make lazy sequences possible?

By the end of this lesson you will understand the iterator protocol, why each iteration should get a fresh iterator, and how generator functions produce values one at a time without loading everything into memory.

We used iter to make objects work in for loops. An iterable is an object that can supply an iterator. When you write for item in something, Python calls the special dunder method iter on something to get an iterator. So the iterable is the container or stream; the iterator is the object that actually produces values one by one and remembers where it is.

An iterable implements the special dunder method iter and returns an iterator. An iterator implements the special dunder method next. Each time next is called, the iterator returns the next value or raises an exception when there are no more. So iter gives you a fresh iterator; next drives it. Built-in functions like sum and list call iter on the object you pass, so any iterable works with them.

A common mistake is to return the same iterator from every call to iter. If the first consumer exhausts it, the second consumer gets nothing. So the special dunder method iter must return a new iterator each time, or return a fresh view over the data. That way multiple loops or multiple calls to sum and list all work correctly.

Here we define a class named Batch. Inside it we store a list in a variable named dataStore. We implement the special dunder method iter. It returns the result of calling iter on dataStore, which is a new iterator over that list. Because we return a fresh iterator each time, we can use the Batch in a for loop, pass it to sum, or pass it to list, and each use gets its own iteration. So the object is a proper iterable.

A generator function is a function that uses yield. When you call it, it does not run the body immediately. It returns a generator object, which is an iterator. Each time you call next on that generator, the function runs until the next yield and produces one value. So the function produces values lazily, one at a time. That allows streaming: you can process a large or infinite sequence without loading it all into memory.

Here we define a function named countUpTo that takes a parameter named limit. Inside it we loop from zero to limit. In the loop we yield the current value. So countUpTo is a generator function. When we call it we get a generator. When we iterate over that generator, we get zero, one, two, and so on up to limit minus one. The values are produced on demand. We never build a full list in memory. So yield turns a function into a lazy iterator.

If you read a large file and return a list of all lines from the special dunder method iter, you load the entire file into memory. If instead you open the file and yield one line at a time, you only hold one line in memory. So generators enable streaming pipelines: one stage yields values, the next stage consumes them, and the data flows without building huge lists. This pattern is used in data processing and APIs everywhere.

An iterable provides an iterator via the special dunder method iter. An iterator provides values via the special dunder method next. The special dunder method iter should return a fresh iterator each time. Generator functions use yield to produce values lazily; they are the easiest way to implement iterators. Lazy iteration enables streaming and keeps memory use bounded. This is the foundation of scalable data processing in Python.
