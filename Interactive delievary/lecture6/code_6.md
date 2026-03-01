# Example 1.1

```python
def logger(func):                                     # line 1
    def wrapper(*args, **kwargs):                     # line 2
        print("Calling:", func.__name__)               # line 3
        return func(*args, **kwargs)                   # line 4
    return wrapper                                    # line 5

@logger                                               # line 6
def add(a, b):                                        # line 7
    return a + b                                      # line 8

print(add(2, 3))                                      # line 9
```

# Example 3.1

```python
def limitCalls(maxCalls):                             # line 1
    def decorator(func):                              # line 2
        count = 0                                    # line 3
        def wrapper(*args, **kwargs):                 # line 4
            nonlocal count                            # line 5
            count += 1                                # line 6
            if count > maxCalls:                      # line 7
                raise RuntimeError("Limit exceeded")  # line 8
            return func(*args, **kwargs)              # line 9
        return wrapper                               # line 10
    return decorator                                 # line 11

@limitCalls(2)                                        # line 12
def apiHandler():                                     # line 13
    print("API called")                               # line 14

apiHandler()                                          # line 15
apiHandler()                                          # line 16
```
