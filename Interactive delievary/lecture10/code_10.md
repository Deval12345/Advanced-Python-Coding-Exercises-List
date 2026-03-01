# Example 3.1

```python
class Point:                                          # line 1
    __slots__ = ("x", "y")                            # line 2
    def __init__(self, x, y):                         # line 3
        self.x = x                                    # line 4
        self.y = y                                    # line 5

p = Point(1, 2)                                       # line 6
print(p.x, p.y)                                       # line 7
```
