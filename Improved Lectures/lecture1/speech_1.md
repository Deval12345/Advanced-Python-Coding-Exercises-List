Welcome to the lecture on the Data Model!

Before we write a single line of code, I want you to think about something. What separates a Python developer who builds production systems at Google, or architects data pipelines at a hedge fund, from someone who just knows the syntax?

It is not knowing more libraries.
It is not memorizing more functions.

It is understanding the machine underneath.

At the core of Python is the Data Model.
Master it, and you master the language.

Let us begin!

Think about this. Every time you call len on something in Python, or loop through a list with a for loop, or check if an item is inside a collection, Python is not just calling a simple function. It is reaching into your object and asking a very specific question. Does this object know how to answer?

This is the foundation of the Python Data Model. Python was designed as a framework, and the objects you create are the plugins. And this is a deliberate architectural choice. Instead of giving you a fixed set of data types and saying, Here, use only these, it gives you a protocol. Implement specific special methods, and your objects become first-class citizens of the language.

Now why does this matter in industry?

Imagine you are building a data pipeline at a fintech company. You need custom data structures that behave exactly like Python lists or dictionaries, but underneath they connect to a database or stream data from an API.

Because of the Data Model, you can build these objects, and every Python library, every framework, every tool built by you or any other team already knows how to work with them. If it works with Python’s built-in lists and dictionaries, it will work with your custom objects. No special adapters. No translation layers.

This is not a theoretical exercise. jango QuerySets work this way. Pandas DataFrames work this way. PyTorch tensors work this way. The biggest, most battle tested Python projects in the world are built on exactly this principle.

Now let me show you a concrete example that makes this real. Imagine you are building an inventory management system. You have a collection of products, and you want this collection to behave like a native Python sequence.

Here is our ProductCatalog class. Inside the special dunder method init, we store a list of products in an instance variable called itemList. Now here is the key. We define three special methods. Special dunder method len returns the length of itemList. Special dunder method getitem accepts an index parameter and returns the item at that position in itemList. And special dunder method contains accepts a productName parameter and checks whether any product in itemList has a matching name.

Now watch what happens. We create a catalog, and without any extra work, we can call len on it, we can use square bracket indexing, and we can use the in operator. Python sees these operations, looks for the corresponding special methods, and calls them automatically. Our custom object behaves exactly like a built in type.

Now let us go deeper into these special methods, because understanding them individually is where the real power comes from.

Special dunder method len. When you call len on any object, Python looks specifically for this method. This is your object's answer to the question, how big are you? In a machine learning pipeline, you might have a Dataset class that wraps thousands of training samples stored across multiple files. By implementing special dunder method len, your Dataset object can tell a DataLoader exactly how many samples it contains, and the DataLoader never needs to know whether those samples are in memory, on disk, or streaming from a remote server.

Special dunder method getitem. This is the method behind square bracket access. When you write myObject at position five, Python calls special dunder method getitem with five as the argument. But here is where it gets interesting. If you implement special dunder method getitem, Python automatically gives your object two additional capabilities for free. It becomes iterable, meaning you can use it in a for loop, and it becomes compatible with slicing. One method, three behaviors.

Think about this in a web backend context. You could build a PaginatedResults class that fetches data from an API page by page. Implement special dunder method getitem, and suddenly your results object supports indexing, iteration, and slicing, all backed by lazy network calls. The consumer of your class does not need to know about pagination at all.

Let me show you this with a TimeSeries class, something you might actually build in a quantitative finance application.

We define a class called TimeSeries. In special dunder method init, we accept two parameters, dateLabels and priceValues, and store them as instance variables. We implement special dunder method len to return the length of priceValues. We implement special dunder method getitem to return the price at a given position.

Now here is the beautiful part. We create a TimeSeries with three dates and three prices. We can call len on it and get three. We can index into it with square brackets. But we can also iterate over it with a for loop, even though we never explicitly defined special dunder method iter. Python sees that special dunder method getitem exists and automatically uses it to generate an iteration protocol. It starts at index zero, calls getitem, moves to index one, calls getitem again, and keeps going until it gets an IndexError.

This is the Python Data Model at work. One method unlocked three behaviors.

Now here is something that surprises most intermediate Python developers. When you call len on a list, Python does not actually go through the normal method lookup process. It does not search through the method resolution order. It does not call list dot special dunder method len through the standard Python dispatch mechanism.

Instead, CPython, the reference implementation of Python, takes a shortcut. For built in types like lists, tuples, strings, and dictionaries, the len function goes directly to a C level structure called PyVarObject, reads a field called ob size, and returns it immediately. This is a direct memory read. No Python level method call at all. It is extremely fast.

Why does this matter? Because in performance critical applications, this difference is significant. When you are processing millions of records in a data pipeline, or evaluating neural network layers in a training loop, these tiny per call savings add up to real time savings. The Python designers chose this architecture deliberately. They wanted the syntax to be clean and consistent, len of x for everything, but they also wanted built in types to be as fast as possible.

For your custom objects, Python does call special dunder method len through normal dispatch. But the key insight is this. The consistent interface, calling len of x rather than x dot length or x dot size or x dot count, means that every function and library that uses len will work with your objects too. You get the clean interface. Built in types get the speed optimization. Everyone wins.

Let us close with one of the most important ideas in Python, and one that separates Python from languages like Java or Cplusplus. Duck typing.

The name comes from the old saying. If it walks like a duck and quacks like a duck, then it is a duck. In Python terms, this means we do not care what type an object is. We care what it can do. We care about its behavior, not its inheritance tree.

Why was Python designed this way? Because in real software systems, rigid type hierarchies create fragile code. In Java, if you want an object to be sortable, it must implement the Comparable interface. If you want it to be iterable, it must implement Iterable. You end up with classes that implement five or six interfaces just to be useful. And if a library expects a specific interface you did not plan for, you are stuck.

Python takes the opposite approach. If your object has special dunder method len, it has length. If it has special dunder method getitem, it is indexable. No registration required. No interface declaration. No inheritance from a special base class. Your object just needs to respond to the right questions.

This is incredibly powerful in industry. Think about testing. In Java, you need complex mocking frameworks to create test doubles. In Python, you can pass in any object that has the right methods. Building a function that reads from a file? Pass in any object with a read method. It could be a real file, a network stream, a string buffer, it does not matter. If it has the method, it works.

Here is a practical example. We write a function called computeStats that accepts a parameter called dataSource. Inside, it calls len of dataSource to get the count, and uses square bracket indexing to calculate statistics.

Now here is the critical point. This function does not check the type of dataSource. It does not require dataSource to inherit from any particular class. It simply calls len and uses square bracket access. Any object that implements special dunder method len and special dunder method getitem will work perfectly.

We can pass in a plain Python list, and it works. We can pass in our TimeSeries object from earlier, and it works. We could pass in a Pandas Series, a NumPy array, or any custom class that implements these two methods. The function does not know and does not care.

This is duck typing in action. And this is why the Python Data Model is not just an academic curiosity. It is the architectural foundation that makes Python the dominant language in data science, machine learning, web development, and automation. Every major Python library is built on this principle.

In the next lecture, we will take this further and build a complete custom collection class that implements the full sequence protocol. You will see how just a handful of special methods can make your objects indistinguishable from built in Python types.
