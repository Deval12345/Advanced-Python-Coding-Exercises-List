# Slides — Lecture 5
# Memory Layout and Python Object Model: Identity, Mutability, and Copying

---

Slide 0, Title: Transition — From Protocols to Memory
Point 1: Previous lectures built objects that respond to Python syntax through special methods and protocols.
Point 2: New question: how does Python actually store and share objects in memory?
Point 3: Python's reference semantics govern every object assignment, function call, and data mutation.
Point 4: Understanding this model eliminates an entire class of subtle, hard-to-reproduce production bugs.

---

Slide 1, Title: Variables Are References, Not Values
Point 1: In Python, a variable is a label attached to an object — not a box that contains a value.
Point 2: When Python executes x = [1, 2, 3], it creates the list object first, then attaches the name x to it.
Point 3: Writing y = x does NOT create a new list. It creates a second label pointing to the same object.
Point 4: This is aliasing: two names, one object in memory.
Point 5: Key contrast: C and Java store primitive values directly in variables. Python stores references.

---

Slide 2, Title: Aliasing — Two Names, One Object
Point 1: Aliasing example: productList = ["laptop", "phone", "tablet"], then catalogAlias = productList.
Point 2: Both names point to the same list object in memory.
Point 3: Calling append through catalogAlias modifies the shared object — productList reflects the change.
Point 4: Industrial impact: configuration sharing bugs in Django and Flask web applications.
Point 5: Fix: understand aliasing and copy intentionally when independence is required.

---

Slide 3, Title: Identity vs Equality — is vs ==
Point 1: == calls __eq__ and asks: do these two objects have equal content or value?
Point 2: is asks: are these two names attached to the exact same object in memory?
Point 3: Two separately created lists with identical content are value-equal (==) but not identity-equal (is).
Point 4: An alias of a list is identity-equal to the original because both names point to the same object.
Point 5: is does not call any method — it compares raw memory addresses.

---

Slide 4, Title: The None Rule — Always Use is None
Point 1: Never write == None. Always write is None and is not None.
Point 2: __eq__ can be overridden by any class to return True even when compared to None.
Point 3: The is operator cannot be overridden — it is a direct memory-address comparison.
Point 4: None is a singleton: there is exactly one None object in the entire Python runtime.
Point 5: Therefore, is None is always reliable. == None is not. (Fluent Python, ~p. 332)

---

Slide 5, Title: Mutability and Aliasing
Point 1: Mutable objects — lists, dicts, sets, most custom objects — can be changed in-place.
Point 2: Immutable objects — int, str, tuple, frozenset — cannot be changed once created.
Point 3: Modifying a mutable object through any alias changes the shared object for all aliases.
Point 4: "Modifying" an immutable object creates a new object; the original is untouched.
Point 5: Pandas consequence: modifying a DataFrame slice can silently modify the original DataFrame.

---

Slide 6, Title: The copy Module — Three Levels of Copying
Point 1: Level 1 — Assignment (y = x): no copy, both names point to the same object.
Point 2: Level 2 — Shallow copy (copy.copy(x)): new outer container, inner objects are still shared.
Point 3: Level 3 — Deep copy (copy.deepcopy(x)): recursively copies everything, completely independent.
Point 4: Shallow copy of a nested dict: modifying an inner value through the copy also changes the original.
Point 5: Deep copy of a nested dict: modifying the copy has no effect on the original.

---

Slide 7, Title: Practical Rules for Copying
Point 1: Use shallow copy when the outer structure needs independence and inner objects are immutable.
Point 2: Use deep copy when you need full independence: configuration templates, test fixtures, state snapshots.
Point 3: Rule of thumb: if the data might be mutated and side effects are unacceptable, use deep copy.
Point 4: Industrial use case: generating isolated configuration instances from a shared template.
Point 5: Getting copy depth wrong is one of the most common sources of stateful bugs in data pipelines.

---

Slide 8, Title: Truth-Value Testing — __bool__ and __len__
Point 1: Python evaluates truthiness by checking __bool__ first, then __len__, then defaulting to True.
Point 2: Built-in falsy values: None, 0, 0.0, "", [], {}, set().
Point 3: All non-empty containers and non-zero numbers are truthy.
Point 4: Define __len__ on a custom class to make it behave correctly in boolean contexts.
Point 5: Enables clean Pythonic patterns: "if not queue:" instead of "if len(queue) == 0:".

---

Slide 9, Title: The Sentinel Pattern
Point 1: Problem: you cannot distinguish between "no argument passed" and "None explicitly passed" using None as default.
Point 2: Solution: create a sentinel with MISSING = object() at module level.
Point 3: object() creates a unique object with a unique identity — guaranteed to match nothing else.
Point 4: Use it as the default argument and check with is: if value is MISSING.
Point 5: Enables three distinct cases: no value, None explicitly, and an actual value.

---

Slide 10, Title: Final Takeaway — Reference Semantics in Practice
Point 1: Python's reference semantics are a deliberate design choice: efficient, flexible, and powerful.
Point 2: They make function calls with large data structures cheap — no copying by default.
Point 3: They require programmer awareness: aliasing, copy depth, identity checks, truthiness.
Point 4: Use is for identity, is None for None checks, copy.deepcopy for full independence.
Point 5: These concepts govern data pipelines, web application state, and every test suite you write.
Point 6: Next lecture: functions as first-class objects — passing, storing, and returning behavior as data.
