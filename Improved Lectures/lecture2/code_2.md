# Example 1.1

```python
class ProductInventory:
    def __init__(self, productList):          # line 1
        self.productList = productList        # line 2

    def __len__(self):                        # line 3
        return len(self.productList)          # line 4

    def __getitem__(self, index):             # line 5
        return self.productList[index]        # line 6

    def __contains__(self, productName):      # line 7
        return any(                           # line 8
            p["name"] == productName          # line 9
            for p in self.productList         # line 10
        )                                     # line 11

    def __iter__(self):                       # line 12
        return iter(self.productList)         # line 13
```

# Example 2.1

```python
class TransactionBatch:
    def __init__(self, transactionList):               # line 1
        self.transactionList = transactionList         # line 2

    def __len__(self):                                 # line 3
        return len(self.transactionList)               # line 4

    def __getitem__(self, position):                   # line 5
        return self.transactionList[position]          # line 6

    def __contains__(self, transactionId):             # line 7
        return any(                                    # line 8
            t["id"] == transactionId                   # line 9
            for t in self.transactionList              # line 10
        )                                              # line 11

    def __iter__(self):                                # line 12
        return iter(self.transactionList)              # line 13
```

# Example 3.1

```python
class PriceAmount:
    def __init__(self, amount, currency):              # line 1
        self.amount = amount                           # line 2
        self.currency = currency                       # line 3

    def __add__(self, otherPrice):                     # line 4
        if self.currency != otherPrice.currency:       # line 5
            raise ValueError("Currency mismatch")     # line 6
        return PriceAmount(                            # line 7
            self.amount + otherPrice.amount,           # line 8
            self.currency                              # line 9
        )                                              # line 10

    def __repr__(self):                                # line 11
        return f"PriceAmount({self.amount}, {self.currency})"  # line 12

    def __str__(self):                                 # line 13
        return f"{self.currency}{self.amount:.2f}"     # line 14
```

# Example 5.1

```python
class FeatureStore:
    def __init__(self, featureList):                   # line 1
        self.featureList = featureList                 # line 2

    def __len__(self):                                 # line 3
        return len(self.featureList)                   # line 4

    def __getitem__(self, index):                      # line 5
        return self.featureList[index]                 # line 6

    def __contains__(self, featureName):               # line 7
        return any(                                    # line 8
            f["name"] == featureName                   # line 9
            for f in self.featureList                  # line 10
        )                                              # line 11

    def __iter__(self):                                # line 12
        return iter(self.featureList)                  # line 13

    def __add__(self, otherStore):                     # line 14
        return FeatureStore(                           # line 15
            self.featureList + otherStore.featureList  # line 16
        )                                              # line 17

    def __repr__(self):                                # line 18
        return f"FeatureStore({len(self.featureList)} features)"  # line 19
```
