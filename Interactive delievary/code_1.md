Code Module 1
Line 1:
class CustomList:
Line 2:
def init(self, data):
Line 3:
self._data = list(data)
Line 4:

Line 5:
def len(self):
Line 6:
return len(self._data)
Line 7:

Line 8:
def getitem(self, index):
Line 9:
return self._data[index]
Line 10:

Line 11:
def contains(self, item):
Line 12:
return item in self._data
Line 13:

Line 14:
def iter(self):
Line 15:
return iter(self._data)

Code Module 2
Line 1:
def analyze(source):
Line 2:
lines = list(source)
Line 3:
return {
Line 4:
"line_count": len(lines),
Line 5:
"word_count": sum(len(line.split()) for line in lines),
Line 6:
"first_line": lines[0] if lines else None
Line 7:
}

Code Module 3
Line 1:
class LazyFileLoader:
Line 2:
def init(self, filename):
Line 3:
self.filename = filename
Line 4:

Line 5:
def getitem(self, index):
Line 6:
with open(self.filename) as f:
Line 7:
for i, line in enumerate(f):
Line 8:
if i == index:
Line 9:
return line.strip()
Line 10:
raise IndexError

Code Module 4
Line 1:
class Discount10:
Line 2:
def apply(self, price):
Line 3:
return price * 0.9
Line 4:

Line 5:
class Discount20:
Line 6:
def apply(self, price):
Line 7:
return price * 0.8
