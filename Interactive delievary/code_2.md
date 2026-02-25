# Code Module 1

``` python
# Module: Code Module 1

class Sequence:  # line 1
    def __init__(self, data):  # line 2
        self.data = data  # line 3

    def __len__(self):  # line 5
        return len(self.data)  # line 6

    def __getitem__(self, index):  # line 8
        return self.data[index]  # line 9
```

------------------------------------------------------------------------

# Code Module 2

``` python
# Module: Code Module 2

def total(values):  # line 1
    result = 0  # line 2
    for v in values:  # line 3
        result += v  # line 4
    return result  # line 5


print(total([1, 2, 3]))  # line 7
print(total((4, 5, 6)))  # line 8
```

------------------------------------------------------------------------

# Code Module 3

``` python
# Module: Code Module 3

class JsonSerializer:  # line 1
    def serialize(self, data):  # line 2
        import json  # line 3
        return json.dumps(data)  # line 4


class XmlSerializer:  # line 6
    def serialize(self, data):  # line 7
        return "<data>" + str(data) + "</data>"  # line 8


def save(serializer, data):  # line 10
    return serializer.serialize(data)  # line 11


print(save(JsonSerializer(), {"x": 1}))  # line 13
print(save(XmlSerializer(), {"x": 1}))  # line 14
```

------------------------------------------------------------------------

# Code Module 4

``` python
# Module: Code Module 4

class MemoryBuffer:  # line 1
    def __init__(self):  # line 2
        self.data = ""  # line 3

    def write(self, text):  # line 5
        self.data += text  # line 6


buffer = MemoryBuffer()  # line 8
buffer.write("Hello")  # line 9
```

------------------------------------------------------------------------

# Code Module 5

``` python
# Module: Code Module 5

from typing import Protocol  # line 1


class Writable(Protocol):  # line 3
    def write(self, text: str) -> None:  # line 4
        ...  # line 5


def log(writer: Writable):  # line 7
    writer.write("Log entry")  # line 8


log(MemoryBuffer())  # line 10
```
