Slide 1, Title: From Custom Containers to Protocols
Point 1: In Lecture 2, we built custom containers using special dunder methods.
Point 2: The next question is: how does Python decide if an object is compatible with a system?
Point 3: The answer is protocols — behavioral agreements that govern compatibility.

Slide 2, Title: The Problem with Rigid Type Systems
Point 1: Java and C++ require compile-time interface declarations for compatibility.
Point 2: This prevents adding interfaces to third-party library classes.
Point 3: Python chose behavioral compatibility — if the object has the right methods, it works.
Point 4: This decision enables Django, data science tools, and web frameworks to connect freely.

Slide 3, Title: What is a Protocol?
Point 1: A protocol is an informal agreement: implement these methods to participate in this system.
Point 2: The sequence protocol requires special dunder method len and special dunder method getitem.
Point 3: The iteration protocol requires special dunder method iter.
Point 4: No registration, no declaration — Python checks for methods at runtime.

Slide 4, Title: Duck Typing — Behavior Over Type
Point 1: "If it walks like a duck and quacks like a duck, it is a duck."
Point 2: Python's built-ins (len, sum, sorted, for) never check isinstance.
Point 3: They call the relevant method. If it works, compatibility is established.
Point 4: Type is irrelevant — behavior is everything.

Slide 5, Title: The Industrial Power of Duck Typing
Point 1: print() writes to files, StringIO, network sockets — any object with write().
Point 2: The for loop works on lists, generators, database results, custom streams.
Point 3: NumPy, Pandas, and PyTorch tensors are interchangeable in many operations.
Point 4: Mock objects for testing require only the right methods — no inheritance needed.

Slide 6, Title: The Consenting Adults Philosophy
Point 1: Python trusts developers — no forced enforcement of protocol correctness.
Point 2: Incorrect protocol implementations fail at runtime, not at design time.
Point 3: Tradeoff: enormous flexibility in exchange for developer responsibility.
Point 4: Well-structured code with good test coverage makes runtime failures rare.

Slide 7, Title: Runtime Failure Modes for Protocols
Point 1: Missing method entirely: raises AttributeError at the point of use.
Point 2: Method exists but behaves incorrectly: raises TypeError.
Point 3: These errors surface during testing — not silently in production.
Point 4: The flexibility benefit far outweighs the design-time uncertainty.

Slide 8, Title: The File-Like Protocol
Point 1: Any object with read(), write(), and close() works wherever a file is expected.
Point 2: Covers gzip files, network sockets, StringIO, BytesIO, HTTP responses.
Point 3: No shared base class, no registration — just behavioral agreement.
Point 4: This is the most widely used protocol in the Python ecosystem.

Slide 9, Title: How Protocols Make Python the Glue Language
Point 1: Any data source connects to any data consumer that speaks the same protocol.
Point 2: A compression library reads from files, sockets, or in-memory buffers identically.
Point 3: A network parser accepts any object with a read method.
Point 4: Behavioral contracts are the connective tissue of the Python ecosystem.

Slide 10, Title: Lecture 3 Takeaways
Point 1: Protocols are informal behavioral agreements — no declarations required.
Point 2: Duck typing evaluates compatibility through methods, not type hierarchy.
Point 3: The consenting adults philosophy gives flexibility with developer responsibility.
Point 4: Next lecture: how protocols enable design patterns and plugin architectures.
