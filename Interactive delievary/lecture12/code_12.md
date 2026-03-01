# Example 1.1

```python
import threading                                      # line 1
import time                                           # line 2

def fetch(urlId):                                     # line 3
    time.sleep(0.2)                                   # line 4
    return f"Result {urlId}"                          # line 5

threads = [                                           # line 6
    threading.Thread(target=fetch, args=(i,))         # line 7
    for i in range(5)                                 # line 8
]                                                     # line 9
for t in threads:                                     # line 10
    t.start()                                         # line 11
for t in threads:                                     # line 12
    t.join()                                          # line 13
```

# Example 2.1

```python
import threading                                      # line 1

counter = 0                                           # line 2
lock = threading.Lock()                               # line 3

def increment():                                      # line 4
    global counter                                    # line 5
    for _ in range(1000):                            # line 6
        with lock:                                    # line 7
            counter += 1                              # line 8

threads = [threading.Thread(target=increment) for _ in range(5)]  # line 9
for t in threads:                                     # line 10
    t.start()                                         # line 11
for t in threads:                                     # line 12
    t.join()                                          # line 13
print(counter)                                        # line 14
```
