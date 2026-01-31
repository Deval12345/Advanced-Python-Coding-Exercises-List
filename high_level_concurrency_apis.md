
---

## 📄 `high_level_concurrency_apis.md`

```markdown
# High-level Concurrency APIs

## Topic
Structured concurrency with asyncio.

---

## Problem Scenario
Launch many tasks concurrently and wait for completion.

---

## Code Example

```python
await asyncio.gather(task1(), task2())

