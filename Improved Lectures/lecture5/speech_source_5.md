# Speech Source — Lecture 5
# Memory Layout and Python Object Model: Identity, Mutability, and Copying

---

## CONCEPT 0.1 — Transition from Protocols

Over the last four lectures we have been building objects that respond to Python syntax through special methods and protocols. We defined custom containers, we implemented operator overloading, we used duck typing to write flexible, behavior-driven systems. Our objects now feel native to Python.

But here is a question that underlies every single line we have written: how does Python actually store and share these objects in memory? When you pass a container to a function, what exactly is being passed? When two variables refer to the same thing, what does "the same thing" mean?

This lecture answers those questions. We are going into Python's reference semantics — the memory model that governs how every object is stored, shared, and copied. Understanding this model will make you a more precise programmer and will save you from an entire class of subtle, hard-to-reproduce bugs that appear in real production systems.

---

## CONCEPT 1.1 — Variables Are References, Not Values

Let us start with the most important insight of this lecture, and arguably one of the most important insights in Python overall.

In Python, a variable is not a box that contains a value. A variable is a label attached to an object.

When you come from C or Java, you are used to thinking of a variable as a location in memory that holds a value. When you write int x equals 5 in C, x is a memory slot, and 5 is stored directly inside it. Python is completely different.

When Python executes the assignment productList equals a list containing laptop, phone, and tablet, Python does two things in sequence. First, it creates a list object somewhere in memory — that list object exists independently. Second, it attaches the name productList to that object. The name is not the object. The name is a label pointing to the object.

Now here is the consequence: if you write catalogAlias equals productList, Python does not create a new list. Python creates a second label, catalogAlias, and attaches it to the exact same object in memory. You now have two names pointing to one list.

This is called aliasing. Two labels on one object.

### EXAMPLE 1.1 — Aliasing with a Product List

Here we define a list of product names and assign it to productList.

```python
productList = ["laptop", "phone", "tablet"]
catalogAlias = productList

catalogAlias.append("monitor")

print(productList)   # ["laptop", "phone", "tablet", "monitor"]
print(catalogAlias)  # ["laptop", "phone", "tablet", "monitor"]
```

Inside this example, we assign catalogAlias to productList — this is not a copy, it is a second name for the same object. We then call append on catalogAlias, adding the string monitor. When we print productList, we see all four items. The append happened to the object, not to a particular name. Both names show the updated state because both names point to the same object.

---

## CONCEPT 1.2 — Why This Matters in Industry

Consider a web application running on Django or Flask. A developer stores a user's session configuration in a dictionary — things like their preferred language, their access level, their timeout settings. That dictionary gets passed to multiple functions: one to render the page, one to validate permissions, one to log activity.

If any of those functions modifies the dictionary, every other function sees the change. And if the original dictionary stored in the session cache also holds that reference, the change persists across requests. The bug manifests as: behavior that changes depending on call order, or state that leaks between requests, or a permission level that was supposed to be read-only but was accidentally elevated.

The root cause is always the same: the developer did not realise they were passing a reference, not a copy.

Understanding aliasing is the foundation for writing code that behaves predictably at scale.

---

## CONCEPT 2.1 — Identity vs Equality: is vs ==

Python gives us two different ways to compare two variables, and they answer two completely different questions.

The double-equals operator calls the special dunder method eq on the left-hand object, passing the right-hand object as the argument. It is asking: are these two objects equal in value? Equal in content? The answer depends entirely on how dunder method eq is implemented.

The is operator asks a completely different question: are these two names attached to the exact same object in memory? It does not call any method. It compares memory addresses directly.

### EXAMPLE 2.1 — Identity vs Equality

Here we define two lists with identical content, then create an alias of the first.

```python
firstList = [1, 2, 3]
secondList = [1, 2, 3]
aliasOfFirst = firstList

print(firstList == secondList)   # True  — same values
print(firstList is secondList)   # False — different objects in memory
print(firstList is aliasOfFirst) # True  — same object in memory
```

Inside this example, firstList and secondList both contain the integers 1, 2, and 3. So double-equals returns True — they are value-equal. But they are not the same object. They were created separately, so they live at different memory addresses. The is operator returns False for those two. However, aliasOfFirst was created with aliasOfFirst equals firstList — that means aliasOfFirst is just another label on the exact same object as firstList. So is returns True for those two.

---

## CONCEPT 2.2 — The None Rule: Always Use is None

Here is a rule that every professional Python programmer follows: never write double-equals None. Always write is None.

The reason is subtle but important. The double-equals operator calls dunder method eq. A custom class can override dunder method eq to return True under any condition — including when compared to None. So if you write objectInstance double-equals None, and the author of that class made dunder method eq return True for everything, your None check will silently fail. You will believe you are handling a None case, but you are not.

The is operator does not call any method. It is a direct memory-address comparison. None is a singleton in Python — there is exactly one None object in the entire runtime. So is None reliably asks: is this the actual None object? No custom class can interfere with that check.

This comes directly from Fluent Python, around page 332. Use is None. Use is not None. Never use double-equals None.

---

## CONCEPT 3.1 — Mutability and Aliasing in Detail

Not all objects behave the same way when you have multiple names pointing to them. The behavior depends on whether the object is mutable or immutable.

Mutable objects are objects whose internal state can be changed in-place: lists, dictionaries, sets, and most custom objects. When you call a method that modifies a mutable object, the modification happens to the object itself, and every name pointing to that object sees the change.

Immutable objects are objects whose internal state cannot change: integers, strings, tuples, and frozensets. When Python appears to "modify" an immutable object, it is actually creating a new object. The original is left untouched.

This distinction is critical when you have aliases.

### EXAMPLE 3.1 — Mutable Aliasing with a Score List

Here we define a list of exam scores and assign a backup reference to it.

```python
scoreList = [85, 90, 75]
backupRef = scoreList

scoreList.append(95)

print(scoreList)  # [85, 90, 75, 95]
print(backupRef)  # [85, 90, 75, 95]
```

Inside this example, both scoreList and backupRef point to the same list. When we append 95 to scoreList, the list object itself is modified. backupRef is not a backup at all — it is just another name for the same list. Both names show the new value.

The name "backupRef" is misleading and dangerous. This is exactly the kind of bug that hides in production code: a developer assumes a second variable is an independent copy, but it is only an alias.

---

## CONCEPT 4.1 — The copy Module: Three Levels of Copying

Python's copy module gives us precise control over how to duplicate objects. There are three levels, and choosing the wrong one is a common source of bugs.

Level one is assignment. Writing newName equals oldName does not copy anything. It creates a new name pointing to the same object.

Level two is shallow copy. The copy dot copy function creates a new outer container object, but the elements inside the container are not copied. They are the same objects, now referenced from two places.

Level three is deep copy. The copy dot deepcopy function recursively copies every object in the entire structure. The result is a completely independent copy with no shared references.

### EXAMPLE 4.1 — Shallow vs Deep Copy on a Configuration Dictionary

Here we define an application configuration dictionary with nested settings.

```python
import copy

originalConfig = {
    "settings": {"timeout": 30, "retries": 3},
    "active": True
}

shallowConfig = copy.copy(originalConfig)
deepConfig = copy.deepcopy(originalConfig)

shallowConfig["settings"]["timeout"] = 60
deepConfig["settings"]["timeout"] = 999

print(originalConfig["settings"]["timeout"])  # 60  — shallow copy shared the inner dict!
print(deepConfig["settings"]["timeout"])       # 999 — deep copy is fully independent
```

Inside this example, the shallow copy creates a new outer dictionary, but the value associated with the key settings is still the exact same dictionary object in both originalConfig and shallowConfig. So modifying shallowConfig's nested timeout also modifies originalConfig's nested timeout. The deep copy, however, created a completely new inner dictionary. Modifying deepConfig's timeout has no effect on originalConfig.

---

## CONCEPT 4.2 — Practical Rule for Copying

Use shallow copy when the outer container needs to be independent but the inner objects are either immutable or intentionally shared — for example, a list of strings, or a list of read-only configuration entries.

Use deep copy when you need complete independence. The primary use cases are: configuration templates that must produce isolated instances, test fixtures that must not contaminate each other, and snapshots of state that must remain stable even as the original changes.

The rule of thumb: if the data might be mutated and you cannot afford shared side effects, use deep copy.

---

## CONCEPT 5.1 — Truth-Value Testing and __bool__

Python evaluates the truthiness of any object in a boolean context — inside an if statement, a while loop, or a boolean operator. The rules are layered.

Python first checks if the class defines dunder method bool. If it does, Python calls it and uses the result. If dunder method bool is not defined but dunder method len is, Python calls dunder method len and treats zero as falsy and any positive number as truthy. If neither method is defined, Python defaults to True.

The built-in falsy values are: None, the integer zero, the float zero, the empty string, the empty list, the empty dictionary, and the empty set. Every non-empty container is truthy. Every non-zero number is truthy.

### EXAMPLE 5.1 — Custom Class with __len__ for Truth Testing

Here we define a TaskQueue class that uses dunder method len to determine truthiness.

```python
class TaskQueue:
    def __init__(self):
        self.taskList = []

    def addTask(self, taskName):
        self.taskList.append(taskName)

    def __len__(self):
        return len(self.taskList)

processingQueue = TaskQueue()

if not processingQueue:
    print("Queue is empty, nothing to process.")

processingQueue.addTask("sendEmail")

if processingQueue:
    print("Queue has tasks. Begin processing.")
```

Inside this example, when we evaluate not processingQueue, Python calls dunder method len on the TaskQueue instance. The taskList is empty, so dunder method len returns zero. Zero is falsy, so not processingQueue evaluates to True, and we print the empty message. After adding a task, the length is 1, which is truthy, so the second branch executes.

This enables the clean Pythonic pattern of writing if not queue instead of the verbose if len of queue equals zero.

---

## CONCEPT 5.2 — The Sentinel Pattern

There is a subtle problem with using None as a default argument. Sometimes you need to distinguish between two cases: no argument was provided, and None was explicitly provided as an argument.

If you use None as your default value, you cannot tell the difference.

The solution is the sentinel pattern. You create a unique object using object parentheses, assign it to a module-level name like MISSING, and use that as the default. Because object parentheses creates a new object every time with a unique identity, your sentinel is guaranteed to be unique. You check for it using is.

### EXAMPLE 5.2 — Sentinel for Optional Configuration

Here we define a module-level sentinel and a configuration update function.

```python
MISSING = object()

def updateCacheTimeout(newTimeout=MISSING):
    if newTimeout is MISSING:
        print("No timeout provided. Using existing value.")
    elif newTimeout is None:
        print("Timeout explicitly cleared. Disabling cache.")
    else:
        print(f"Updating cache timeout to {newTimeout} seconds.")

updateCacheTimeout()         # No timeout provided. Using existing value.
updateCacheTimeout(None)     # Timeout explicitly cleared. Disabling cache.
updateCacheTimeout(120)      # Updating cache timeout to 120 seconds.
```

Inside this example, the default value for newTimeout is not None — it is the MISSING sentinel. This means passing None explicitly is now distinguishable from passing nothing at all. Each case is handled separately and correctly.

---

## CONCEPT 6.1 — Final Takeaway

Python's reference semantics are a deliberate design choice. They make passing large data structures efficient, they enable shared configuration and mutable state management, and they give Python its dynamic, flexible character.

But they place responsibility on the programmer. You must know when two names point to the same object. You must know when to copy. You must use is for identity checks and is None for None checks. You must choose between shallow and deep copy based on whether your inner objects are mutable and whether you can afford shared state.

These concepts are not academic. They appear in every data pipeline when you pass DataFrames between processing steps. They appear in every web application when you share configuration between request handlers. They appear in every test suite when test fixtures are not properly isolated.

Now that you understand how Python stores and shares objects in memory, you are ready for the next lecture, where we explore Python's treatment of functions as first-class objects — values that can be passed, stored, and returned just like any other data.
