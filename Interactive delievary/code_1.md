# =========================
# Code Module 1
# =========================

class CustomList:                               # Module 1 Line 1
    def __init__(self, data):                   # Module 1 Line 2
        self._data = list(data)                 # Module 1 Line 3

    def __len__(self):                          # Module 1 Line 6
        return len(self._data)                  # Module 1 Line 7

    def __getitem__(self, index):               # Module 1 Line 10
        return self._data[index]                # Module 1 Line 11

    def __contains__(self, item):               # Module 1 Line 14
        return item in self._data               # Module 1 Line 15

    def __iter__(self):                         # Module 1 Line 18
        return iter(self._data)                 # Module 1 Line 19


# =========================
# Code Module 2
# =========================

def analyze(source):                            # Module 2 Line 1
    lines = list(source)                        # Module 2 Line 2
    return {                                    # Module 2 Line 3
        "line_count": len(lines),               # Module 2 Line 4
        "word_count": sum(len(line.split())     # Module 2 Line 5
                           for line in lines),
        "first_line": lines[0] if lines else None  # Module 2 Line 6
    }                                           # Module 2 Line 7


# =========================
# Code Module 3
# =========================

class LazyFileLoader:                           # Module 3 Line 1
    def __init__(self, filename):               # Module 3 Line 2
        self.filename = filename                # Module 3 Line 3

    def __getitem__(self, index):               # Module 3 Line 6
        with open(self.filename) as f:          # Module 3 Line 7
            for i, line in enumerate(f):        # Module 3 Line 8
                if i == index:                  # Module 3 Line 9
                    return line.strip()         # Module 3 Line 10
        raise IndexError                        # Module 3 Line 11


# =========================
# Code Module 4
# =========================

class Discount10:                               # Module 4 Line 1
    def apply(self, price):                     # Module 4 Line 2
        return price * 0.9                      # Module 4 Line 3


class Discount20:                               # Module 4 Line 6
    def apply(self, price):                     # Module 4 Line 7
        return price * 0.8                      # Module 4 Line 8
