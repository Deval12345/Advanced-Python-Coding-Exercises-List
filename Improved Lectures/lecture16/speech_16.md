# Speech — Lecture 16: Abstract Base Classes (ABCs) & collections.abc — Validated Duck Typing

---

We have spent the last several lectures building up a toolkit of Python's most powerful runtime behaviors. Duck typing gave us flexibility. Descriptors gave us control over attribute access. The dunder method getattr gave us dynamic dispatch. But here is a question that should bother you — what happens when a class that is supposed to implement an interface simply forgets a method? Who catches that?

Today we answer that question, and the answer is Abstract Base Classes.

---

Let me paint a picture of the problem. You are building a plugin system for a data pipeline. Every plugin is supposed to implement a cleanup method that releases resources when the job is done. You write the base class. You document it. You tell your team. Six months later, someone ships a plugin without cleanup. The plugin passes import checks. It passes initialization. It passes every unit test that doesn't explicitly call cleanup. And then, deep in production, after eight hours of processing, the pipeline tries to clean up, hits an AttributeError, and everything crashes.

That is the silent failure of duck typing at scale. Duck typing is powerful but it is quiet. It trusts you. And trust, in a large codebase with many contributors, is a liability.

Abstract Base Classes solve this with one simple principle — early failure. The moment you try to instantiate a class that inherits from an ABC but has not implemented all required abstract methods, Python raises a TypeError. It names every missing method. You find the bug at object creation, not six hours into a production run.

---

So how do you define an ABC? You import ABC and abstractmethod from the abc module. Your base class inherits from ABC. Every method that a subclass must implement gets decorated with the abstractmethod decorator. That is it. From this point forward, Python enforces the contract for you.

Now here is where it gets really interesting. An ABC can also define concrete methods — regular methods with full implementations. Subclasses inherit these for free. This is the template method pattern. You define the algorithm structure in the base class, and subclasses only fill in the steps.

Think about what that enables. Your base class can define a method called runPipeline. It calls load, then process, then save in order. Load, process, and save are abstract — every subclass must implement them. But runPipeline itself is concrete. Every subclass gets it for free, and it always runs the steps in the right order. Your team can never accidentally skip a step or run them out of sequence. The algorithm is locked in. Only the implementation details vary.

This pattern is everywhere in major Python frameworks. Scikit-learn uses it in BaseEstimator. Django uses it in its View class. Apache Beam uses it throughout its pipeline API. These frameworks are not doing anything magical — they are using the same ABC mechanism you have available right now.

---

Now I want to address a question that comes up constantly — what is the difference between an ABC and a Protocol?

They are both tools for defining interfaces, but they work in fundamentally different ways.

ABCs use inheritance and enforce at runtime. You subclass the ABC, and Python checks that you have implemented every abstract method when you try to create an instance. The check happens right there in your running program.

Protocols use structural subtyping. A class satisfies a Protocol if it has the right methods and attributes, regardless of whether it inherits from anything. The checking happens at static analysis time, with tools like mypy, not at runtime in your program.

Choose ABCs when you want explicit inheritance, runtime enforcement, and the ability to provide mixin methods. Choose Protocols when you are working with duck typing and want static type checker support, or when you cannot modify the classes you are working with.

---

Now let me introduce collections dot abc, because this is where ABCs go from a useful pattern to an essential tool.

Python's standard library ships with a full set of ABCs for all built-in container types. Sequence, Mapping, Set, Iterator, Callable, MutableSequence, MutableMapping — they are all there. And the reason you care is not just isinstance checking. It is the mixin methods.

When you inherit from collections dot abc dot Mapping, you only need to implement three methods. The special dunder method getitem, the special dunder method iter, and the special dunder method len. Three methods. In return, you get the following for free: get, keys, values, items, the special dunder method contains, and the special dunder method eq. Python derives all of them from your three implementations.

This is an incredible deal. You write three methods. You get a complete, correct, fully-featured mapping type in return. The mixin logic lives in the ABC. You get it for free by declaring the relationship.

And critically — isinstance checks against Mapping now work for your class. Any code that asks whether something is a Mapping will recognize your class as one, because you declared the contract and fulfilled it.

---

There is one more mechanism I want you to understand, and it is called virtual subclassing. You register an existing class with an ABC using the register method. After that, isinstance and issubclass return True for that class and ABC pair, even though the class never inherited from the ABC.

This is essential for third-party integration. Imagine you are using a library written five years ago. It has a class that implements all the methods your ABC requires, but it predates your ABC entirely. You cannot go modify that library. You cannot make it inherit from your ABC. But you can call your ABC dot register and pass it that class. From that point forward, your code recognizes it as a valid implementation.

The trade-off is important to understand. Register does not verify that the class actually implements the required methods. It takes you at your word. So virtual subclassing is a tool for declaring compatibility, not for enforcing it. Use it when you have inspected the class and confirmed it meets the interface.

---

Let me close with the full picture of where ABCs shine.

Plugin systems are the canonical use case. You define the interface. Every plugin that inherits from your ABC and tries to instantiate will be checked immediately. Missing a method means failing right at startup, not at the worst possible moment in production.

Framework APIs are the second major use case. You want to give users a base class that handles the complex parts — routing, error handling, lifecycle management — while they fill in only the business logic. The template method pattern with ABCs is exactly the right tool.

And collections dot abc is your tool whenever you are implementing a custom container type. Inherit from the right ABC, implement the required minimum, and get a complete container interface for free.

ABCs enforce at instantiation, not at use. They fail early, loudly, and with a clear error message naming exactly what is missing. After years of Python, that clarity is genuinely one of the most valuable things the language offers.

---
