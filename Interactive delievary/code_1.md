Example 2.2
Line 1: class A:
Line 2:     def __len__(self): return 5
Line 3: len(A())

Example 2.3
Line 1: class B:
Line 2:     def __getitem__(self, i): return i * 10
Line 3: B()[2]

Example 2.4
Line 1: class C:
Line 2:     def __iter__(self): return iter([1, 2, 3])
Line 3: list(C())

Example 2.5
Line 1: class D:
Line 2:     def __contains__(self, x): return x == 5
Line 3: 5 in D()

Example 3.2
Line 1: class E:
Line 2:     def __init__(self): self._data = [1, 2, 3]
Line 3: E()._data

Example 3.3
Line 1: class F:
Line 2:     def __len__(self): return 4
Line 3: len(F())

Example 3.4
Line 1: class G:
Line 2:     def __getitem__(self, i): return "A"
Line 3: G()[2]

Example 3.5
Line 1: class H:
Line 2:     def __contains__(self, x): return True
Line 3: "x" in H()

Example 3.6
Line 1: class I:
Line 2:     def __iter__(self): return iter(["a", "b"])
Line 3: list(I())

Example 3.7
Line 1: def consume(x):
Line 2:     return list(x)
Line 3: consume((i for i in range(3)))

Example 4.1
Line 1: class J:
Line 2:     def __add__(self, other): return "added"
Line 3: J() + J()

Example 5.2
Line 1: def collect(x):
Line 2:     return list(x)
Line 3: collect(["a", "b"])

Example 5.3
Line 1: def count_words(lines):
Line 2:     return sum(len(line.split()) for line in lines)
Line 3: count_words(["hello world"])

Example 6.2
Line 1: class K:
Line 2:     def __getitem__(self, i): return i
Line 3: K()[3]

Example 6.3
Line 1: class L:
Line 2:     def __getitem__(self, i): raise IndexError
Line 3: L()[1]

Example 7.2
Line 1: class M:
Line 2:     def apply(self, price): return price * 0.9
Line 3: M().apply(100)
