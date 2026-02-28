# Lecture 7: Closures, Lambda Functions, and Advanced Function Patterns

---

## CONCEPT 0.1 — Transition from Lecture 6

In the previous lecture, we saw how functions can be stored, passed, and returned as values. We built a simple factory: a function that creates another function based on parameters. But something was missing. When that returned function ran, how did it remember the minimum amount? The answer is closures. Closures are the mechanism behind function factories, callbacks, and much of Python's functional style.

---

## CONCEPT 1.1 — The Problem Closures Solve

Imagine you are building an alert system. You need validators that check whether financial transactions exceed different thresholds. One validator for high-value transactions over ten thousand. Another for standard transactions over one hundred. In a language without closures, you would need to write a class for each validator, or pass the threshold as a parameter every single time. In Python, closures give you a better option: bake the threshold into the function itself.

---

## CONCEPT 1.2 — What is a Closure

A closure is a function that remembers variables from the scope where it was created, even after that outer scope no longer exists. When the outer function finishes and returns the inner function, Python does not discard the local variables of the outer function. Instead, it bundles them up with the inner function as captured state. The inner function carries its creation context wherever it goes.

---

## EXAMPLE 1.1 — makeThresholdValidator

Here we define a function named makeThresholdValidator that receives a parameter named thresholdAmount.
Inside it, we define another function named checkValue that receives a parameter named transactionValue.
The checkValue function returns True if transactionValue is greater than or equal to thresholdAmount.
The makeThresholdValidator function returns checkValue.

When we call makeThresholdValidator with ten thousand, we get back checkValue — but checkValue still has access to thresholdAmount, which is ten thousand, even though makeThresholdValidator has already returned. That is the closure. The returned function carries the captured thresholdAmount with it.

---

## CONCEPT 1.3 — Free Variables and the Closure Cell

The variable thresholdAmount is a free variable for checkValue. It is used inside checkValue but defined in the enclosing makeThresholdValidator scope. Python stores free variables in special storage called closure cells, which are attached to the function object. You can inspect them using the closure attribute of any function.

---

## CONCEPT 2.1 — Closures in Industrial Systems

Closures appear everywhere in real systems. Event-driven systems use a callback that captures context from when the event was registered. Retry logic captures the number of retries and the target function. Caching wrappers capture the cached data alongside the wrapped function. Middleware generators produce specialized middleware with baked-in configuration.

In a payment processor, a closure can bake the merchant configuration into a fee-calculation function. Instead of passing the config on every call, the function carries it.

---

## EXAMPLE 2.1 — makePaymentProcessor

Here we define a function named makePaymentProcessor that receives a parameter named feeRate and a parameter named currency.
Inside, we define a function named processPayment that receives a parameter named amount.
The processPayment function returns a dictionary with the currency, the original amount, and the computed fee using feeRate.
The makePaymentProcessor returns processPayment.

We create usdProcessor by calling makePaymentProcessor with a feeRate of zero point zero two and a currency of "USD". We create eurProcessor with a feeRate of zero point zero three and "EUR". Each processor independently remembers its own configuration.

---

## CONCEPT 3.1 — The nonlocal Keyword

By default, reading a captured variable works. But what if the closure needs to modify the captured value? For example, counting how many times the function has been called. Without nonlocal, Python would treat any assignment as creating a new local variable, shadowing the captured one.

The nonlocal keyword declares that an assignment inside the inner function should modify the variable in the enclosing scope, not create a new local one.

---

## EXAMPLE 3.1 — makeCounter with nonlocal

Here we define a function named makeCounter.
Inside, we define a variable named callCount and set it to zero.
We define an inner function named increment.
Inside increment, we use the keyword nonlocal with callCount.
We add one to callCount and return it.
We return the increment function.

When we create a counter and call increment multiple times, callCount persists between calls. The closure modifies the captured variable each time. This creates a function with built-in stateful behavior, without needing a class.

---

## CONCEPT 4.1 — Lambda Functions

Lambda is a way to create a small, anonymous function inline. It creates the same kind of function object as def, but it is limited to a single expression and has no name attached.

Lambda is most useful when you need a function briefly for a single operation. Sorting by a key field. Filtering with a condition. Mapping a simple transformation. In these cases, defining a full named function with def would be verbose.

---

## EXAMPLE 4.1 — Lambda for Sorting Transactions

Here we have a list named transactionList where each item is a dictionary with keys "amount" and "currency".
We use sorted with a key argument set to a lambda that extracts the "amount" key from each transaction.
The lambda takes a single parameter named record and returns record at the "amount" key.
This sorts the list by amount without defining a separate named function.

---

## CONCEPT 4.2 — When NOT to Use Lambda

Lambda should stay small and readable. If the function needs more than one expression, use def. If the lambda is complex enough to need a comment, use def. If you are assigning the lambda to a variable and never using it as an inline argument, use def. The goal is readability. Lambda is a convenience tool, not a replacement for proper function definitions.

---

## CONCEPT 5.1 — Closures vs Classes for State

A closure with captured state is structurally equivalent to a class with one method. The closure captures variables the way a class stores instance attributes. For simple, single-method behavior, closures are more concise. For complex state with multiple behaviors, classes are clearer.

Rule of thumb: if you need one callable behavior with private state, use a closure. If you need multiple methods or want to expose the state clearly, use a class.

---

## CONCEPT 6.1 — Final Takeaway

Closures allow functions to carry their creation context with them. They enable configuration baking, stateful callbacks, and a lightweight alternative to single-method classes. The nonlocal keyword allows closures to modify captured state. Lambda provides inline anonymous functions for simple transformations. Together, these tools make Python's function system remarkably expressive. In the next lecture, we move to iterators and generators, where functions that produce sequences transform how Python processes data.
