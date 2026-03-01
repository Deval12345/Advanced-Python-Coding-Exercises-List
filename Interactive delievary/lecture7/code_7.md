# Example 3.1

```python
import time                                            # line 1

class Timer:                                            # line 2
    def __enter__(self):                               # line 3
        self.start = time.time()                       # line 4
        return self                                    # line 5
    def __exit__(self, excType, excVal, tb):           # line 6
        elapsed = time.time() - self.start              # line 7
        print("Elapsed:", elapsed)                      # line 8
        return False                                   # line 9

with Timer():                                           # line 10
    time.sleep(0.5)                                     # line 11
```
