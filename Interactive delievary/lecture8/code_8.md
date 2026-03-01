# Example 2.1

```python
responses = [                                         # line 1
    {"user": 1, "amount": 100},                        # line 2
    {"bad": "data"},                                  # line 3
    {"user": 2, "amount": 200},                        # line 4
]                                                     # line 5

for r in responses:                                   # line 6
    try:                                              # line 7
        print("Processing:", r["user"], r["amount"])  # line 8
    except KeyError:                                  # line 9
        print("Skipping malformed response:", r)      # line 10
```

# Example 3.1

```python
def parseRequest(data):                               # line 1
    try:                                              # line 2
        uid = int(data["user_id"])                    # line 3
        discount = data.get("discount_code")          # line 4
        return (uid, discount)                        # line 5
    except (KeyError, TypeError, ValueError):          # line 6
        return None                                   # line 7
```
