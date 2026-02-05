
# Stage 1 — Designing a Pluggable Analytics Engine

This stage builds the foundation of a real evolving system.

We design a modular analytics backend that can:

- ingest data
- apply configurable processing strategies
- support plugins
- evolve without rewriting core code

This is not a toy example.

This mirrors real analytics/recommendation engines used in production systems.

Goal of Stage 1:

Teach architecture before performance.

We introduce:

- abstract base classes
- protocols
- plugin architecture
- strategy pattern
- extensibility design

All future stages will extend this system.

---

## Real-World Scenario

Imagine a company building a real-time recommendation engine.

It processes:

- user clicks
- product views
- search queries
- purchase events

Different teams want to plug in their own scoring logic:

- ranking team
- fraud team
- personalization team
- analytics team

The system must:

✔ allow new algorithms without rewriting core  
✔ isolate modules  
✔ be testable  
✔ support multiple strategies  
✔ scale later without redesign  

We need a pluggable architecture.

---

# Part 1 — Abstract Base Pipeline

We define a base interface for processing stages.

This prevents chaos.

Every plugin must obey the same contract.

---

## Abstract Stage Definition

```python
from abc import ABC, abstractmethod

class Stage(ABC):
    @abstractmethod
    def process(self, event: dict) -> dict:
        pass
```

This enforces structure.

All stages must implement `process`.

The engine can trust the interface.

---

# Part 2 — Concrete Strategies

Different teams implement their own logic.

---

## Ranking Strategy

```python
class RankingStage(Stage):
    def process(self, event):
        event["rank_score"] = len(event["user"]) * 2
        return event
```

---

## Fraud Detection Strategy

```python
class FraudStage(Stage):
    def process(self, event):
        event["fraud_flag"] = event["amount"] > 1000
        return event
```

Each stage is independent.

They don’t know about each other.

---

# Part 3 — Pipeline Engine

The engine runs stages in sequence.

```python
class Pipeline:
    def __init__(self, stages):
        self.stages = stages

    def run(self, event):
        for stage in self.stages:
            event = stage.process(event)
        return event
```

Now we compose behavior dynamically.

---

## Running the Pipeline

```python
pipeline = Pipeline([
    RankingStage(),
    FraudStage()
])

event = {"user": "alice", "amount": 1200}
print(pipeline.run(event))
```

We didn’t change engine code.

We swapped strategies.

This is extensible architecture.

---

# Part 4 — Protocol-Based Plugins

Abstract classes enforce inheritance.

Protocols enforce behavior.

Protocols allow duck typing with guarantees.

---

## Protocol Interface

```python
from typing import Protocol

class StageProtocol(Protocol):
    def process(self, event: dict) -> dict: ...
```

Now any object with `process` qualifies.

No inheritance required.

This allows third-party plugins.

---

## Plugin Without Inheritance

```python
class AnalyticsPlugin:
    def process(self, event):
        event["analytics_tag"] = "tracked"
        return event
```

The engine accepts it.

Because it obeys the protocol.

This enables ecosystem growth.

---

# Part 5 — Plugin Loader

Real systems load modules dynamically.

We simulate a plugin registry.

```python
PLUGIN_REGISTRY = []

def register(plugin):
    PLUGIN_REGISTRY.append(plugin)

register(RankingStage())
register(FraudStage())
register(AnalyticsPlugin())

pipeline = Pipeline(PLUGIN_REGISTRY)
```

Teams add plugins.

Core system stays untouched.

---

# Part 6 — Why This Architecture Matters

Without this structure:

- teams fork code
- pipelines duplicate logic
- updates break unrelated modules
- scaling becomes impossible

With this architecture:

✔ safe extensibility  
✔ independent teams  
✔ predictable behavior  
✔ testable components  
✔ future concurrency ready  

This foundation supports every later stage.

---

# Exercises

1. Add a new plugin that scores engagement.
2. Reorder pipeline dynamically at runtime.
3. Write a test verifying stage independence.
4. Implement a plugin that logs every event.

These tasks simulate real engineering extensions.

---

# Final Insight

Pluggable architecture is not optional.

It is what allows systems to evolve
without collapsing.

Stage 1 builds the skeleton
that all later performance layers depend on.
