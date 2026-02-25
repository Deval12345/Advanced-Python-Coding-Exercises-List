# Python Special Methods -- Combined Examples

------------------------------------------------------------------------

## Example 2.2

``` python
class A:                      # Line 1
    def __len__(self):        # Line 2
        return 5
len(A())                      # Line 3
```

------------------------------------------------------------------------

## Example 2.3

``` python
class B:                              # Line 1
    def __getitem__(self, i):          # Line 2
        return i * 10
B()[2]                                  # Line 3
```

------------------------------------------------------------------------

## Example 2.4

``` python
class C:                              # Line 1
    def __iter__(self):                # Line 2
        return iter([1, 2, 3])
list(C())                              # Line 3
```

------------------------------------------------------------------------

## Example 2.5

``` python
class D:                              # Line 1
    def __contains__(self, x):         # Line 2
        return x == 5
5 in D()                               # Line 3
```

------------------------------------------------------------------------

## Example 3.2

``` python
class E:                              # Line 1
    def __init__(self):                # Line 2
        self._data = [1, 2, 3]
E()._data                               # Line 3
```

------------------------------------------------------------------------

## Example 3.3

``` python
class F:                              # Line 1
    def __len__(self):                 # Line 2
        return 4
len(F())                               # Line 3
```

------------------------------------------------------------------------

## Example 3.4

``` python
class G:                              # Line 1
    def __getitem__(self, i):          # Line 2
        return "A"
G()[2]                                  # Line 3
```

------------------------------------------------------------------------

## Example 3.5

``` python
class H:                              # Line 1
    def __contains__(self, x):         # Line 2
        return True
"x" in H()                             # Line 3
```

------------------------------------------------------------------------

## Example 3.6

``` python
class I:                              # Line 1
    def __iter__(self):                # Line 2
        return iter(["a", "b"])
list(I())                              # Line 3
```

------------------------------------------------------------------------

## Example 3.7

``` python
def consume(x):                       # Line 1
    return list(x)
consume((i for i in range(3)))        # Line 3
```

------------------------------------------------------------------------

## Example 4.1

``` python
class J:                              # Line 1
    def __add__(self, other):          # Line 2
        return "added"
J() + J()                              # Line 3
```

------------------------------------------------------------------------

## Example 5.2

``` python
def collect(x):                       # Line 1
    return list(x)
collect(["a", "b"])                   # Line 3
```

------------------------------------------------------------------------

## Example 5.3

``` python
def count_words(lines):                       # Line 1
    return sum(len(line.split()) for line in lines)
count_words(["hello world"])                  # Line 3
```

------------------------------------------------------------------------

## Example 6.2

``` python
class K:                              # Line 1
    def __getitem__(self, i):          # Line 2
        return i
K()[3]                                  # Line 3
```

------------------------------------------------------------------------

## Example 6.3

``` python
class L:                              # Line 1
    def __getitem__(self, i):          # Line 2
        raise IndexError
L()[1]                                  # Line 3
```

------------------------------------------------------------------------

## Example 7.2

``` python
class M:                              # Line 1
    def apply(self, price):            # Line 2
        return price * 0.9
M().apply(100)                         # Line 3
```
