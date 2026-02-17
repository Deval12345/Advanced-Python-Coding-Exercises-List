Code Module 1

Line 1: class CustomList:

Line 2: def init(self, data):

Line 3: self.\_data = list(data)

Line 4:

Line 5:

Line 6: def len(self):

Line 7: return len(self.\_data)

Line 8:

Line 9:

Line 10: def getitem(self, index):

Line 11: return self.\_data\[index]

Line 12:

Line 13:

Line 14: def contains(self, item):

Line 15: return item in self.\_data

Line 16:

Line 17:

Line 18: def iter(self):

Line 19: return iter(self.\_data)



Code Module 2

Line 1: def analyze(source):

Line 2: lines = list(source)

Line 3: return {

Line 4: "line\_count": len(lines),

Line 5: "word\_count": sum(len(line.split()) for line in lines),

Line 6: "first\_line": lines\[0] if lines else None

Line 7: }



Code Module 3

Line 1: class LazyFileLoader:

Line 2: def init(self, filename):

Line 3: self.filename = filename

Line 4:

Line 5:

Line 6: def getitem(self, index):

Line 7: with open(self.filename) as f:

Line 8: for i, line in enumerate(f):

Line 9: if i == index:

Line 10: return line.strip()

Line 11: raise IndexError



Code Module 4

Line 1: class Discount10:

Line 2: def apply(self, price):

Line 3: return price \* 0.9

Line 4:

Line 5:

Line 6: class Discount20:

Line 7: def apply(self, price):

Line 8: return price \* 0.8

