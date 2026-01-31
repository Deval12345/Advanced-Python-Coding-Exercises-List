# Async I/O and Event Loops in Python

This file explains **what an event loop is**, how it schedules asynchronous tasks,
and why **blocking operations break async programs**.

This topic builds directly on:
- `async / await` fundamentals
- I/O-bound vs CPU-bound work

It is essential for understanding:
- asyncio internals
- why async scales for I/O
- how real async frameworks work

(Reference: *Fluent Python*, Part V – asyncio and event loops)

---

## 1. Why Event Loops Exist (Problem First)

`async / await` alone is not enough.

Once you write:

```python
async def fetch():
    await something()
