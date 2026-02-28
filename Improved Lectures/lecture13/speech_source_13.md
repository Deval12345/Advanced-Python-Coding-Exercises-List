# Speech Source — Lecture 13: Descriptors Part 1 — The Descriptor Protocol and Reusable Attribute Validation

---

## CONCEPT 0.1 — Transition from Previous Lecture

In the previous lecture, we studied Python's exception model and the EAFP philosophy — assume success, handle failure explicitly. We saw how Python uses special methods like __enter__ and __exit__ to make control flow structural. Now we go one level deeper. Descriptors control what happens when you access an attribute on an object. They are the engine behind Python's property, classmethod, staticmethod, and the major ORM frameworks like SQLAlchemy and Django. Everything you have used with dot notation on a class — all of that behavior is defined by descriptors.

---

## CONCEPT 1.1 — The Attribute Validation Problem

Every real-world class has constraints. An employee's salary must be positive. A server configuration's port must fall between 1 and 65535. A machine learning model's probability output must lie between 0 and 1. A temperature sensor reading must stay within its operational range. These are not optional business rules — they are invariants that, if violated, produce incorrect results, corrupted data, or silent failures that are extremely difficult to debug.

The naive approach is to validate in __init__ and in every setter method. If a class has three validated fields, you write three range checks in __init__, and possibly three more in setter methods. When another class needs the same range check, you copy the code. After a while, you have five classes each with their own version of the same validation logic. When the requirement changes — say the port range now excludes certain reserved ports — you update it in five places and hope you got them all. This is the duplication problem, and it scales badly. Descriptors solve this by making validation a reusable, self-contained object that can be attached to any class attribute.

---

## CONCEPT 1.2 — The Descriptor Protocol

A descriptor is any object that defines __get__, __set__, or __delete__. When you assign a descriptor instance to a class attribute, Python routes all attribute access through the descriptor's methods instead of going to the instance dictionary directly.

__get__ is called when you read the attribute. __set__ is called when you assign to it. __delete__ is called when you delete it. These three methods form the descriptor protocol. An object that defines only __get__ is called a non-data descriptor. An object that defines __get__ and __set__, or __get__ and __delete__, is called a data descriptor.

This distinction matters enormously for lookup priority. A data descriptor takes priority over the instance's __dict__. That means even if the instance has an entry in its own dictionary under the same name, the data descriptor wins and its __get__ is called. A non-data descriptor, by contrast, loses to the instance __dict__. This lookup priority is not arbitrary — it is the mechanism that makes caching, lazy loading, and framework patterns possible. Python's built-in property is a data descriptor. Properties can intercept both reads and writes. A lazy cached property that computes the value once and stores it in the instance __dict__ must be a non-data descriptor, so the cached value shadows it on the next access. We will cover that pattern in the next lecture.

---

## EXAMPLE 1.1 — RangeValidated Descriptor

Here we define a class called RangeValidated that implements the descriptor protocol for a numeric range check.

The constructor takes a minimum value and a maximum value and stores them. It also initializes an attributeName to None, which will be set later.

The __set_name__ method receives the owner class and the name of the attribute this descriptor is assigned to. We store that name so the descriptor knows what to call itself in error messages.

The __get__ method receives the instance and the owner class. If instance is None, the attribute is being accessed through the class rather than through an object — so we return self, the descriptor itself. This is the standard pattern. If instance is not None, we look up the value from the instance's __dict__ using the attribute name we stored.

The __set__ method validates the value against the stored minimum and maximum. If the value falls outside the range, it raises a ValueError with a descriptive message that includes the attribute name, the allowed bounds, and the actual bad value. If the value is valid, it is stored in the instance's __dict__ under the attribute name.

We define a SensorReading class with two class-level attributes: temperature using RangeValidated(-50, 150) and humidity using RangeValidated(0, 100). In __init__, assigning to self.temperature triggers the descriptor's __set__. Creating a reading with valid values succeeds. Accessing reading.temperature triggers __get__ and returns the stored value.

```python
# Example 13.1
class RangeValidated:                                  # line 1
    def __init__(self, minValue, maxValue):             # line 2
        self.minValue = minValue                        # line 3
        self.maxValue = maxValue                        # line 4
        self.attributeName = None                       # line 5

    def __set_name__(self, ownerClass, name):           # line 6
        self.attributeName = name                       # line 7

    def __get__(self, instance, ownerClass):            # line 8
        if instance is None:                            # line 9
            return self                                 # line 10
        return instance.__dict__.get(self.attributeName)  # line 11

    def __set__(self, instance, value):                 # line 12
        if not (self.minValue <= value <= self.maxValue):  # line 13
            raise ValueError(                          # line 14
                f"{self.attributeName} must be between "  # line 15
                f"{self.minValue} and {self.maxValue}, got {value}"  # line 16
            )
        instance.__dict__[self.attributeName] = value  # line 17

class SensorReading:                                   # line 18
    temperature = RangeValidated(-50, 150)             # line 19
    humidity = RangeValidated(0, 100)                  # line 20

    def __init__(self, temperature, humidity):         # line 21
        self.temperature = temperature                 # line 22
        self.humidity = humidity                       # line 23

reading = SensorReading(22.5, 65)                      # line 24
print(reading.temperature)                             # line 25
```

---

## CONCEPT 2.1 — __set_name__ and Descriptor Binding

In Python 3.6 and later, when a descriptor is assigned to a class attribute, Python automatically calls __set_name__ on the descriptor, passing the owner class and the name of the attribute as a string. This lets the descriptor discover and store its own name at class-creation time without requiring the programmer to repeat the name manually.

Before __set_name__ existed, the common pattern was to pass the attribute name explicitly to the descriptor's constructor, like this: salary = PositiveNumber('salary'). This was error-prone because the name appeared twice — once in the assignment and once in the constructor argument. If you renamed the attribute, you had to remember to update the string too. Forgetting this was a frequent source of subtle bugs where the wrong attribute name appeared in error messages, or worse, where the wrong dictionary key was used and two attributes silently shared storage. __set_name__ eliminated this class of bug entirely.

---

## CONCEPT 2.2 — Why Descriptors Over Properties

Python's built-in property is itself a descriptor. It intercepts reads and writes using functions you provide. But property is one-per-attribute. You write @property, @name.setter, and @name.deleter for each attribute individually. You cannot share the same property logic across multiple attributes in the same class, let alone across multiple classes.

A descriptor class, by contrast, can be instantiated once per attribute, across any number of classes, and the validation logic lives in exactly one place. This is the pattern used by SQLAlchemy columns, Django model fields, Python's own dataclasses, and the attrs library. When the validation rule changes, you change it in one descriptor class and every attribute using that descriptor immediately reflects the new behavior. This is the fundamental advantage: descriptors are reusable, property definitions are not.

---

## EXAMPLE 2.1 — PositiveNumber Descriptor Reused Across Multiple Classes

Here we define a descriptor called PositiveNumber that validates that a value is numeric and strictly greater than zero.

__set_name__ stores the attribute name. __get__ returns the value from the instance's __dict__, defaulting to zero if not yet set. __set__ checks that the value is an int or float, then checks that it is greater than zero, raising TypeError or ValueError with a precise message that names the attribute and the bad value.

We then attach PositiveNumber to two completely unrelated classes. Product has price and quantity. Invoice has totalAmount and taxRate. Both classes get the same positive-number enforcement without duplicating a single line of validation logic. Setting product.price to 29.99 goes through the descriptor's __set__. Trying to set it to a negative number would raise ValueError immediately.

```python
# Example 13.2
class PositiveNumber:                                  # line 1
    def __set_name__(self, ownerClass, name):          # line 2
        self.attributeName = name                      # line 3

    def __get__(self, instance, ownerClass):           # line 4
        if instance is None:                           # line 5
            return self                                # line 6
        return instance.__dict__.get(self.attributeName, 0)  # line 7

    def __set__(self, instance, value):                # line 8
        if not isinstance(value, (int, float)):        # line 9
            raise TypeError(f"{self.attributeName} must be numeric, got {type(value).__name__}")  # line 10
        if value <= 0:                                 # line 11
            raise ValueError(f"{self.attributeName} must be positive, got {value}")  # line 12
        instance.__dict__[self.attributeName] = value  # line 13

class Product:                                         # line 14
    price = PositiveNumber()                           # line 15
    quantity = PositiveNumber()                        # line 16

class Invoice:                                         # line 17
    totalAmount = PositiveNumber()                     # line 18
    taxRate = PositiveNumber()                         # line 19

product = Product()                                    # line 20
product.price = 29.99                                  # line 21
product.quantity = 10                                  # line 22
print(f"Product: {product.price} x {product.quantity}")  # line 23
```

---

## CONCEPT 3.1 — The instance is None Check in __get__

Every well-written descriptor's __get__ method begins with the same pattern: if instance is None, return self. This check is necessary because Python calls __get__ in two different situations. When you access the attribute through an instance — reading.temperature — instance is the object and ownerClass is the class. When you access the attribute through the class itself — SensorReading.temperature — instance is None and ownerClass is the class.

If you do not check for None and return early, accessing the descriptor through the class will fail in confusing ways, because the rest of __get__ tries to look something up in instance.__dict__, but instance is None. Returning self when instance is None means that SensorReading.temperature gives you back the RangeValidated descriptor object itself. This is exactly the behavior that frameworks depend on. In Django, MyModel.field_name returns the field descriptor, which is how query builders like MyModel.objects.filter(field_name=value) work — they access the descriptor object and call methods on it to construct SQL.

---

## CONCEPT 4.1 — Descriptors in the Standard Library

Python's own built-in features are implemented as descriptors. This is not an abstraction on top of the language; it is how the language works internally.

property is a descriptor. When you write @property above a method, Python creates a property descriptor object and assigns it to the class attribute. When you access the attribute on an instance, __get__ on the property descriptor calls your getter function. When you assign to it, __set__ calls your setter function. The @property decorator is syntactic sugar for the descriptor protocol.

classmethod is a descriptor. Its __get__ returns a bound method where the first argument is the class rather than the instance. staticmethod is a descriptor. Its __get__ returns the raw function, unbound, so it receives neither the instance nor the class. Understanding descriptors is not just useful for writing frameworks — it means understanding how Python itself works at the protocol level.

---

## CONCEPT 5.1 — Final Takeaway Lecture 13

Descriptors transform attribute access into a programmable protocol. Any object that defines __get__, __set__, or __delete__ is a descriptor. When a descriptor is assigned to a class attribute, Python routes attribute access through the descriptor's methods.

Data descriptors define both __get__ and __set__ (or __delete__). They take priority over the instance's __dict__. Non-data descriptors define only __get__. The instance __dict__ takes priority over them.

__set_name__ is called automatically in Python 3.6 and later, passing the owner class and the attribute name to the descriptor at class-creation time.

The primary use case in this lecture is reusable attribute validation. A descriptor class encapsulates a validation rule. The same descriptor class can be instantiated once per attribute in any number of classes. When the rule changes, you change one class. This is the pattern behind SQLAlchemy columns, Django model fields, Python dataclasses, and the attrs library.

In the next lecture, we will look at non-data descriptors and advanced patterns: lazy loading, computed properties, and how frameworks use descriptors to build powerful class-level APIs.

---
