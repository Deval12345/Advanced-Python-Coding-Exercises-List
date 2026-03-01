In the previous session we studied descriptors and saw how behavior can be attached to attributes and reused across classes. That idea of reusing behavior is not limited to attributes. In Python, functions themselves are objects. They can be stored, passed around, and chosen at runtime.

Today we focus on functions as first-class objects. We will see how passing a function as an argument lets you swap behavior without changing the caller, and how this pattern appears everywhere in real applications.

By the end of this lesson you will understand first-class functions, how to select behavior at runtime instead of long conditionals, and how callables fit into the same model.

We have seen special methods and descriptors control how objects behave. The next step is to treat behavior itself as data. In Python, a function is an object. You can assign it to a variable, put it in a list or dictionary, and pass it to another function. That is what we mean by first-class functions.

When you define a function, you create an object. That object can be assigned to another name. Calling either name invokes the same function. There is no special syntax. Functions behave like any other value. This is why Python can support strategies, callbacks, and plugins without extra machinery.

Here we define a function named greet that takes a parameter named name and returns a greeting string. We assign greet to a variable named func. We then call func with a string. Because func refers to the same function object as greet, the call produces the same result. The function was stored and invoked through a different name. That is first-class behavior.

The most powerful use of first-class functions is passing a function as an argument. The caller does not need to know which exact function runs. It just calls whatever it was given. That lets you plug in different behavior without changing the caller’s code. This is the strategy pattern we saw with objects; with functions it is even simpler.

Here we define a function named process that takes a list of numbers and a parameter named rule. Inside process we apply rule to each number and return a new list. We define small functions named addTax and applyDiscount. We call process with a list and addTax, then with the same list and applyDiscount. Because process accepts any function that takes one value and returns one value, we can switch behavior by passing a different function. No conditionals. Just different behavior injected from the outside.

Instead of long if-elif chains that choose what to do based on a string or type, you can store functions in a dictionary keyed by strategy name. When the user picks a strategy, you look up the function and call it. The logic is selected at runtime. New strategies are added by adding one entry to the dictionary, not by editing a big conditional.

Here we define three functions named flatFee, percentFee, and tieredFee. Each takes an amount and returns a modified amount. We store them in a dictionary named strategies under keys flat, percent, and tiered. We define a function named checkout that takes an amount and a strategy name, looks up the function in strategies, and calls it with the amount. Because the function is chosen by name, adding a new fee type means adding one function and one dictionary entry. The checkout code never needs to change.

In Python, anything you can call with parentheses is a callable. Functions are callables. Methods are callables. Classes are callables; calling a class creates an instance. Objects that implement the special dunder method call are also callables. So when we say pass a function, we often mean pass any callable. The receiver just calls it. That uniformity is part of what makes Python flexible.

Here we define a class named Multiplier. Inside it we implement the special dunder method call with two parameters, self and factor. The class stores a factor in its initializer. The call method returns the product of factor and the argument passed to the instance. Because the class defines the special dunder method call, we can create an instance and call it like a function. So objects can behave as callables. They are interchangeable with functions when the receiver only needs to call something with arguments.

Functions are first-class objects. They can be assigned, stored in data structures, and passed as arguments. Passing a function lets the caller inject behavior without conditionals. Dictionaries of functions enable runtime strategy selection. Callables include functions, methods, classes, and any object implementing the special dunder method call. This model is at the heart of extensible and testable Python code.
