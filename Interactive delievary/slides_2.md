Slide 1
Title: From Syntax Mechanics to Structural Compatibility

Point 1: The previous lesson explained how Python syntax maps to special methods.
Point 2: This lesson shifts focus from syntax mechanics to structural compatibility.
Point 3: The central question now is how Python decides whether an object qualifies to participate in a system.

Slide 2
Title: Capability Determines System Fit

Point 1: Python evaluates whether something fits into a system.
Point 2: Compatibility is determined by what an object can do.
Point 3: Qualification is based on behavior, not class hierarchy.

Slide 3 Title: Informal Interfaces

Point 1: Other languages require explicit interface inheritance and
enforcement. Point 2: Python does not require explicit declarations.
Point 3: Python checks compatibility at runtime. Point 4: The question
is whether required methods exist right now.

Slide 4 Title: Demonstrated Compatibility

Point 1: An object can behave like a sequence without explicit
declaration. Point 2: This is an informal interface. Point 3:
Compatibility is demonstrated through behavior, not declared.

Slide 5 Title: Duck Typing

Point 1: If it behaves correctly, it is accepted. Point 2: Python checks
capabilities, not ancestry. Point 3: Functions assume required
behaviors, such as iteration and addition. Point 4: Python does not
check type explicitly, only behavior. Point 5: This principle is duck
typing.

Slide 6 Title: Behavior Driven Design

Point 1: Behavior defines compatibility. Point 2: Focus on shared
behavior instead of deep inheritance trees. Point 3: Objects can share
behavior without sharing ancestry. Point 4: Shared capability enables
interchangeable components. Point 5: Functions depend only on required
behavior. Point 6: This resembles a strategy style approach. Point 7:
Behavior driven systems are easier to extend and maintain. Point 8: New
behaviors can be added without modifying existing classes.

Slide 7 Title: Protocol Based Extensibility

Point 1: Extensibility depends on required behavior such as write. Point
2: Any object providing required behavior can substitute. Point 3:
Systems remain unaware of concrete implementations.

Slide 8 Title: Static Structural Protocols

Point 1: Protocols make expectations explicit for type checkers. Point
2: Tools can warn about missing required behavior. Point 3: Runtime
behavior remains unchanged. Point 4: Type checking enforces expectations
while preserving flexibility.

Slide 9 Title: Core Design Principles

Point 1: Python favors behavior over inheritance. Point 2: Python
supports informal interfaces. Point 3: Python encourages trust based
design. Point 4: Python enables protocol driven extensibility. Point 5:
Compatibility comes from capability, not hierarchy. Point 6: Design by
required behavior. Point 7: Do not begin design with inheritance.

Slide 10 Title: Refactoring Toward Behavior

Point 1: Remove inheritance used only for compatibility. Point 2:
Replace it with behavior based design or protocols. Point 3: Document
structural changes and design improvements.
