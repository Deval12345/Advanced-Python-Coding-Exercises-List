Slide 0
Title: Previous Lecture Takeaway
Point 1: Iterators and generators keep memory bounded; today we look at per-object memory and how to reduce it for many instances.

Slide 1
Title: How Python Stores Attributes
Point 1: Normal objects store attributes in a per-instance dictionary; that flexibility has memory overhead.
Point 2: For many small objects, dictionary overhead can dominate; understanding this helps at scale.

Slide 2
Title: What slots does
Point 1: *slots* tells Python not to create a per-instance dictionary; reserves fixed slots for listed names; no dynamic attributes.
Point 2: Lower memory and often faster access; trade-off is flexibility versus memory.

Slide 3
Title: When to Use slots
Point 1: Use for many instances with a fixed set of attributes (e.g. Point with *x*, *y*); cannot add new attributes on instances.
Point 2: Good for data records, events, nodes; avoid when you need dynamic attributes or instance *dict*.

Slide 4
Title: Final Takeaways
Point 1: *slots* declares fixed attribute names and removes per-instance dictionary; use for high-volume fixed-schema data.
