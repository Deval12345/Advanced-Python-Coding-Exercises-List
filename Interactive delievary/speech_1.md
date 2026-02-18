Alright everyone, let us get started.

Today we are going to talk about something that quietly powers everything you do in Python, but very few people learn it explicitly. It is called the Python Data Model, also known as magic methods or dunder methods. [slide 1 point 1]

For a moment, forget the idea that Python is just a scripting language with nice syntax. Python is actually a framework, and the real interface of that framework is objects. [slide 1 point 2]

When you build a web application using a framework, you do not directly control everything. You plug into the framework by following certain rules. Python works the same way. You plug into its syntax by implementing certain methods. [slide 1 point 3]

By the end of this lesson, you should be able to look at normal Python syntax and mentally translate it into method calls. Once that clicks, Python feels predictable instead of magical. [slide 1 point 4]

Before we dive in, here is what I want you to walk away with.

By the end of this session, you will understand Python as an object centric framework. [slide 2 point 1] You will know how normal syntax like calling the len method with a variable named x, or indexing a variable named x at position i, maps to special methods. [slide 2 point 2] You will see duck typing in action. [slide 2 point 3] You will learn how to design your own objects so they behave like built in types. [slide 2 point 4] And finally, you will understand operator overloading in a responsible way. [slide 2 point 5]

If you have ever wondered how libraries like pandas or PyTorch make their objects feel native to Python, today you will understand the mechanism behind that. [slide 2 point 6]

Here is a mindset shift.

Python does not add new syntax for new features. Instead, Python says that if your object implements certain methods, it will let that object participate in its syntax. [slide 3 point 1]

When you call the len method with a variable named x, Python internally checks whether that variable named x has a special len method defined. If it does, Python calls it. [slide 3 point 2] That is why the len method works on strings, lists, tuples, sets and dictionaries. They are totally different types, but they follow the same protocol. [slide 3 point 3]

When you index a variable named x at position i, Python translates that into a get item method call with i as the argument on that object. [slide 3 point 4]

When you write a for loop over a variable named obj, Python checks whether it can get an iterator from that object using a special iter method. [slide 3 point 5]

When you check whether a variable named x is inside a variable named obj, Python checks whether that object knows how to perform containment using a special contains method. [slide 3 point 6]

The syntax is just surface level sugar. The real work happens in special methods. Once you accept this, Python becomes incredibly consistent. [slide 3 point 7]

Now let us make this concrete.

We are going to build a custom object that behaves like a list, not by inheriting from list, but by speaking Python language. [slide 4 point 1]

Imagine you are building a logging system, or wrapping a database result, or adding validation around a collection. You still want it to feel like a list to the user. [slide 4 point 2]

Before we look at the code, think about this. What operations do lists support. Which special methods enable those operations. [slide 4 point 3]

Let us consider this code block.

In [module 1 line 1], we define a new class named CustomList. At this point, it is just a brand new type.

In [module 1 line 2], we define the init method with parameters self and data. This is standard object initialization.

In [module 1 line 3], we assign a variable named self data to the result of calling the list constructor on data. Whatever comes in gets converted into a list and stored internally. That internal variable is meant to represent private state.

In real systems, this could be wrapping API data, cached results, or validated inputs. Externally, though, we still want users to interact with it like a normal list.

In [module 1 line 6], we define the len method for this class. This method gets called when someone calls the len method with a variable named custom list.

In [module 1 line 7], we return the result of calling the len method on the internal variable named self data. We delegate the length logic to the internal list.

In [module 1 line 10], we define the get item method with parameters self and index. This unlocks indexing syntax on a variable named custom list at position i.

In [module 1 line 11], we return the element at the given index from the internal variable named self data. If this method did not exist, indexing would fail.

In [module 1 line 14], we define the contains method with parameters self and item. This enables the in keyword.

In [module 1 line 15], we return whether the given item exists inside the internal variable named self data.

In [module 1 line 18], we define the iter method. This allows iteration.

In [module 1 line 19], we return the iterator of the internal variable named self data.

The key takeaway is powerful. Python does not care what your object is. It only cares what methods it implements. [slide 5 point 1] The input argument data to the constructor could be any type, as long as it supports the operations the class expects. [slide 5 point 2] In real systems, that data could be a file handle, a network socket, a database connection, a string buffer, a sensor stream, an API response stream, or even a mock object used for testing. Python treats them the same if they behave the same. [slide 5 point 3]

Now let us make another idea explicit.

When you write an addition between variables named x and y, Python translates that into an add method call on x with y as the argument. [slide 6 point 1]

When you index a variable named x at position i, Python performs a get item method call with i. [slide 6 point 2]

When you call the len method with a variable named x, Python calls the len method defined on that object. [slide 6 point 3]

Operators are not special. They are just method calls with elegant syntax. [slide 6 point 4]

That is why operator overloading exists, and also why it can be abused. A common mistake is overloading operators with surprising behavior. If addition suddenly deletes data instead of combining it, that is confusing. [slide 6 point 5]

Always ask whether a reasonable Python user would expect that operator to behave that way. [slide 6 point 6]

Now let us talk about duck typing.

Duck typing means that if something behaves like a certain kind of object, Python will treat it like that object. [slide 7 point 1]

Let us consider this code block.

In [module 2 line 1], we define a function named analyze with a parameter named source. Notice there are no type checks and no inheritance checks.

In [module 2 line 2], we assign a variable named lines to the result of converting source into a list. This requires that source is iterable. That is the only requirement.

In [module 2 line 3], we return a dictionary containing three pieces of information. We compute the line count by calling the len method on lines. We compute the word count by summing the lengths of each line after splitting it. We retrieve the first line by accessing the element at index zero of lines, if lines is not empty.

This function works with lists of strings, file objects, generators, or custom iterable objects. There are no explicit checks, only behavioral expectations. That is duck typing done right. [slide 7 point 2]

Now let us explore something interesting.

What if we want indexing, but we do not want to load everything into memory. Imagine a huge log file with millions of lines. Loading it fully just to read one line would be wasteful. [slide 8 point 1]

Let us consider this code block.

In [module 3 line 1], we define a class named LazyFileLoader. This object represents a view into a file, not the full file contents loaded into memory.

In [module 3 line 2], we define the init method with parameters self and filename.

In [module 3 line 3], we store the filename in a variable named self filename. No data is loaded yet.

In [module 3 line 6], we define the get item method with parameters self and index. This enables indexing on a variable named loader at position i.

In [module 3 line 7], we open the file using the stored filename.

In [module 3 line 8], we iterate through the file line by line, keeping track of the index.

In [module 3 line 9], we check whether the current index matches the requested index.

In [module 3 line 10], if it matches, we return the stripped version of that line.

In [module 3 line 11], if the loop completes without finding the index, we raise an IndexError. This is important because Python expects this exception for invalid indexing, just like a real list would do.

This object feels like a list, acts lazily, and uses constant memory. That is powerful. [slide 8 point 2]

Finally, let us look at a very common real world pattern.

We want interchangeable behaviors without conditional logic. Imagine an e commerce checkout system where different discount strategies can be applied. [slide 9 point 1]

Let us consider this code block.

In [module 4 line 1], we define a class named Discount10.

In [module 4 line 2], we define a method named apply with parameters self and price.

In [module 4 line 3], we return the price multiplied by zero point nine.

In [module 4 line 6], we define another class named Discount20.

In [module 4 line 7], we again define a method named apply with parameters self and price.

In [module 4 line 8], we return the price multiplied by zero point eight.

Both classes define the same apply method. That is the contract. Anything that has an apply method can be used as a discount strategy. [slide 9 point 2]

A checkout system does not need conditional logic based on types. It simply calls the apply method on the chosen strategy with the given price. No inheritance is required. No base class is needed. Just behavior. [slide 9 point 3]

Let us wrap this up.

Special methods are not advanced Python. They are Python itself. [slide 10 point 1]

They form the interface between your objects and Python syntax. [slide 10 point 2]

If you understand the data model, Python stops feeling magical. Your designs become cleaner. Your code becomes more expressive. [slide 10 point 3]

You will start recognizing these patterns everywhere, in frameworks, in libraries, and in production systems. [slide 10 point 4]

In the next lessons, this foundation will pay off again and again. [slide 10 point 5]

Great work today.
