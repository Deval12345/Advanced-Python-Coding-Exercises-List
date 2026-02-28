# Lecture 7 Slides: Closures, Lambda Functions, and Advanced Function Patterns

---

Slide 1, Title: Transition from Lecture 6
Point 1: Lecture 6 showed functions as first-class objects, stored, passed, and returned as values.
Point 2: A function factory returned a new function configured by its parameters.
Point 3: The open question was: how does the returned function remember the values it was created with.
Point 4: The answer is closures, the mechanism behind factories, callbacks, and functional patterns.

---

Slide 2, Title: The Problem Closures Solve
Point 1: Consider building an alert system that validates financial transactions against thresholds.
Point 2: One validator for high-value transactions over ten thousand, another for standard over one hundred.
Point 3: Without closures, each validator requires a class definition or repeated parameter passing.
Point 4: Closures allow the threshold to be baked into the function at creation time.

---

Slide 3, Title: What is a Closure
Point 1: A closure is a function that remembers variables from the scope where it was created.
Point 2: This memory persists even after the outer function has returned and its scope no longer exists.
Point 3: Python bundles the captured variables with the inner function as attached state.
Point 4: The inner function carries its creation context wherever it goes.

---

Slide 4, Title: Free Variables and the Closure Cell
Point 1: A free variable is used inside a function but defined in an enclosing scope.
Point 2: In makeThresholdValidator, thresholdAmount is a free variable for checkValue.
Point 3: Python stores captured free variables in closure cells attached to the function object.
Point 4: You can inspect these cells using the __closure__ attribute of any function.

---

Slide 5, Title: Closures in Industrial Systems
Point 1: Event-driven systems use closures so callbacks capture the context of registration time.
Point 2: Retry logic uses closures to capture the retry count and target function together.
Point 3: Caching wrappers use closures to attach cached data to the wrapped function.
Point 4: Payment processors use closures to bake merchant configuration into fee calculators.

---

Slide 6, Title: The nonlocal Keyword
Point 1: By default, any assignment inside a function creates a new local variable.
Point 2: This shadows any captured variable of the same name in the enclosing scope.
Point 3: The nonlocal keyword declares that assignment should modify the enclosing variable, not create a local one.
Point 4: nonlocal enables closures that can modify captured state, not just read it.

---

Slide 7, Title: Stateful Closures with nonlocal
Point 1: makeCounter defines callCount as zero in the outer scope.
Point 2: The inner function increment declares callCount as nonlocal, then increments it.
Point 3: Each call to increment modifies the same captured callCount variable.
Point 4: This produces a stateful callable without defining a class.

---

Slide 8, Title: Lambda Functions
Point 1: Lambda creates a small, anonymous function from a single expression.
Point 2: It produces the same kind of function object as def, with no attached name.
Point 3: Lambda is best used inline: as the key argument to sorted, or inside map or filter.
Point 4: Lambda is not appropriate for multi-line logic, complex conditions, or named reuse.

---

Slide 9, Title: When NOT to Use Lambda
Point 1: If the function body needs more than one expression, use def instead.
Point 2: If the logic is complex enough to require a comment, use def instead.
Point 3: If you are assigning the lambda to a variable for reuse, use def instead.
Point 4: Lambda is a readability tool, not a substitute for proper function definitions.

---

Slide 10, Title: Closures vs Classes for State
Point 1: A closure with captured state is structurally equivalent to a class with one method.
Point 2: Captured variables in a closure play the same role as instance attributes in a class.
Point 3: Closures are more concise for single-method, single-behavior stateful callables.
Point 4: Classes are better when multiple methods or explicit state inspection is needed.

---

Slide 11, Title: Final Takeaway
Point 1: Closures carry their creation context with them, enabling configuration baking and stateful callbacks.
Point 2: The nonlocal keyword allows closures to modify captured state across calls.
Point 3: Lambda provides inline anonymous functions for simple, single-expression transformations.
Point 4: Next lecture: iterators and generators, where sequences are produced lazily and on demand.
