# Example 3.1

```python
class ReadableBuffer:                                          # line 1
    def __init__(self, content):                               # line 2
        self.content = content                                 # line 3
        self.position = 0                                      # line 4
    def read(self, size=-1):                                   # line 5
        if size == -1:                                         # line 6
            result = self.content[self.position:]              # line 7
            self.position = len(self.content)                  # line 8
            return result                                      # line 9
        result = self.content[self.position:self.position + size]  # line 10
        self.position += size                                  # line 11
        return result                                          # line 12
```

# Example 3.2

```python
class DataBuffer:                                              # line 1
    def __init__(self, data):                                  # line 2
        self.dataStore = data                                  # line 3
    def __len__(self):                                         # line 4
        return len(self.dataStore)                             # line 5
    def __iter__(self):                                        # line 6
        return iter(self.dataStore)                            # line 7

def processData(dataSource):                                   # line 8
    print(len(dataSource))                                     # line 9
    for item in dataSource:                                    # line 10
        print(item)                                            # line 11
```

# Example 3.3

```python
class BrokenContainer:                                         # line 1
    def __init__(self, data):                                  # line 2
        self.dataStore = data                                  # line 3
    def __len__(self):                                         # line 4
        return len(self.dataStore)                             # line 5
    # No __iter__ defined — iterating raises TypeError         # line 6

brokenList = BrokenContainer([10, 20, 30])                     # line 7
print(len(brokenList))                                         # line 8
for item in brokenList:                                        # line 9
    print(item)                                                # line 10
```
