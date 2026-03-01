# Example 2.1

```python
class Config:                                         # line 1
    def __init__(self, source):                       # line 2
        self.sourceStore = source                     # line 3
        self.valueCache = {}                          # line 4
    def __getattr__(self, name):                      # line 5
        if name in self.valueCache:                   # line 6
            return self.valueCache[name]              # line 7
        value = self.sourceStore.get(name)            # line 8
        if value is None:                             # line 9
            raise AttributeError(name)                # line 10
        self.valueCache[name] = value                # line 11
        return value                                  # line 12
```

# Example 3.1

```python
class RemoteAPI:                                      # line 1
    def __getattr__(self, name):                      # line 2
        return f"<remote:{name}>"                     # line 3

api = RemoteAPI()                                     # line 4
print(api.user_profile)                               # line 5
```

# Example 4.1

```python
class Tracer:                                         # line 1
    def __getattribute__(self, name):                 # line 2
        print("Access:", name)                        # line 3
        return super().__getattribute__(name)         # line 4

t = Tracer()                                          # line 5
t.x = 10                                              # line 6
print(t.x)                                            # line 7
```
