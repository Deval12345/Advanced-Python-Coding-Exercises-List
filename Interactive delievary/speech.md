Alright everyone let us get started



Today we are going to talk about something that quietly powers everything you do in Python but very few people learn it explicitly the Python data model also called magic methods



I want you to momentarily forget the idea that Python is just a scripting language with pleasant syntax Python is actually a framework and the interface of that framework is not functions it is objects



When you build a web application using a framework you do not directly control everything you plug into the framework by following certain rules Python works the same way You plug into its syntax by implementing specific methods



By the end of this lesson you should be able to look at normal Python syntax and mentally translate it into method calls Once that clicks Python starts to feel predictable instead of magical



Before we dive in here is what I want you to walk away with



By the end of this session you will understand Python as an object centric framework You will know how normal syntax like len method call with x or indexing x maps to special methods You will see duck typing in action You will learn how to design your own objects so they behave like built in types And you will understand operator overloading in a responsible way



If you have ever wondered how libraries make their objects feel native to Python today you will understand the mechanism behind that



Now here is a mindset shift



Python does not introduce new syntax for new features Instead Python says if your object implements certain methods I will allow it to participate in my syntax



When you write len method call with x Python internally checks whether variable x has a function len definition and if it does it calls it



That is why len works on strings lists tuples sets and dictionaries completely different types but the same protocol



When you use indexing on a variable like x Python translates that into a getitem method call with the index as argument



That is why indexing works on lists tuples strings and even numerical arrays



When you write a for loop over an object Python asks whether it can obtain an iterator using the iter method



That is why you can loop over files database cursors generators and any iterable object



When you check membership using the in keyword Python checks whether the object knows how to handle containment using the contains method



The visible syntax is simply a convenient surface The real work happens in special methods



Let us consider this code block



In \[module 1 line 1] we define a new class named CustomList This is simply a new type



In \[module 1 line 2] we define the function init with self and data This handles object initialization



In \[module 1 line 3] we assign the internal state by converting the incoming data into a list and storing it in variable self dot data This underscore name indicates internal state



In \[module 1 line 6] we define the function len This method is triggered when someone calls len on a CustomList object



In \[module 1 line 7] we return the length of the internal data by delegating to the built in len method



In \[module 1 line 10] we define the function getitem with self and index This enables indexing on the custom\_list variable



In \[module 1 line 11] we return the element from the internal list at the requested index



In \[module 1 line 14] we define the function contains with self and item This enables membership checks using the in keyword



In \[module 1 line 15] we return whether the item exists in the internal list



In \[module 1 line 18] we define the function iter This enables iteration over the object



In \[module 1 line 19] we return the iterator of the internal list



The key takeaway is that Python does not care about the concrete type of your object It only cares about the methods it implements The constructor argument data could be any type as long as it supports the required operations In real systems that could be a file handle a network socket a database connection a string buffer a sensor stream an API response stream or even a mock object used for testing Python treats them the same if they behave the same



When you write x plus y Python actually performs an add method call on x with y as argument



When you use indexing on x Python performs a getitem method call



When you call len on x Python performs a len method call



Operators are simply method calls with convenient syntax That is why operator overloading exists and also why it must be used responsibly



If an operator produces surprising behavior users become confused Always ask whether a reasonable Python user would expect that operator to behave in that way



Let us consider this code block



In \[module 2 line 1] we define a function named analyze with parameter source Notice that there are no type checks



In \[module 2 line 2] we convert source into a list and store it in variable lines This requires only that source is iterable



In \[module 2 line 3] we return a dictionary containing computed values



In \[module 2 line 4] we compute the line count using len method call with lines



In \[module 2 line 5] we compute the word count by summing the lengths of each line after splitting it



In \[module 2 line 6] we compute the first line by accessing index zero of lines if it exists otherwise returning None



This function works with lists of strings file objects generators and custom iterable objects It does not check types It relies purely on behavior This is duck typing done correctly



Let us consider this code block



In \[module 3 line 1] we define a class named LazyFileLoader This object represents a view into a file not the full contents



In \[module 3 line 2] we define the function init with self and filename



In \[module 3 line 3] we store the filename in the object without loading the file into memory



In \[module 3 line 6] we define the function getitem with self and index This enables indexing on the loader variable



In \[module 3 line 7] we open the file using the stored filename



In \[module 3 line 8] we iterate through the file line by line using enumeration



In \[module 3 line 9] we check whether the current position matches the requested index



In \[module 3 line 10] we return the stripped line when the index matches



In \[module 3 line 11] we raise an IndexError if the index is not found This matches Python expectations for invalid indexing



This object feels like a list but operates lazily and uses constant memory That is the strength of the data model



Let us consider this code block



In \[module 4 line 1] we define a class named Discount10



In \[module 4 line 2] we define a method apply with self and price



In \[module 4 line 3] we return the price multiplied by zero point nine



In \[module 4 line 6] we define another class named Discount20



In \[module 4 line 7] we define the same apply method



In \[module 4 line 8] we return the price multiplied by zero point eight



Both classes expose the same apply method That is the contract Any object that implements apply can serve as a discount strategy No inheritance is required No base class is necessary Only behavior matters



Let us conclude



Special methods are not an advanced feature They are Python itself They form the interface between your objects and Python syntax



When you understand the data model Python stops feeling magical Your designs become cleaner Your code becomes more expressive And you will begin to recognize these patterns everywhere in frameworks libraries and production systems



Great work today

