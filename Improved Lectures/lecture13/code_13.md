# Code — Lecture 13: Descriptors Part 1 — The Descriptor Protocol and Reusable Attribute Validation

---

## Example 13.1 — RangeValidated: Data Descriptor with Range Enforcement

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
print(reading.humidity)                                # line 26

try:
    bad = SensorReading(200, 65)                       # line 27
except ValueError as e:
    print(f"Error: {e}")                               # line 28
```

**Output:**
```
22.5
65
Error: temperature must be between -50 and 150, got 200
```

**Explanation:**
- Line 1: RangeValidated is an ordinary class; it becomes a descriptor by implementing __get__ and __set__
- Line 2: the constructor receives the valid minimum and maximum for this particular attribute
- Line 5: attributeName is initialized to None; it will be populated by __set_name__ at class creation time
- Line 6: __set_name__ is called automatically by Python when RangeValidated() is assigned to a class attribute
- Line 7: the descriptor stores its own name so it can use it in error messages and as the __dict__ key
- Line 8: __get__ is called every time the attribute is read on an instance or the class
- Line 9: instance is None when the attribute is accessed through the class (SensorReading.temperature)
- Line 10: returning self exposes the descriptor object itself for class-level access — required by framework patterns
- Line 11: when accessed through an instance, the value is retrieved from the instance's own __dict__
- Line 12: __set__ is called every time the attribute is assigned — this is what makes it a data descriptor
- Line 13: Python's chained comparison checks both bounds in one expression
- Line 14-16: the ValueError message names the attribute, the bounds, and the rejected value for clear debugging
- Line 17: valid values are stored in the instance's __dict__ directly, bypassing the descriptor on storage
- Line 19: assigning RangeValidated(-50, 150) to a class attribute triggers __set_name__ with name='temperature'
- Line 22: self.temperature = temperature triggers __set__ on the descriptor, not a plain __dict__ write
- Line 24: both __set__ calls run during __init__; the values are validated immediately on construction

---

## Example 13.2 — PositiveNumber: Descriptor Reused Across Unrelated Classes

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

invoice = Invoice()                                    # line 24
invoice.totalAmount = 299.90                           # line 25
invoice.taxRate = 0.08                                 # line 26
print(f"Invoice: ${invoice.totalAmount} at {invoice.taxRate * 100}% tax")  # line 27

try:
    product.price = -5                                 # line 28
except ValueError as e:
    print(f"Error: {e}")                               # line 29

try:
    product.quantity = "ten"                           # line 30
except TypeError as e:
    print(f"Error: {e}")                               # line 31
```

**Output:**
```
Product: 29.99 x 10
Invoice: $299.9 at 8.0% tax
Error: price must be positive, got -5
Error: quantity must be numeric, got str
```

**Explanation:**
- Line 1: PositiveNumber has no constructor arguments — it requires no configuration beyond its attribute name
- Line 2: __set_name__ is still called automatically because it is defined; name is assigned at class creation
- Line 7: the default of 0 means uninitialized attributes return 0 rather than None, avoiding type errors downstream
- Line 9: isinstance(value, (int, float)) accepts both integer and floating-point numbers, rejecting strings and booleans
- Line 10: type(value).__name__ produces a readable type name like 'str' rather than "<class 'str'>"
- Line 11: value <= 0 rejects both zero and negative numbers; positive means strictly greater than zero
- Line 14-16: Product gets two independently validated attributes with zero duplicated code
- Line 17-19: Invoice gets two independently validated attributes sharing the same descriptor class
- Line 20: Product() creates an instance with no __init__ arguments; attributes start unset (defaulting to 0)
- Line 21: product.price = 29.99 triggers PositiveNumber.__set__ on the price descriptor
- Line 22: product.quantity = 10 triggers PositiveNumber.__set__ on the quantity descriptor — a different descriptor instance
- Line 28: assigning -5 triggers __set__ which reaches the value <= 0 check and raises ValueError
- Line 30: assigning a string triggers __set__ which fails the isinstance check and raises TypeError

---

## Example 13.3 — Class-Level Access and the instance is None Pattern

```python
# Example 13.3
class TypeChecked:                                     # line 1
    def __init__(self, expectedType):                  # line 2
        self.expectedType = expectedType               # line 3
        self.attributeName = None                      # line 4

    def __set_name__(self, ownerClass, name):          # line 5
        self.attributeName = name                      # line 6

    def __get__(self, instance, ownerClass):           # line 7
        if instance is None:                           # line 8
            return self                                # line 9
        return instance.__dict__.get(self.attributeName)  # line 10

    def __set__(self, instance, value):                # line 11
        if not isinstance(value, self.expectedType):   # line 12
            raise TypeError(                           # line 13
                f"{self.attributeName} must be {self.expectedType.__name__}, "  # line 14
                f"got {type(value).__name__}"          # line 15
            )
        instance.__dict__[self.attributeName] = value  # line 16

class Config:                                          # line 17
    host = TypeChecked(str)                            # line 18
    port = TypeChecked(int)                            # line 19
    debug = TypeChecked(bool)                          # line 20

    def __init__(self, host, port, debug):             # line 21
        self.host = host                               # line 22
        self.port = port                               # line 23
        self.debug = debug                             # line 24

cfg = Config("localhost", 8080, False)                 # line 25
print(f"Config: {cfg.host}:{cfg.port} debug={cfg.debug}")  # line 26

descriptor = Config.host                               # line 27
print(f"Class-level access returns: {descriptor}")     # line 28
print(f"Descriptor type: {type(descriptor).__name__}") # line 29
print(f"Expected type stored: {descriptor.expectedType.__name__}")  # line 30

try:
    cfg.port = "8080"                                  # line 31
except TypeError as e:
    print(f"Error: {e}")                               # line 32
```

**Output:**
```
Config: localhost:8080 debug=False
Class-level access returns: <__main__.TypeChecked object at 0x...>
Descriptor type: TypeChecked
Expected type stored: str
Error: port must be int, got str
```

**Explanation:**
- Line 1: TypeChecked is a generic type-enforcing descriptor parameterized by the expected type
- Line 2: expectedType is stored so __set__ can use isinstance dynamically for any type
- Line 7: __get__ handles both access paths: through an instance and through the class
- Line 8: the None check is the guard for class-level access — without it, line 10 would raise AttributeError
- Line 9: returning self exposes the full descriptor object, including its configuration, to class-level callers
- Line 10: instance.__dict__.get uses the stored attributeName as the key — correct because __set_name__ ran
- Line 17: Config is a class with three type-validated attributes, each enforcing a different type
- Line 27: Config.host accesses the descriptor through the class — __get__ is called with instance=None
- Line 28: the returned value is the TypeChecked descriptor object itself, not a string
- Line 29: type(descriptor).__name__ confirms the returned object is a TypeChecked instance
- Line 30: because we have the descriptor object, we can inspect its configuration — expectedType is 'str'
- Line 31: assigning a string to port triggers TypeChecked.__set__ which sees int != str and raises TypeError

---

## Example 13.4 — Descriptor Storing Per-Instance Data Correctly

```python
# Example 13.4
class BoundedFloat:                                    # line 1
    def __init__(self, low, high, default):            # line 2
        self.low = low                                 # line 3
        self.high = high                               # line 4
        self.default = default                         # line 5

    def __set_name__(self, ownerClass, name):          # line 6
        self.storageName = f"_{name}"                  # line 7

    def __get__(self, instance, ownerClass):           # line 8
        if instance is None:                           # line 9
            return self                                # line 10
        return instance.__dict__.get(self.storageName, self.default)  # line 11

    def __set__(self, instance, value):                # line 12
        value = float(value)                           # line 13
        if not (self.low <= value <= self.high):       # line 14
            raise ValueError(                          # line 15
                f"Value {value} out of range [{self.low}, {self.high}]"  # line 16
            )
        instance.__dict__[self.storageName] = value    # line 17

class NeuralLayer:                                     # line 18
    dropoutRate = BoundedFloat(0.0, 1.0, default=0.5) # line 19
    learningRate = BoundedFloat(1e-6, 1.0, default=0.001)  # line 20

layer1 = NeuralLayer()                                 # line 21
layer2 = NeuralLayer()                                 # line 22

layer1.dropoutRate = 0.3                               # line 23
layer2.dropoutRate = 0.7                               # line 24
layer1.learningRate = 0.001                            # line 25
layer2.learningRate = 0.01                             # line 26

print(f"Layer1: dropout={layer1.dropoutRate}, lr={layer1.learningRate}")  # line 27
print(f"Layer2: dropout={layer2.dropoutRate}, lr={layer2.learningRate}")  # line 28
print(f"Layer1 dict keys: {list(layer1.__dict__.keys())}")  # line 29
```

**Output:**
```
Layer1: dropout=0.3, lr=0.001
Layer2: dropout=0.7, lr=0.01
Layer1 dict keys: ['_dropoutRate', '_learningRate']
```

**Explanation:**
- Line 1: BoundedFloat combines range validation with type coercion (float()) and a default value
- Line 6: __set_name__ constructs a private storage key by prepending an underscore to the attribute name
- Line 7: using '_dropoutRate' as the key avoids any naming collision with the class attribute 'dropoutRate'
- Line 11: the default parameter means uninitialized attributes return a sensible value rather than None
- Line 13: float(value) coerces integers to floats, making the descriptor accept both types transparently
- Line 17: values are stored under the private key, not the public attribute name
- Line 18: NeuralLayer uses two BoundedFloat descriptors covering common machine learning hyperparameters
- Line 21-22: two independent instances each have their own __dict__ storage
- Line 23: layer1.dropoutRate = 0.3 stores 0.3 in layer1.__dict__['_dropoutRate']
- Line 24: layer2.dropoutRate = 0.7 stores 0.7 in layer2.__dict__['_dropoutRate'] — completely independent
- Line 27-28: each instance retrieves its own values; the descriptor does not conflate them
- Line 29: the __dict__ keys use the underscore-prefixed names, showing where the data actually lives

---
