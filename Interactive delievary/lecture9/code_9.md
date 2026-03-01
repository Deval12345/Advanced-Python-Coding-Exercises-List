# Example 2.1

```python
class Batch:                                          # line 1
    def __init__(self, data):                          # line 2
        self.dataStore = data                          # line 3
    def __iter__(self):                                # line 4
        return iter(self.dataStore)                    # line 5

batch = Batch([10, 20, 30])                           # line 6
print(sum(batch))                                      # line 7
print(list(batch))                                     # line 8
```

# Example 3.1

```python
def countUpTo(limit):                                  # line 1
    for i in range(limit):                             # line 2
        yield i                                        # line 3

gen = countUpTo(4)                                     # line 4
print(list(gen))                                       # line 5
```
