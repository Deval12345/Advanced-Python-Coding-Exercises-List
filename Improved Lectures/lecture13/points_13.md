Slide 0
Title: Welcome back

Point 1: Previous lecture: Python's exception model and EAFP — assume success, handle failure explicitly using try/except rather than defensive pre-checks.

Point 2: Today: descriptors — the protocol that controls attribute access. They are the engine behind property, classmethod, staticmethod, SQLAlchemy, and Django model fields.

Slide 1
Title: The Attribute Validation Problem

Point 1: Real classes have invariants — salary must be positive, port must be 1-65535, probability must be 0-1. Validating in __init__ and every setter creates one copy of the logic per class.

Point 2: With five classes sharing a rule, a rule change requires five updates — any of which can be missed. Descriptors solve this by making validation a reusable, self-contained object.

Slide 2
Title: The Descriptor Protocol

Point 1: A descriptor is any object with __get__, __set__, or __delete__. When assigned to a class attribute, Python routes all access through the descriptor's methods instead of the instance __dict__.

Point 2: Data descriptor (has __set__): takes priority over instance __dict__. Non-data descriptor (only __get__): instance __dict__ takes priority over it.

Slide 3
Title: __set_name__ and Descriptor Binding

Point 1: Python 3.6+ calls __set_name__(ownerClass, name) automatically when a descriptor is assigned to a class attribute, so the descriptor knows its own name without manual repetition.

Point 2: Before __set_name__, names had to be passed in the constructor — appearing twice and easily going out of sync. __set_name__ eliminated this class of bug.

Slide 4
Title: Why Descriptors Over Properties

Point 1: property is a descriptor, but one-per-attribute — validation logic cannot be shared across classes. A descriptor class is instantiated once per attribute in any number of classes.

Point 2: SQLAlchemy columns, Django model fields, dataclasses, and attrs all use descriptor classes for reusable field behavior — one class, many attributes, one place to update rules.

Slide 5
Title: The instance is None Check

Point 1: __get__ is called with instance=None when the attribute is accessed through the class (SensorReading.temperature), not through an object.

Point 2: Returning self when instance is None exposes the descriptor object for class-level callers — this is how Django and SQLAlchemy query builders access field configuration.

Slide 6
Title: -

Point 1: Descriptors transform attribute access into a programmable protocol: __set_name__ binds the name, __get__ handles reads, __set__ enforces validation; data descriptors always win over instance __dict__; reusable validation is the primary use case — one class, any number of classes and attributes.

