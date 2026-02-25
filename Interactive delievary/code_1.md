# Code Module 1 – Basic len Example

numbers = [1, 2, 3]  # Module 1 Line 1
len(numbers)         # Module 1 Line 2


# Code Module 2 – Indexing Example

numbers = [10, 20, 30]  # Module 2 Line 1
value = numbers[1]      # Module 2 Line 2


# Code Module 3 – Iteration Example

for item in numbers:  # Module 3 Line 1
    print(item)       # Module 3 Line 2


# Code Module 4 – Membership Example

if 20 in numbers:     # Module 4 Line 1
    print("found")    # Module 4 Line 2


# Code Module 5 – CustomList Implementation

class CustomList:                     # Module 5 Line 1
    def __init__(self, data):         # Module 5 Line 2
        self._data = list(data)       # Module 5 Line 3

    def __len__(self):                # Module 5 Line 4
        return len(self._data)        # Module 5 Line 5

    def __getitem__(self, index):     # Module 5 Line 6
        return self._data[index]      # Module 5 Line 7


# Code Module 6 – Operator Example

a = 5            # Module 6 Line 1
b = 10           # Module 6 Line 2
result = a + b   # Module 6 Line 3


# Code Module 7 – Duck Typing Example

def total(values):           # Module 7 Line 1
    result = 0               # Module 7 Line 2
    for value in values:     # Module 7 Line 3
        result += value      # Module 7 Line 4
    return result            # Module 7 Line 5


# Code Module 8 – Lazy Loader Example

class LazyFileLoader:                 # Module 8 Line 1
    def __init__(self, filename):     # Module 8 Line 2
        self.filename = filename      # Module 8 Line 3

    def __getitem__(self, index):     # Module 8 Line 4
        with open(self.filename) as f:
            for i, line in enumerate(f):
                if i == index:
                    return line.strip()
        raise IndexError


loader = LazyFileLoader("log.txt")  # Module 8 Line 10
line = loader[5]                    # Module 8 Line 11


# Code Module 9 – Strategy Example

class Discount10:                   # Module 9 Line 1
    def apply(self, price):         # Module 9 Line 2
        return price * 0.9          # Module 9 Line 3


discount = Discount10()             # Module 9 Line 4
final_price = discount.apply(100)   # Module 9 Line 5
