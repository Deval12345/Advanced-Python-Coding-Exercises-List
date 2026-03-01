# Example 2.1

```python
from multiprocessing import Pool                      # line 1
import math                                           # line 2

def compute(n):                                       # line 3
    return sum(math.sqrt(i) for i in range(n))         # line 4

if __name__ == "__main__":                            # line 5
    with Pool(4) as pool:                             # line 6
        results = pool.map(compute, [100000] * 4)     # line 7
    print(len(results), results[0])                    # line 8
```
