# Lecture 7 Code Examples: Closures, Lambda Functions, and Advanced Function Patterns

---

# Example 7.1

```python
def makeThresholdValidator(thresholdAmount):          # line 1
    def checkValue(transactionValue):                 # line 2
        return transactionValue >= thresholdAmount    # line 3
    return checkValue                                 # line 4

highValueValidator = makeThresholdValidator(10000)    # line 5
standardValidator = makeThresholdValidator(100)       # line 6

print(highValueValidator(15000))   # line 7 — True
print(highValueValidator(5000))    # line 8 — False
print(standardValidator(150))      # line 9 — True
print(standardValidator(50))       # line 10 — False

print(highValueValidator.__closure__[0].cell_contents)  # line 11 — 10000
```

---

# Example 7.2

```python
def makePaymentProcessor(feeRate, currency):          # line 1
    def processPayment(amount):                       # line 2
        fee = amount * feeRate                        # line 3
        return {                                      # line 4
            "currency": currency,                     # line 5
            "originalAmount": amount,                 # line 6
            "feeCharged": fee,                        # line 7
            "totalCharged": amount + fee              # line 8
        }                                             # line 9
    return processPayment                             # line 10

usdProcessor = makePaymentProcessor(0.02, "USD")      # line 11
eurProcessor = makePaymentProcessor(0.03, "EUR")      # line 12

print(usdProcessor(500))   # line 13 — {'currency': 'USD', 'originalAmount': 500, 'feeCharged': 10.0, 'totalCharged': 510.0}
print(eurProcessor(500))   # line 14 — {'currency': 'EUR', 'originalAmount': 500, 'feeCharged': 15.0, 'totalCharged': 515.0}
```

---

# Example 7.3

```python
def makeCounter():                       # line 1
    callCount = 0                        # line 2
    def increment():                     # line 3
        nonlocal callCount               # line 4
        callCount += 1                   # line 5
        return callCount                 # line 6
    return increment                     # line 7

counter = makeCounter()                  # line 8
print(counter())   # line 9  — 1
print(counter())   # line 10 — 2
print(counter())   # line 11 — 3

secondCounter = makeCounter()            # line 12
print(secondCounter())  # line 13 — 1   (independent state)
print(counter())        # line 14 — 4   (original counter unaffected)
```

---

# Example 7.4

```python
transactionList = [                                              # line 1
    {"amount": 450,  "currency": "USD"},                        # line 2
    {"amount": 1200, "currency": "EUR"},                        # line 3
    {"amount": 75,   "currency": "USD"},                        # line 4
    {"amount": 890,  "currency": "GBP"},                        # line 5
]                                                               # line 6

sortedByAmount = sorted(                                        # line 7
    transactionList,                                            # line 8
    key=lambda record: record["amount"]                         # line 9
)                                                               # line 10

largeTransactions = list(filter(                                # line 11
    lambda record: record["amount"] > 400,                      # line 12
    transactionList                                             # line 13
))                                                              # line 14

scaledAmounts = list(map(                                       # line 15
    lambda record: record["amount"] * 1.1,                      # line 16
    transactionList                                             # line 17
))                                                              # line 18

print(sortedByAmount)     # line 19 — sorted ascending by amount
print(largeTransactions)  # line 20 — transactions over 400
print(scaledAmounts)      # line 21 — all amounts scaled by 1.1
```

---

# Example 7.5

```python
# Closure approach
def makeBalanceTracker(initialBalance):         # line 1
    currentBalance = initialBalance             # line 2
    def applyTransaction(amount):               # line 3
        nonlocal currentBalance                 # line 4
        currentBalance += amount                # line 5
        return currentBalance                   # line 6
    return applyTransaction                     # line 7

closureTracker = makeBalanceTracker(1000)       # line 8
print(closureTracker(200))   # line 9  — 1200
print(closureTracker(-50))   # line 10 — 1150

# Class approach — equivalent behavior
class BalanceTracker:                           # line 11
    def __init__(self, initialBalance):         # line 12
        self.currentBalance = initialBalance    # line 13
    def applyTransaction(self, amount):         # line 14
        self.currentBalance += amount           # line 15
        return self.currentBalance              # line 16

classTracker = BalanceTracker(1000)             # line 17
print(classTracker.applyTransaction(200))  # line 18 — 1200
print(classTracker.applyTransaction(-50))  # line 19 — 1150

# The closure is leaner for single-method behavior.
# The class is preferable when multiple methods or
# direct state inspection are needed.         # line 20
```
