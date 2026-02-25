Alright everyone, let us get started.
[slide 1 point 1]

Today we are going to talk about something that quietly powers
everything you do in Python, but very few people ever learn it
explicitly. The Python Data Model, also called magic methods or dunder
methods.
[slide 1 point 2]

For a moment, forget the idea that Python is just a scripting language
with nice syntax. Python is actually a framework, and the real interface
of that framework is objects.
[slide 1 point 3]

Every time you use addition, length calculation, indexing, or a loop,
you are not just using syntax. You are interacting with object behavior.
[slide 1 point 4]

By the end of this lesson, you should be able to look at normal Python
syntax and mentally translate it into method calls. Once that clicks,
Python feels predictable instead of magical.
Before we dive in, here is what I want you to walk away with.
[slide 2 point 1]

You will understand Python as an object centric framework. You will see
how normal syntax maps to special methods. You will see duck typing in
action. You will learn how to design your own objects so they behave
like built in types. And you will understand operator overloading in a
responsible way.
Let us begin.
Here is a mindset shift.
[slide 2 point 2]

Python does not add new syntax for new features. Instead, Python says
that if your object implements certain methods, it can participate in
the language syntax.
[slide 2 point 3]

When you write length of x, Python internally checks whether the object
has a special dunder method len and if it exists, it calls it.
[slide 2 point 4]

Suppose we create a small list of numbers. If we apply length to it,
Python asks the object for its size. If the list has three elements, it
returns three.
[slide 2 point 5]

When you use indexing on x with an index value, Python internally calls
the special dunder method getitem.
[slide 3 point 1]

Imagine we have a word stored in a variable named x. If we access
position one, Python forwards that request to the object. The object
decides what element to return.
[slide 3 point 2]

When you use a for loop over an object, Python checks whether the object
implements the special dunder method iter.
[slide 3 point 3]

Think of it this way. Python just needs a way to fetch elements one by
one. If the object can provide an iterator, the loop works naturally.
[slide 3 point 4]

When you check whether something is inside an object using the in
keyword, Python looks for the special dunder method contains.
[slide 3 point 5]

For example, suppose we have a container with three numbers. When we ask
whether two is inside it, Python delegates that question to the object
itself. The object decides how to answer.
[slide 3 point 6]

The syntax is only a layer of sugar. The real work happens inside
special methods.
Now let us make this concrete.
We are creating a class called CustomList. Internally it stores data in
a real list, but externally Python will treat it like a native
container.
[module 1 line 1]

We define a class named CustomList. This is a brand new type. Like
defining a new tool, it does nothing until we give it behavior.
[module 1 line 2]

We define the initializer method. This method runs when we create an
instance of CustomList. For example, when we create a variable named c
using CustomList with some numbers, this method prepares the internal
state.
[module 1 line 3]

Inside the initializer, we store the incoming data as a list in an
internal variable named data. Think of this as the engine under the
hood. The outside world does not interact with it directly.
[module 1 line 6]

We define the special dunder method len. This method is called whenever
someone asks for the length of a CustomList instance.
Suppose we create a variable named c with three elements. When we ask
for its length, Python calls this method. If there are three elements,
it returns three.
[module 1 line 9]

We define the special dunder method getitem. This enables indexing
behavior.
Imagine we create a variable named c with values ten, twenty, and
thirty. If we access index one, Python calls this method. It returns the
element at that position.
[module 1 line 12]

We define the special dunder method contains. This enables the in
keyword.
Suppose we check whether two exists inside a variable named c. Python
calls this method. The method decides whether to return true or false.
[module 1 line 15]

We define the special dunder method iter. This allows the object to be
used in a loop.
Imagine we loop over a variable named c. Python requests an iterator
from this method. Then it pulls values one by one until the data is
exhausted.
[slide 4 point 1]

The key takeaway is simple. Python does not care about the actual type.
It only cares about which methods are implemented. If two objects
implement the same behavior, Python treats them similarly.
Now let us talk about operators.
[slide 4 point 2]

When you write addition between two variables named a and b, Python
actually calls the special dunder method add on the left side object.
Suppose a variable named a stores five and a variable named b stores
ten. When we add them, Python asks the object a how to add b. The result
depends on how that method is implemented.
[slide 4 point 3]

Operators are just method calls with elegant syntax.
Let us now discuss duck typing.
[slide 5 point 1]

Duck typing means that if something behaves like a certain kind of
object, Python will treat it that way.
We define a function named analyze that accepts a parameter named
source.
[module 2 line 1]

This function does not check the type of source. It simply uses it.
[module 2 line 2]

We convert source into a list. This means source must be iterable.
For example, suppose source is a list of text lines. Converting it to a
list simply gathers those lines. If source were a file object, it would
still work as long as it can be iterated.
[module 2 line 3]

We compute the length of the lines. This relies on the special dunder
method len.
If there are five lines, the count becomes five. The object just needs
to support length behavior.
[module 2 line 4]

We split each line into words and count them. This assumes each element
behaves like a string.
For instance, if a line contains two words separated by space, splitting
produces two pieces. Counting them increases the word total.
[slide 5 point 2]

This function works with lists of strings, file objects, generators, or
any custom object that supports iteration and string like elements.
That is duck typing done properly. No type checks, only behavior.
Now let us explore lazy indexing.
We define a class named LazyFileLoader.
[module 3 line 1]

This class represents a view of a file, not the file content in memory.
[module 3 line 2]

We define the initializer. It stores a variable named filename. No file
content is loaded yet.
[module 3 line 5]

We define the special dunder method getitem. This enables indexing on a
LazyFileLoader instance.
Suppose we create a variable named loader with a file name and access
index zero. Python calls this method. The method opens the file and
reads line by line until it reaches the requested index.
If the index does not exist, it raises an IndexError. This is important
because Python expects this exact exception for invalid indexing.
Imagine a list with two elements. If we try to access position five,
Python raises IndexError. Our object follows the same contract.
[slide 7 point 1]

This object feels like a list, but it loads data lazily and uses
constant memory.
Finally, let us look at a strategy pattern.
We define a class named Discount10.
[module 4 line 1]

Inside it, we define a method named apply.
[module 4 line 2]

This method takes a variable named price and returns a reduced value.
Suppose the price is one hundred. Applying this method returns ninety.
The logic defines the discount.
We define another class named Discount20.
[module 4 line 6]

It also defines a method named apply.
[module 4 line 7]

If the price is one hundred, this version returns eighty.
[slide 9 point 1]

The key idea is that any object that defines an apply method with the
expected behavior can act as a strategy.
For example, we create a variable named strategy using Discount10. When
we call apply with one hundred, the object handles the calculation.
No inheritance is required. No base class is required. Only behavior
matters.
Let us wrap up.
[slide 10 point 1]

Special methods are not advanced Python. They are Python itself.
[slide 10 point 2]

They form the interface between your objects and the language syntax.
[slide 10 point 3]

If you understand the data model, Python stops feeling magical. Your
designs become cleaner. Your code becomes more expressive.
From now on, whenever you see syntax, ask yourself what method Python is
actually calling.
Great work today.
