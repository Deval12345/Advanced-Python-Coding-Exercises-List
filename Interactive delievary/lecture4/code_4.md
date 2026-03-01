# Example 3.1

```python
class ValidatedField:                                 # line 1
    def __init__(self, minVal, maxVal):               # line 2
        self.minVal = minVal                          # line 3
        self.maxVal = maxVal                          # line 4
        self.storageName = None                       # line 5
    def __set_name__(self, owner, name):              # line 6
        self.storageName = "_" + name                 # line 7
    def __get__(self, obj, owner):                    # line 8
        if obj is None:                               # line 9
            return self                               # line 10
        return getattr(obj, self.storageName)         # line 11
    def __set__(self, obj, value):                    # line 12
        if not (self.minVal <= value <= self.maxVal): # line 13
            raise ValueError("Out of range")           # line 14
        setattr(obj, self.storageName, value)         # line 15

class Product:                                        # line 16
    price = ValidatedField(0, 10000)                  # line 17

p = Product()                                         # line 18
p.price = 500                                         # line 19
print(p.price)                                        # line 20
```

# Example 4.1

```python
class DBField:                                        # line 1
    def __get__(self, obj, owner):                    # line 2
        if obj is None:                               # line 3
            return self                               # line 4
        if "value" not in obj.dataCache:              # line 5
            obj.dataCache["value"] = f"Data for {obj.id}"  # line 6
        return obj.dataCache["value"]                 # line 7

class User:                                           # line 8
    data = DBField()                                  # line 9
    def __init__(self, id):                           # line 10
        self.id = id                                  # line 11
        self.dataCache = {}                           # line 12

u = User(10)                                          # line 13
print(u.data)                                         # line 14
print(u.data)                                         # line 15
```
