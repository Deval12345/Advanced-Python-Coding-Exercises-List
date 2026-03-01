Slide 0
Title: Welcome back

Point 1: Previous lecture: special methods *len*, *getitem*, *iter*, and *add* let objects participate in Python syntax; behavior matters more than inheritance.

Point 2: Today: how Python decides compatibility based purely on behavior: protocols and informal interfaces.

Slide 1
Title: Informal Interfaces

Point 1: Python checks for required behavior at runtime instead of enforcing inheritance

Point 2: Implementing *len* and *getitem* makes an object behave like a sequence through demonstrated behavior

Slide 2
Title: -
Point 1: Compatibility is determined by capability such as supporting iteration and addition rather than by type or ancestry

Slide 3
Title: -
Point 1: Objects become interchangeable when they share method signatures enabling loosely coupled systems

Slide 4
Title: -
Point 1: Implementing required methods satisfies a protocol without inheritance enforcement

Slide 5
Title: -
Point 1: Python prioritizes behavior, informal interfaces, duck typing, and protocol-driven extensibility where compatibility comes from capability
