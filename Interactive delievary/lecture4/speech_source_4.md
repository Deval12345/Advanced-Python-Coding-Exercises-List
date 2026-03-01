# Descriptors — Reusable Attribute Behavior

## Lesson Opening

In the previous session we learned how to intercept attribute access using the special dunder method getattr and getattribute. We used them for lazy loading and proxy objects when attributes were missing or virtual.

Today we look at a different tool. Descriptors attach behavior to attributes that are known at class design time. They let you reuse validation, conversion, and lazy loading across many classes without duplicating logic.

By the end of this lesson you will understand the descriptor protocol, how descriptors live on the class but operate on instances, and why they are used in real frameworks like ORMs and validation libraries.

------------------------------------------------------------------------

## Section 1 — Why Descriptors Exist

### CONCEPT 0.1

Last time we used getattr to handle attributes that did not exist yet. Descriptors solve a different problem. They control the behavior of attributes that are declared on the class. When you need the same validation or the same lazy loading logic on many classes, properties alone are not enough. Descriptors let you write that behavior once and reuse it.

### CONCEPT 1.1

A property is defined per class. If you need validated or computed behavior on an attribute in ten different classes, you would repeat the same logic ten times. Descriptors let you define that behavior in one place and attach it to any class attribute. So descriptors solve reuse at the attribute level, not just the method level.

------------------------------------------------------------------------

## Section 2 — The Descriptor Protocol

### CONCEPT 2.1

A descriptor is an object that implements the special dunder method get, or set, or delete, or a combination of them. When Python looks up an attribute and finds a descriptor on the class, it calls the descriptor’s get or set instead of returning the object directly. The descriptor receives the instance and the owner class so it can read or write instance state.

### CONCEPT 2.2

Descriptors live on the class, but they operate on instances. When you access an attribute on an instance, Python finds the descriptor on the class and calls its get with that instance. So one descriptor object serves all instances of the class, but each call gets the right instance. That separation is what makes reusable attribute behavior possible.

------------------------------------------------------------------------

## Section 3 — Reusable Validated Field

### CONCEPT 3.1

A validated field descriptor can enforce a range or type every time the attribute is set. The logic lives in the descriptor. Any class that uses that descriptor as a class attribute gets the same validation without writing it again.

### EXAMPLE 3.1

Here we define a class named ValidatedField. Inside it we implement the special dunder method set name to learn the attribute name from the class, and we store minimum and maximum in variables named minVal and maxVal. We implement the special dunder method get to return the value stored on the instance, and the special dunder method set to check that the value is in range before storing. Because ValidatedField implements the descriptor protocol, when we assign to an attribute that is a ValidatedField Python calls the descriptor’s set. The validation runs in one place and is reused for any class that uses ValidatedField.

### CONCEPT 3.2

The descriptor stores the backing value on the instance under a private name, often using the name set by set name. So the descriptor does not hold the value itself. It holds the rule. The instance holds the data. That keeps multiple instances independent.

------------------------------------------------------------------------

## Section 4 — Lazy Loading with a Descriptor

### CONCEPT 4.1

A descriptor can implement lazy loading. When the attribute is first read, the descriptor fetches or computes the value and caches it on the instance. Later reads return the cached value. This pattern appears in ORM fields that load from a database on first access.

### EXAMPLE 4.1

Here we define a class named DBField. Inside it we implement only the special dunder method get. The get method receives the instance and the owner. If the value is not yet in the instance cache, we load it and store it in the cache. Then we return the cached value. Because DBField is a descriptor, when we access user dot data Python calls get. The first access triggers the load; later accesses use the cache. The class stays clean and the lazy behavior is reusable.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 5.1

Descriptors attach behavior to attributes. They implement the special dunder method get and optionally set and delete. They live on the class and operate on instances. They enable reusable validation, lazy loading, and computed fields. Many real-world frameworks rely on descriptors for ORM fields, validation, and configuration. They are one of Python’s most powerful abstractions for attribute-level control.
