# Python Data Model (Magic / Dunder Methods) — Instructor Script

## Lesson Opening

Alright everyone, let’s get started.

Today we’re going to talk about something that quietly powers everything you do in Python, but very few people ever learn it explicitly — the Python Data Model, also called magic methods or dunder methods.

Now, I want you to forget for a moment the idea that Python is just a scripting language with some nice syntax. Python is actually a framework, and the API of that framework is not functions — it’s objects.

By the end of this lesson, you should be able to look at normal Python syntax and mentally translate it into method calls. Once that clicks, Python suddenly feels predictable instead of magical.

## Learning Goals (Spoken)

Before we dive in, here’s what I want you to walk away with.

By the end of this session:

- You’ll understand Python as an object-centric framework.
- You’ll know how normal syntax like len(x) or x[i] maps to special methods.
- You’ll see duck typing in action. 
- You’ll learn how to design your own objects so they behave like built-ins.
- And finally, you’ll understand operator overloading in a responsible way.

Alright — let’s begin.

## 1. Python as a Framework

Here’s a mindset shift.

Python does not add new syntax for new features.

Instead, Python says:

> “If your object implements certain methods, I’ll let it participate in my syntax.”

So things like:

```
len(x)
x[i]
for x in obj
x in obj
```

These are not hardcoded for lists or strings.

They work because the object implements special methods.

Let’s slow down and look at the mapping.

### Syntax → Method Mapping (Spoken Explanation)

When you write:

```
len(x)
```

Python internally does:

“Does x have a __len__ method? If yes, call it.”

```
x[i]
```

Python translates that to:

```
x.__getitem__(i)
```

```
for x in obj
```

Python asks:

“Can I get an iterator from this object using __iter__?”

```
x in obj
```

Python checks:

“Does this object know how to check containment using __contains__?”

So the syntax is just sugar.  
The real work happens in special methods.

Once you accept this, Python becomes incredibly consistent.

## 2. Exercise — Build a Native-Like Container

Now let’s make this concrete.

I’m going to build a custom object that behaves like a list — not by inheriting from list, but by speaking Python’s language.

Before I show the code, here’s what I want you to think about:

What operations do lists support?

Which special methods enable those operations?

Alright, let’s write the class.

### Instructor Explanation Before Code

We’re creating a class called CustomList.  
Internally, it will store data in a real list, but externally, Python will treat it like a native container.

Pay attention to the method names — every one of them unlocks syntax.

```python
class CustomList:
    def __init__(self, data):
        self._data = list(data)


    def __len__(self):
        return len(self._data)


    def __getitem__(self, index):
        return self._data[index]


    def __contains__(self, item):
        return item in self._data


    def __iter__(self):
        return iter(self._data)
```

### Instructor Walkthrough — Line by Line

Let’s walk through this carefully.

class CustomList:  
We’re defining a brand-new type. Nothing special yet.

def __init__(self, data):  
This is standard object initialization.

self._data = list(data)  
We convert whatever comes in into a list and store it internally.  
Notice the underscore — this is meant to be internal state.

Now here’s where the magic starts.

def __len__(self):  
This method gets called when someone writes len(custom_list).

return len(self._data)  
We delegate the length logic to the internal list.

Next:

def __getitem__(self, index):  
This unlocks square-bracket syntax — custom_list[i].

Again, we forward the request to the internal list.

def __contains__(self, item):  
This enables the in keyword.

So now x in custom_list works naturally.

Finally:

def __iter__(self):  
This is huge. This allows for x in custom_list.

We simply return the iterator of the internal list.

The takeaway here is powerful:

Python doesn’t care what your object is — only what methods it implements.

## 3. Special Methods as Syntax Glue

Let’s make this idea explicit.

When you write:

```
x + y
```

Python actually does:

```
x.__add__(y)
```

When you write:

```
x[i]
```

Python does:

```
x.__getitem__(i)
```

And when you write:

```
len(x)
```

Python calls:

```
x.__len__()
```

This means operators are not special.  
They’re just method calls with fancy syntax.

That’s why operator overloading exists — and also why it can be abused.

A common mistake is overloading operators with surprising behavior.  
Always ask:

“Would a reasonable Python user expect this operator to do this?”

## 4. Duck Typing

Now let’s talk about duck typing — the real version.

Duck typing means:

> “If it behaves like a thing, I’ll treat it like that thing.”

Let’s look at a function.

### Instructor Explanation Before Code

This function does not care about the type of source.  
It only cares that source behaves in certain ways.

```python
def analyze(source):
    lines = list(source)
    return {
        "line_count": len(lines),
        "word_count": sum(len(line.split()) for line in lines),
        "first_line": lines[0] if lines else None
    }
```

### Instructor Walkthrough — Line by Line

def analyze(source):  
Notice — no type annotations, no inheritance checks.

lines = list(source)  
This line requires that source is iterable.  
That’s it.

If source implements __iter__, this works.

len(lines)  
Now we rely on __len__.

line.split()  
We assume each element behaves like a string.

So this function will work with:

- lists of strings
- file objects
- generators
- custom iterable objects

This is duck typing done right:

No checks, no restrictions — just behavior.

## 5. Lazy Indexable Objects

Now let’s do something interesting.

What if we want indexing — but we don’t want to load everything into memory?

This is where Python’s data model shines.

### Instructor Explanation Before Code

We’ll create a class that allows indexing into a file, one line at a time, without reading the entire file.

```python
class LazyFileLoader:
    def __init__(self, filename):
        self.filename = filename


    def __getitem__(self, index):
        with open(self.filename) as f:
            for i, line in enumerate(f):
                if i == index:
                    return line.strip()
        raise IndexError
```

### Instructor Walkthrough — Line by Line

class LazyFileLoader:  
This object represents a file view, not file contents.

def __init__(self, filename):  
We store only the filename — no data loaded yet.

def __getitem__(self, index):  
This enables loader[i].

Inside:

- We open the file
- Walk through it line by line
- Stop when we reach the requested index

If we never find it:  
raise IndexError  
This is important — Python expects this exception for invalid indexing.

This object:

- Feels like a list
- Acts lazily
- Uses constant memory

That’s powerful.

## 6. Strategy Plug-in Pattern

Finally, let’s look at a very common real-world pattern.

We want interchangeable behaviors — without conditionals.

### Instructor Explanation Before Code

Each class represents a strategy.  
They all expose the same method: apply.

```python
class Discount10:
    def apply(self, price):
        return price * 0.9


class Discount20:
    def apply(self, price):
        return price * 0.8
```

### Instructor Walkthrough — Line by Line

Both classes define:  
def apply(self, price):

That’s the contract.

Anything that has an apply method can be used as a discount strategy.

No inheritance required.  
No base class needed.  
Just behavior.

This pattern works beautifully with duck typing and the data model.

## Final Takeaways (Spoken)

Let’s wrap this up.

Special methods are not advanced Python.  
They are Python itself.

They are the API between:

- your objects
- and Python’s syntax

If you understand the data model:

- Python stops feeling magical
- Your designs become cleaner
- Your code becomes more expressive

In the next lessons, this foundation will pay off again and again.

Alright — great work today 👏
