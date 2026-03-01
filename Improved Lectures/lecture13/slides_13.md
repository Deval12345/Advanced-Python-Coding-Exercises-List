# Slides — Lecture 13: Descriptors Part 1 — The Descriptor Protocol and Reusable Attribute Validation

---

## Slide 1
**Title:** From Exception Handling to Attribute Control

- L12 covered Python's EAFP philosophy and the exception model — assume success, handle failure explicitly
- Special methods like __enter__ and __exit__ make resource lifecycle structural
- Descriptors extend this idea: they control what happens when you access an attribute on an object
- They are the engine behind property, classmethod, staticmethod, SQLAlchemy, and Django fields
- Understanding descriptors means understanding how Python works at the protocol level

---

## Slide 2
**Title:** The Attribute Validation Problem

- Real classes have invariants: salary must be positive, port must be 1-65535, probability must be 0-1
- The naive approach validates in __init__ and every setter — one copy of the logic per class
- With five classes sharing the same rule, a rule change requires five updates, any of which can be missed
- This duplication grows without bound as the codebase scales
- Descriptors solve this by making the validation rule a reusable, self-contained object

---

## Slide 3
**Title:** The Descriptor Protocol

- A descriptor is any object that defines __get__, __set__, or __delete__
- When assigned to a class attribute, Python routes access through the descriptor's methods instead of __dict__
- __get__ is called on attribute read; __set__ is called on attribute write; __delete__ is called on deletion
- Data descriptor: defines __get__ and __set__ (or __delete__) — takes priority over instance __dict__
- Non-data descriptor: defines only __get__ — instance __dict__ takes priority over it

---

## Slide 4
**Title:** Example 13.1 — RangeValidated Descriptor

- RangeValidated stores minValue, maxValue, and attributeName (set via __set_name__)
- __get__ returns self when instance is None (class-level access); otherwise reads from instance.__dict__
- __set__ validates the range and raises ValueError with a descriptive message on failure
- SensorReading uses RangeValidated(-50, 150) for temperature and RangeValidated(0, 100) for humidity
- One descriptor class validates two attributes — the logic lives in one place

---

## Slide 5
**Title:** __set_name__ — Automatic Name Binding

- In Python 3.6+, Python calls __set_name__(ownerClass, name) on descriptors at class creation time
- The descriptor stores its attribute name so it can use it in error messages and __dict__ lookups
- Before __set_name__, names had to be passed explicitly: salary = PositiveNumber('salary')
- The name appeared twice — in the assignment and in the constructor argument — and could desync
- __set_name__ eliminates this class of bug: the descriptor always knows exactly which attribute it is

---

## Slide 6
**Title:** Why Descriptors Over Properties

- Python's built-in property is itself a descriptor — it intercepts reads and writes via functions
- But property is one-per-attribute and cannot be reused across classes
- A descriptor class is instantiated once per attribute in any number of classes
- When the validation rule changes, update one class — every attribute using it reflects the change immediately
- SQLAlchemy columns, Django model fields, dataclasses, and attrs all use this exact pattern

---

## Slide 7
**Title:** Example 13.2 — PositiveNumber Reused Across Classes

- PositiveNumber checks that values are numeric (int or float) and strictly greater than zero
- TypeError is raised for wrong types; ValueError is raised for non-positive numbers
- Product uses PositiveNumber() for both price and quantity
- Invoice uses PositiveNumber() for both totalAmount and taxRate
- Four validated attributes across two unrelated classes — zero duplicated validation logic

---

## Slide 8
**Title:** The instance is None Check in __get__

- Python calls __get__ in two situations: through an instance and through the class itself
- Through an instance: instance is the object, ownerClass is the class
- Through the class: instance is None, ownerClass is the class
- Without the None check, class-level access crashes when __get__ tries to use instance.__dict__
- Returning self when instance is None enables the framework pattern: MyModel.field returns the descriptor

---

## Slide 9
**Title:** How Django and SQLAlchemy Depend on This

- Django field descriptors return the field descriptor object when accessed through the class
- query builders call methods on the descriptor to construct SQL: MyModel.objects.filter(name='alice')
- SQLAlchemy Column descriptors work the same way for query expression building
- The instance-is-None return-self pattern is not defensive coding — it is a load-bearing protocol guarantee
- Every serious Python ORM or framework field is built on this single pattern

---

## Slide 10
**Title:** Descriptors in the Python Standard Library

- property is a descriptor: __get__ calls the getter function; __set__ calls the setter function
- classmethod is a descriptor: __get__ returns a bound method where the first argument is the class
- staticmethod is a descriptor: __get__ returns the raw function, unbound, receiving neither self nor cls
- These are not abstractions on top of Python — they are how Python itself is implemented
- Understanding descriptors means understanding why Python's own built-in features work the way they do

---

## Slide 11
**Title:** Data vs Non-Data Descriptors — Lookup Priority

- Data descriptor (has __set__): wins over instance __dict__ — reads always go through __get__
- Non-data descriptor (only __get__): loses to instance __dict__ — a stored value shadows the descriptor
- property is a data descriptor — it can intercept both reads and writes, beating the instance dict
- A lazy cached property must be a non-data descriptor — once it stores the result in __dict__, the dict wins
- This priority chain is the entire explanation for Python's attribute lookup behavior

---

## Slide 12
**Title:** Summary — Lecture 13

- A descriptor is any object with __get__, __set__, or __delete__ assigned to a class attribute
- Data descriptors (with __set__) take priority over instance __dict__; non-data descriptors do not
- __set_name__ is called automatically at class creation, letting the descriptor discover its own name
- The instance-is-None check in __get__ enables both safe class-level access and framework query APIs
- Reusable validation is the primary use case: one descriptor class, many attributes, one place to update
- Next lecture: non-data descriptors, lazy loading, cached_property, and framework class-level APIs

---
