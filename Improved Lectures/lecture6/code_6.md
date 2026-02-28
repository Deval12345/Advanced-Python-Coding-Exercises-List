# Code — Lecture 6: Functions as Objects — First-Class Functions and Runtime Behavior

---

# Example 6.1 — Function Assignment: Two Names, One Object

```python
def computeTotal(amount):               # line 1 — define computeTotal as a regular function
    return amount * 1.2                 # line 2 — return amount with 20% added

applyTax = computeTotal                 # line 3 — assign computeTotal to a second variable name

result1 = computeTotal(100)             # line 4 — call through the original name
result2 = applyTax(100)                 # line 5 — call through the new name

print(result1)                          # line 6 — outputs: 120.0
print(result2)                          # line 7 — outputs: 120.0
print(computeTotal is applyTax)         # line 8 — outputs: True — both names point to the same object
```

---

# Example 6.2 — Functions in a Dictionary: Runtime Routing Table

```python
def flatFee(amount):                    # line 1 — define flat pricing rule
    return amount + 50                  # line 2 — add a fixed fee of 50

def percentFee(amount):                 # line 3 — define percentage pricing rule
    return amount * 1.05                # line 4 — add 5 percent to amount

pricingRules = {                        # line 5 — store both functions in a dictionary
    "flat": flatFee,                    # line 6 — "flat" key maps to flatFee function object
    "percent": percentFee,              # line 7 — "percent" key maps to percentFee function object
}

def checkout(amount, pricingType):      # line 8 — define checkout function
    selectedRule = pricingRules[pricingType]   # line 9 — look up the function by key
    return selectedRule(amount)         # line 10 — call the selected function with amount

print(checkout(200, "flat"))            # line 11 — outputs: 250
print(checkout(200, "percent"))         # line 12 — outputs: 210.0
```

---

# Example 6.3 — Behavior Injection: processTransactions

```python
def flatFee(amount):                    # line 1 — flat pricing: add 50 to any amount
    return amount + 50

def percentFee(amount):                 # line 3 — percent pricing: add 5 percent
    return amount * 1.05

def processTransactions(transactionList, pricingFunction):   # line 5 — accept list and any function
    results = []                        # line 6 — initialize empty results list
    for transaction in transactionList: # line 7 — iterate over each transaction
        priced = pricingFunction(transaction["amount"])      # line 8 — inject behavior here
        results.append(priced)          # line 9 — collect result
    return results                      # line 10 — return all processed results

transactions = [                        # line 11 — sample transaction data
    {"id": 1, "amount": 100},
    {"id": 2, "amount": 250},
    {"id": 3, "amount": 400},
]

flatResults = processTransactions(transactions, flatFee)       # line 15 — inject flatFee
percentResults = processTransactions(transactions, percentFee) # line 16 — inject percentFee

print(flatResults)                      # line 17 — outputs: [150, 300, 450]
print(percentResults)                   # line 18 — outputs: [105.0, 262.5, 420.0]
```

---

# Example 6.4 — Pipeline: Text Cleaning with a List of Functions

```python
def stripWhitespace(textData):          # line 1 — remove leading and trailing spaces
    return textData.strip()

def convertToLowercase(textData):       # line 3 — convert all characters to lowercase
    return textData.lower()

def removeSpecialChars(textData):       # line 5 — keep only alphanumeric characters and spaces
    return "".join(
        char for char in textData       # line 7 — iterate over each character
        if char.isalnum() or char == " " # line 8 — keep if alphanumeric or a space
    )

cleaningPipeline = [                    # line 10 — define the pipeline as an ordered list
    stripWhitespace,                    # line 11 — step 1: strip whitespace
    convertToLowercase,                 # line 12 — step 2: lowercase
    removeSpecialChars,                 # line 13 — step 3: remove special characters
]

def applyPipeline(rawText, pipeline):   # line 15 — define the pipeline runner
    processedText = rawText             # line 16 — start with the original input
    for step in pipeline:               # line 17 — apply each step in order
        processedText = step(processedText)  # line 18 — output of each step feeds next
    return processedText                # line 19 — return fully processed text

dirtyInput = "  Hello, WORLD!!!  "     # line 21 — sample dirty input string
cleanOutput = applyPipeline(dirtyInput, cleaningPipeline)   # line 22 — run through pipeline
print(cleanOutput)                      # line 23 — outputs: "hello world"
```

---

# Example 6.5 — Function Factory: makeValidator

```python
def makeValidator(minimumAmount):       # line 1 — factory receives configuration parameter
    def validate(transactionAmount):    # line 2 — inner function is the actual validator
        return transactionAmount >= minimumAmount   # line 3 — check against the configured minimum
    return validate                     # line 4 — return the validator function, not its result

highValueValidator = makeValidator(100) # line 6 — create validator requiring at least 100
standardValidator = makeValidator(10)   # line 7 — create validator requiring at least 10

print(highValueValidator(150))          # line 9 — outputs: True
print(highValueValidator(50))           # line 10 — outputs: False
print(standardValidator(50))            # line 11 — outputs: True
print(standardValidator(5))             # line 12 — outputs: False

validTransactions = [                   # line 14 — filter a list using the factory-produced function
    amount for amount in [5, 50, 150, 200, 8]
    if highValueValidator(amount)       # line 16 — use validator as a callable in a comprehension
]
print(validTransactions)                # line 17 — outputs: [150, 200]
```
