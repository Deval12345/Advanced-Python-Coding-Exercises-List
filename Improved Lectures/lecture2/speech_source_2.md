# Python Data Model Part 2 — Custom Containers and Operator Overloading

## Transition from Lecture 1

### CONCEPT 0.1

Welcome back.

In our last session, we opened the hood on Python and discovered something important. Python syntax is not magic. Every time you call len on an object, every time you use square brackets, every time you write a for loop, Python is quietly calling a special method on your object. We saw how the special dunder method len, the special dunder method getitem, the special dunder method iter, and the special dunder method contains each let an object participate in one piece of syntax.

But here is what we did not do. We never combined them. We treated each method like an isolated trick. Today, that changes.

Today we are going to take those individual methods and weave them together into a complete, native-feeling container. And then we are going to push further into operator overloading, where your objects start responding to plus signs, print statements, and equality checks. By the end of this lecture, you will be able to build objects that feel as natural to use as a Python list or dictionary, and you will understand why that matters in real production systems.

------------------------------------------------------------------------

## Section 1 — Building a Complete Custom Container

### CONCEPT 1.1

Let us start with the big picture. In Lecture 1, we saw each special method in isolation. But in real systems, you rarely need just one. Think about an e-commerce platform. You have an inventory system holding thousands of products. You need to know how many products you have. You need to look up a product by its position or identifier. You need to loop through all products. And you need to check whether a specific product exists in inventory.

That is not one special method. That is four of them working together. When a class implements the special dunder method len, the special dunder method getitem, the special dunder method iter, and the special dunder method contains, it becomes a complete container. Python treats it like a first class citizen.

### EXAMPLE 1.1

Here we define a class named ProductInventory.

Inside the special dunder method init, we store a list of product records inside a variable named productList.

The special dunder method len returns the number of products by returning the length of productList.

The special dunder method getitem accepts a parameter named index and returns the product at that position in productList.

The special dunder method contains accepts a parameter named productName and returns True if any product in productList has a matching name.

The special dunder method iter returns an iterator over productList.

Because the class implements all four methods, we can call len on an instance, access products with square bracket syntax, use the in keyword to check membership, and loop through every product with a for loop. The object behaves exactly like a built in collection.

------------------------------------------------------------------------

### CONCEPT 1.2

Now, why does this matter? Why not just use a plain list?

Here is the real answer. A plain list gives you no control. Anyone can append anything. Anyone can delete items. There is no validation, no business logic, no domain awareness. Your inventory becomes a bag of random objects.

A custom container wraps that raw data with meaning. You control what goes in. You control how lookups work. You can add logging, validation, and access control. In financial systems, transaction batches need to enforce ordering and prevent duplicates. In machine learning pipelines, feature stores need to validate data types before they serve features to a model. The container is not just storage. It is a boundary that protects the integrity of your data.

And here is the critical point. Because your container implements the same special methods as a built in list, every piece of Python that already works with lists, every for loop, every len call, every membership check, it all works with your object too. You get domain safety without sacrificing compatibility.

------------------------------------------------------------------------

## Section 2 — Why Custom Containers Matter in Real Systems

### CONCEPT 2.1

Let us make this concrete with a second scenario. Imagine you are working on a financial trading platform. You have a batch of transactions that need to be processed in order. You cannot just throw them into a plain list because you need to enforce rules. Every transaction must have a valid timestamp. No duplicate transaction identifiers are allowed. The batch must be immutable once finalized.

A custom container gives you all of that. The special dunder method len tells the system how many transactions are queued. The special dunder method getitem lets compliance teams inspect individual transactions by position. The special dunder method contains lets the system check whether a transaction has already been submitted. And the special dunder method iter lets the processing engine walk through transactions in sequence.

Without this, you end up writing validation logic scattered across fifty different modules, each one checking the same rules independently. With a custom container, the rules live in one place.

### EXAMPLE 2.1

Here we define a class named TransactionBatch.

Inside the special dunder method init, we store a list of validated transaction records inside a variable named transactionList.

The special dunder method len returns the count of transactions.

The special dunder method getitem accepts a parameter named position and returns the transaction at that position.

The special dunder method contains accepts a parameter named transactionId and checks whether any transaction in the list matches that identifier.

The special dunder method iter returns an iterator over the transactionList.

Because TransactionBatch implements all four container methods, the processing engine can use it exactly like a list, but with all business rules enforced internally.

------------------------------------------------------------------------

## Section 3 — Operator Overloading

### CONCEPT 3.1

Now let us shift to operator overloading. We touched on this briefly in Lecture 1, but today we go deeper.

Operator overloading means giving your custom objects the ability to respond to Python operators, the plus sign, the equality check, the print function. The mechanism is the same as before. Python sees an operator, and it translates that into a special method call.

The plus operator maps to the special dunder method add. The str function and print map to the special dunder method str. And the repr function, which gives you the developer-facing representation, maps to the special dunder method repr.

Let us see why this matters. Imagine you are building a pricing engine. You have price objects that represent amounts in a specific currency. When an analyst writes price one plus price two, that should produce a new price object with the combined total. Not a number. Not a string. A proper price with currency information preserved.

### EXAMPLE 3.1

Here we define a class named PriceAmount.

Inside the special dunder method init, we store two values. A numeric value inside a variable named amount, and a string inside a variable named currency.

The special dunder method add accepts a parameter named otherPrice. It first checks whether the currencies match. If they do not, it raises a ValueError. If they match, it returns a new PriceAmount with the combined total and the same currency.

The special dunder method repr returns a developer-friendly string showing the class name, the amount, and the currency.

The special dunder method str returns a user-friendly string showing the currency symbol followed by the amount.

Because this class defines the special dunder method add, we can write priceOne plus priceTwo and get a meaningful result. Because it defines the special dunder method repr, debugging in the console shows useful information. And because it defines the special dunder method str, printing a price shows exactly what a user would expect.

------------------------------------------------------------------------

## Section 4 — Responsible Operator Overloading

### CONCEPT 4.1

But here is where we need to be careful. Operator overloading is powerful, but it can also be dangerous.

The rule is simple. Overload an operator only when the operation has an obvious, intuitive meaning for your domain. Price plus price makes sense. Everybody understands that. But what would it mean to add two user objects? User plus user? That is confusing. What would it mean to multiply a database connection by three? That is nonsensical.

When operators are overloaded without clear semantics, the code becomes harder to read, not easier. New developers join the team, see a plus sign, and assume it means numeric addition. But instead it is doing something completely unexpected, like merging two configurations or concatenating log entries.

The consequence is real. Misleading operator overloading leads to bugs that are extremely hard to find because the code looks correct at a glance. The syntax is familiar, but the behavior is surprising. That is the worst kind of bug.

### CONCEPT 4.2

So here is the guideline. Ask yourself, if a new developer reads this line of code with no context, would they correctly guess what the operator does? If the answer is yes, overload it. If the answer is maybe or no, use a named method instead.

For our PriceAmount example, priceOne plus priceTwo is obvious. But for combining two inventories, inventoryOne dot mergeWith parenthesis inventoryTwo is clearer than using the plus sign.

Named methods communicate intent explicitly. Operators communicate intent implicitly. Use implicit communication only when the meaning is universally clear.

------------------------------------------------------------------------

## Section 5 — Putting It All Together

### CONCEPT 5.1

Let us combine everything from today. We will build a feature store for a machine learning system. This is a container that holds named features, each with a name, a data type, and a value. Data scientists need to check how many features exist, look up features by index, check if a feature is present, iterate through all features, and combine two feature stores into one.

### EXAMPLE 5.1

Here we define a class named FeatureStore.

Inside the special dunder method init, we store a list of feature records inside a variable named featureList.

The special dunder method len returns the number of features.

The special dunder method getitem accepts a parameter named index and returns the feature at that position.

The special dunder method contains accepts a parameter named featureName and checks whether any feature in the list has that name.

The special dunder method iter returns an iterator over featureList.

The special dunder method add accepts a parameter named otherStore and returns a new FeatureStore containing the combined features from both stores.

The special dunder method repr returns a string showing FeatureStore with the count of features.

This is the complete picture. Container behavior plus operator overloading. The FeatureStore is iterable, indexable, searchable, and combinable. It works seamlessly with every Python construct that expects a collection, while enforcing domain rules internally.

------------------------------------------------------------------------

## Closing

### CONCEPT 6.1

Let us recap what we covered today. We took the individual special methods from Lecture 1 and combined them to build complete custom containers. We saw why this matters in real systems, from e-commerce inventory to financial transaction batches to machine learning feature stores. We explored operator overloading with the special dunder method add, the special dunder method repr, and the special dunder method str. And we discussed responsible overloading, the idea that operators should only be used when their meaning is immediately obvious.

In the next lecture, we will look at how Python formalizes these behavioral expectations through protocols, how duck typing scales to larger systems, and how informal interfaces keep Python flexible without sacrificing clarity.
