# Example 4.1

```python
class TextCleaner:                                         # line 1
    def process(self, textData):                           # line 2
        return textData.strip().lower()                    # line 3
                                                           # line 4
class SentimentScorer:                                     # line 5
    def process(self, textData):                           # line 6
        return {"text": textData, "score": 0.8}            # line 7
                                                           # line 8
def runPipeline(steps, inputData):                         # line 9
    result = inputData                                     # line 10
    for step in steps:                                     # line 11
        result = step.process(result)                      # line 12
    return result                                          # line 13
```

# Example 4.2

```python
class ConsoleOutput:                                       # line 1
    def write(self, messageText):                          # line 2
        print(messageText)                                 # line 3
                                                           # line 4
class FileOutput:                                          # line 5
    def __init__(self, filePath):                          # line 6
        self.filePath = filePath                           # line 7
                                                           # line 8
    def write(self, messageText):                          # line 9
        with open(self.filePath, "a") as f:                # line 10
            f.write(messageText + "\n")                    # line 11
                                                           # line 12
class JsonOutput:                                          # line 13
    def write(self, messageText):                          # line 14
        import json                                        # line 15
        print(json.dumps({"message": messageText}))        # line 16
                                                           # line 17
def logMessage(destination, messageText):                  # line 18
    destination.write(messageText)                         # line 19
```

# Example 4.3

```python
class RegularDiscount:                                     # line 1
    def calculate(self, basePrice):                        # line 2
        return basePrice * 0.95                            # line 3
                                                           # line 4
class HolidayDiscount:                                     # line 5
    def calculate(self, basePrice):                        # line 6
        return basePrice * 0.80                            # line 7
                                                           # line 8
class LoyaltyDiscount:                                     # line 9
    def calculate(self, basePrice):                        # line 10
        return basePrice * 0.70                            # line 11
                                                           # line 12
class PricingEngine:                                       # line 13
    def __init__(self):                                    # line 14
        self.activeStrategy = None                         # line 15
                                                           # line 16
    def setStrategy(self, strategy):                       # line 17
        self.activeStrategy = strategy                     # line 18
                                                           # line 19
    def computePrice(self, basePrice):                     # line 20
        return self.activeStrategy.calculate(basePrice)    # line 21
```
