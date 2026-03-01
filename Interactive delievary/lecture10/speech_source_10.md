# Memory Layout and the special dunder method slots

## Lesson Opening

In the previous session we saw how iterators and generators keep memory bounded by producing values one at a time. Today we look at how Python stores object attributes in memory and how you can reduce memory use when you have very many instances of a class.

By the end of this lesson you will understand why normal objects have per-instance memory overhead, what the special dunder method slots does, and when it is appropriate to use it in real systems.

------------------------------------------------------------------------

## Section 1 — How Python Stores Attributes

### CONCEPT 0.1

We have been defining classes with attributes and storing data on instances. Each normal Python object has a dictionary that holds its attributes. That dictionary is flexible: you can add and remove attributes at runtime. But that flexibility has a cost. Each object carries the overhead of that dictionary plus the data structures that make it work. When you create millions of objects, that overhead adds up.

### CONCEPT 1.1

When you assign to an attribute on an instance, Python normally stores it in the instance dictionary. So every object has at least the memory for that dictionary. The dictionary itself has overhead: buckets, hashes, and pointers. For small objects with a few fixed attributes, the dictionary can use more memory than the actual data. So understanding this helps when you design systems that create many small objects.

------------------------------------------------------------------------

## Section 2 — What the special dunder method slots Does

### CONCEPT 2.1

The special dunder method slots is a class attribute that tells Python not to create a per-instance dictionary for that class. Instead, Python reserves a fixed number of slots for the attribute names you list. So each instance has a fixed layout. You cannot add new attributes at runtime on instances of that class. But each instance uses less memory because there is no dictionary. The trade-off is flexibility versus memory.

### CONCEPT 2.2

You set the special dunder method slots to a sequence of strings, usually a tuple, listing the names of the only attributes that instances of the class will have. Python then allocates exactly that many slots per instance. Attribute access is still by name, but the implementation uses the slot index instead of a hash table. So you get lower memory use and often slightly faster attribute access, at the cost of no dynamic attributes.

------------------------------------------------------------------------

## Section 3 — When to Use the special dunder method slots

### EXAMPLE 3.1

Here we define a class named Point. We set the special dunder method slots to a tuple containing the strings x and y. In the initializer we assign to the x and y attributes. Because we declared slots, instances of Point do not have a dictionary. They only have the two slots. So if we create many points, each one uses less memory than a normal object with a dictionary. We cannot add a new attribute like z on an instance; that would raise an error. So slots is for classes whose instances have a fixed set of attributes and where you care about memory or scale.

### CONCEPT 3.2

Use the special dunder method slots when you have a large number of instances and each has a fixed set of attributes. Data records, events, nodes in a graph, or cached entries are good candidates. Do not use it when you need to add attributes dynamically or when you use features that rely on the instance dictionary. It is an optimization and a design constraint, not a default for every class.

------------------------------------------------------------------------

## Final Takeaways

### CONCEPT 4.1

Normal Python objects store attributes in a per-instance dictionary, which has overhead. The special dunder method slots declares a fixed set of attribute names and removes the per-instance dictionary. That reduces memory use and can speed up attribute access, but you cannot add new attributes on instances. Use slots for high-volume, fixed-schema data; keep normal classes when you need dynamic attributes.
