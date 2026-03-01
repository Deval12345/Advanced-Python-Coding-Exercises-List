# Speech Source — Lecture 42: Big Project Stage 8 Part 2 — Advanced Internals: Metaclasses, Class Decorators, and Dynamic Generation

---

## CONCEPT: Why class creation itself can be intercepted

BLOCK_START
In lecture 41, we intercepted attribute access using descriptors. But there is a deeper layer of interception: class creation itself.

When Python processes a class body, it goes through a defined sequence. It reads the class statement, collects the namespace, and then calls a callable to assemble the class object. That callable — by default — is type. The built-in type function is not just for checking types. It is the metaclass: the class that creates classes.

The insight: if you can substitute a custom callable at that moment, you can rewrite the class before it is ever used. You can inject descriptors automatically. You can validate that required methods exist. You can register every class in a global registry the moment it is defined. None of this requires the user of the class to do anything — it happens automatically during class creation.

The three tools for this level of interception are: class decorators (apply a function to a class after it is created), __init_subclass__ (a hook called on the parent whenever a subclass is defined), and metaclasses (replace type itself with a custom class factory).

We use the mildest tool that solves the problem. Class decorators for simple post-processing. __init_subclass__ for parent-controlled subclass registration. Metaclasses for deep structural enforcement that requires access to the class namespace before the class object is finalized.
BLOCK_END

---

## EXAMPLE: Auto-registering pipeline stages with __init_subclass__

BLOCK_START
The pluggable analytics engine needs to discover all available pipeline stages dynamically — so that configuration files can refer to stages by string name without any import statement.

The traditional approach: maintain a dictionary and manually register every new stage. Two problems. First, you forget. Second, the registration logic is scattered across the codebase.

The __init_subclass__ approach: put the registration logic in the base class. Whenever anyone defines a subclass — in any file, anywhere in the project — Python calls __init_subclass__ on the parent. You add the subclass to a registry there.

Here is how it works. Define a base class PipelineStage. In its __init_subclass__ classmethod, accept a keyword argument called stageType. Store it as the registration key in a class-level registry dictionary. Subclasses declare their own type by passing keyword arguments in their class statement: class ThresholdFilter(PipelineStage, stageType="threshold"). Python routes that keyword argument to __init_subclass__ automatically.

The factory function then becomes trivial: look up the stageType string in the registry, call the constructor. No if-elif chain. No imports. The registry builds itself.

This is exactly how plugin systems and framework extension points work in production — Django's model metaclass, SQLAlchemy's declarative base, Pydantic's model registration.
BLOCK_END

---

## CONCEPT: Class decorators for structural transformation

BLOCK_START
A class decorator is a function that receives a class object and returns a class object. It runs after the class body is processed and the class object is assembled — before any instances are created.

The simplest use: add methods to a class from outside. The powerful use: inspect the class body and add descriptors for every annotated attribute automatically.

The dataclass decorator is the most famous example. You annotate fields with type hints. The decorator reads cls.__annotations__, generates __init__, __repr__, and __eq__ automatically, and returns the enhanced class. That is all a decorator is.

For our pipeline: imagine we want every class that processes records to automatically have a descriptor for every annotated attribute. No manual descriptor assignment. No explicit class variable. Just annotate the field, and the descriptor appears.

The pattern: write a function called monitor_fields. It iterates over cls.__annotations__. For each annotated field name, it creates an AuditedAttribute descriptor and calls setattr to inject it into the class. It returns the class. Apply it as a decorator. That is four lines of metaclass-like power with none of the complexity.

Use class decorators when you need to transform the class once, post-hoc. Use metaclasses when you need access to the class namespace before the class is assembled.
BLOCK_END

---

## EXAMPLE: A metaclass that enforces the pipeline protocol at definition time

BLOCK_START
We want every PipelineStage subclass to be required to implement a transform method — at class definition time, not at instantiation time.

Without a metaclass, the check happens when you call the method. The class exists for potentially a long time before anyone calls transform. In a large codebase, you might ship broken code that only errors on a code path exercised in production.

With a metaclass, the check happens the moment the class is defined. If someone writes a PipelineStage subclass without a transform method, they get a TypeError immediately — at import time, before any objects are created, before any data flows.

Here is the metaclass. Inherit from type. Override __new__, which is called during class creation with the class name, bases, and namespace dictionary. Check whether "transform" is in namespace. If not, and if the class is not the abstract base itself, raise TypeError. Return type.__new__(cls, name, bases, namespace) to complete class creation normally.

Apply it with metaclass=PipelineMeta in the base class definition. All subclasses inherit the enforcement automatically.

The critical question: when to use this instead of ABC? ABCMeta raises errors at instantiation time — when you call the constructor. Our metaclass raises errors at definition time. For a framework where incorrect subclasses should be caught during import, not during use, the metaclass is the stronger tool.
BLOCK_END

---

## CONCEPT: Dynamic attribute generation and __slots__ from annotations

BLOCK_START
One of the most powerful uses of metaclasses is generating class bodies dynamically — creating __slots__ automatically from annotations, eliminating the need to declare both.

Normally, to get __slots__ you must write the slot names twice: once in the annotation and once in the __slots__ tuple. They must stay in sync manually, which is an error-prone maintenance burden.

A metaclass can eliminate this duplication. In __new__, before calling type.__new__, read the annotations from the namespace. Build the __slots__ tuple from those annotation keys. Inject __slots__ into the namespace. Call type.__new__. The resulting class has slots generated from annotations automatically.

The implication: annotate fields once, get both type documentation and memory-efficient storage automatically. This is exactly how some ORM and serialization libraries generate their data models.

The last piece: generating __init__ dynamically. Given the slot names from annotations, construct an __init__ function at runtime using exec or by building the function signature with types and setattr calls. This is what dataclasses does under the hood — not magic, just code that writes code.

In the Big Project, this lets us define sensor record classes as pure annotations — Python generates all the wiring automatically.
BLOCK_END

---

## EXAMPLE: Full annotation-driven auto-slotted class with generated __init__

BLOCK_START
We define a metaclass called AutoSlotMeta. In __new__, it reads cls.__annotations__ from the namespace — or an empty dict if none exist. It builds a __slots__ tuple from those keys. It generates an __init__ that takes those keys as parameters and assigns them via setattr. It injects both __slots__ and __init__ into the namespace before calling type.__new__.

The SensorRecord class declares its metaclass as AutoSlotMeta and annotates six fields. No __slots__ line. No __init__ body. The metaclass generates both. Instances have no __dict__. Construction works with six positional arguments.

The benchmark: compare AutoSlotMeta-generated SensorRecord against a hand-written SlottedRecord. They perform identically because the underlying C-level slot mechanism is the same — the metaclass just automates the declaration.

The validation step: after creating ten thousand records, check that no instance has __dict__ using hasattr, and check that each slot value is accessible. Print memory comparison between the auto-generated class and a regular class with identical fields.

This pattern reaches its full power when combined with configuration-driven schema loading: read a JSON schema file, generate a Python class at runtime with the right slots and init — without writing a single class body by hand.
BLOCK_END
