
---

## 📄 `asyncio_event_loop.md`

```markdown
# The asyncio Event Loop

## Topic
Event loop and cooperative multitasking.

---

## Problem Scenario
Run many waiting tasks efficiently without blocking.

---

## Code Example

```python
import asyncio

async def work():
    await asyncio.sleep(1)

asyncio.run(work())

