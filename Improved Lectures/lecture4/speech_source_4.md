# Protocols in Python Part 2 — Behavior-Driven Design and Protocol Extensibility

## Lesson Opening

In the previous session, we established that Python uses informal interfaces and duck typing to decide compatibility. Objects participate in systems by providing the right methods, not by inheriting from specific base classes. That insight gave us a foundation.

Today we push further. We see how this philosophy enables powerful design patterns used in real production systems. Behavior-driven design, protocol-based extensibility, and the strategy pattern without inheritance. These are the patterns that make plugin architectures, web frameworks, and data processing pipelines possible in Python.

By the end of this lesson, you will understand how objects become interchangeable through shared method signatures, how the file-like protocol enables extensibility, how to implement the strategy pattern without class hierarchies, and why Python frameworks rely on protocols to stay flexible.

------------------------------------------------------------------------

## Section 1 — Transition from Lecture 3

### CONCEPT 0.1

In the previous session, we explored informal interfaces and duck typing. We saw that Python checks for the presence of methods at runtime rather than verifying inheritance. An object that defines the special dunder method len behaves like a container. An object that defines write behaves like a writable destination. Python never asks what class you are. It asks what you can do. Today we take that principle and apply it to real-world design. We look at how protocols let you build systems where components are swappable, extensible, and decoupled from each other.

------------------------------------------------------------------------

## Section 2 — Behavior-Driven Design

### CONCEPT 1.1

Behavior-driven design means you define compatibility through method signatures, not through inheritance trees. In a traditional object-oriented language, you might create a base class called Processor with an abstract method called process, and then force every processor to inherit from it. In Python, you skip the base class entirely. If two objects both have a method named process that accepts the same kind of input, they are interchangeable. The system that calls process does not care about ancestry. It cares about capability.

### CONCEPT 1.2

Think about why this matters in practice. Imagine you are building a data pipeline. You have one component that cleans text, another that tokenizes it, and a third that scores sentiment. In an inheritance-based design, all three would inherit from a common base class. But what if the text cleaner comes from a third-party library and the sentiment scorer is a machine learning model with its own class hierarchy. Forcing them into a shared inheritance tree becomes painful. With behavior-driven design, each component just needs a method named process that takes data in and returns data out. No shared ancestor required.

### EXAMPLE 1.1

Here we define two classes, TextCleaner and SentimentScorer. Inside TextCleaner, we define a method named process that receives a parameter named textData. The method returns the textData converted to lowercase with extra whitespace removed. Inside SentimentScorer, we define a method named process that receives a parameter named textData. The method returns a dictionary with the textData and a computed score. We then define a function named runPipeline that receives a parameter named steps and a parameter named inputData. The function loops through each step in steps and calls the process method, passing inputData each time. Because both TextCleaner and SentimentScorer define a method named process, they are interchangeable inside runPipeline. No shared base class is needed.

------------------------------------------------------------------------

## Section 3 — Protocol-Based Extensibility

### CONCEPT 2.1

Protocol-based extensibility means new components can join a system without modifying the system itself. The system defines the expected behavior. Any object that provides that behavior is accepted. This is how Python handles file-like objects. The built-in print function can write to the console, to a file, to a network socket, or to a custom logger. It does not check the type of its destination. It checks whether the destination has a method named write.

### CONCEPT 2.2

This is the pattern behind plugin architectures. A web framework defines that a middleware must have a method named handle that receives a request and returns a response. Any object that provides handle is a valid middleware. You do not register classes. You do not inherit from a framework base class. You just provide the right method. New middleware can be added, removed, or swapped without touching the framework code.

### EXAMPLE 2.1

Here we define three classes, ConsoleOutput, FileOutput, and JsonOutput. Inside ConsoleOutput, we define a method named write that receives a parameter named messageText. The method prints the messageText to the console. Inside FileOutput, we define a method named write that receives a parameter named messageText. The method appends the messageText to a file stored in a variable named filePath. Inside JsonOutput, we define a method named write that receives a parameter named messageText. The method converts the messageText into a JSON structure and prints it. We then define a function named logMessage that receives a parameter named destination and a parameter named messageText. The function calls the write method on destination with messageText. Because all three classes define write, logMessage works with any of them. Adding a new output format means creating a new class with a write method. The logMessage function never changes.

------------------------------------------------------------------------

## Section 4 — Strategy Pattern Without Inheritance

### CONCEPT 3.1

The strategy pattern lets you swap behavior at runtime. In classical object-oriented programming, you implement this with an abstract strategy base class and concrete subclasses. Python does not need that. Because Python checks behavior at runtime, you can pass any object that has the right method. You can even pass a plain function. The strategy pattern in Python is just an object or a function with the expected signature.

### CONCEPT 3.2

Consider a pricing engine for an online store. You want to apply different discount strategies depending on the customer type, the time of year, or a promotional campaign. With inheritance, you would define a DiscountStrategy base class, then create RegularDiscount, HolidayDiscount, and LoyaltyDiscount subclasses. With protocols, you just need objects that have a method named calculate. You can swap strategies at runtime without rebuilding the object hierarchy.

### EXAMPLE 3.1

Here we define three classes, RegularDiscount, HolidayDiscount, and LoyaltyDiscount. Inside RegularDiscount, we define a method named calculate that receives a parameter named basePrice. The method returns basePrice multiplied by zero point nine five, applying a five percent discount. Inside HolidayDiscount, we define a method named calculate that receives a parameter named basePrice. The method returns basePrice multiplied by zero point eight, applying a twenty percent discount. Inside LoyaltyDiscount, we define a method named calculate that receives a parameter named basePrice. The method returns basePrice multiplied by zero point seven, applying a thirty percent discount. We then define a class named PricingEngine. Inside it, we store the current strategy in a variable named activeStrategy. We define a method named setStrategy that receives a parameter named strategy and assigns it to activeStrategy. We define a method named computePrice that receives a parameter named basePrice and calls calculate on activeStrategy. Because all three discount classes define calculate, PricingEngine can swap between them at runtime without any inheritance hierarchy.

------------------------------------------------------------------------

## Section 5 — How Python Frameworks Leverage Protocols

### CONCEPT 4.1

Real Python frameworks are built on this exact philosophy. Django uses protocols for its ORM. When you define a model with fields, Django checks whether each field object provides methods like contribute to class and get prep value. Any object that implements those methods works as a field. You can write custom fields without inheriting from a Django base class, as long as you provide the expected protocol.

### CONCEPT 4.2

Flask uses protocols for route handlers. A route handler is any callable. It can be a function, a method, or an object that defines the special dunder method call. Flask does not check your type. It checks whether it can call you. Pandas uses protocols for custom data access. If an object defines the special dunder method getitem, pandas can index into it. If it defines the special dunder method iter, pandas can loop over it. These frameworks do not define rigid class hierarchies. They define behavioral contracts and trust objects to fulfill them.

### CONCEPT 4.3

This is why Python is uniquely suited for glue code and integration. When you connect a database adapter to a web framework, or a machine learning model to a data pipeline, you do not need adapter classes or complex wrappers. You need objects that speak the same behavioral language. Protocols are that language.

------------------------------------------------------------------------

## Section 6 — What Breaks with Inheritance Hierarchies

### CONCEPT 5.1

Let us consider what happens when you force inheritance instead. Suppose you build a notification system. You create a base class named Notifier with a method named send. You create EmailNotifier, SlackNotifier, and SMSNotifier as subclasses. This works until your team adds a WebhookNotifier from an external library that already has its own class hierarchy. Now you must either wrap it in an adapter, use multiple inheritance, or refactor everything. With protocols, the WebhookNotifier just needs a method named send. It joins the system immediately. No adapters, no refactoring, no fragile inheritance chains.

### CONCEPT 5.2

Inheritance hierarchies also create tight coupling. If you change the base class signature, every subclass must update. If you add a required method to the base, every subclass that does not implement it breaks. Protocols avoid this entirely. Each object is responsible for its own behavior. Changes to one object do not cascade through a class tree.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 6.1

Protocols enable behavior-driven design where objects become interchangeable through shared method signatures. Protocol-based extensibility lets new components join a system without modifying existing code. The strategy pattern works in Python without inheritance because runtime behavior checking replaces compile-time type checking. Real frameworks like Django, Flask, and pandas are built on this philosophy. Protocols are what make Python uniquely extensible and what allow you to write systems that grow without breaking.
