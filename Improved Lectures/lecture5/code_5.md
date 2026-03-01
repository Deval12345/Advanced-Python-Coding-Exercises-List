# Code — Lecture 5
# Memory Layout and Python Object Model: Identity, Mutability, and Copying

---

# Example 5.1 — Aliasing: Two Names, One Object

```python
# line 1: create a list of product names and bind the name productList to it
productList = ["laptop", "phone", "tablet"]

# line 2: bind catalogAlias to the same object — this is NOT a copy
catalogAlias = productList

# line 3: append a new item through catalogAlias
catalogAlias.append("monitor")

# line 4: both names show the updated list because they share one object
print(productList)    # line 5: ["laptop", "phone", "tablet", "monitor"]
print(catalogAlias)   # line 6: ["laptop", "phone", "tablet", "monitor"]

# line 7: confirm identity — both names point to the same object in memory
print(productList is catalogAlias)  # line 8: True
```

---

# Example 5.2 — Identity vs Equality: is vs ==

```python
# line 1: create two separate list objects with identical content
firstList = [1, 2, 3]
secondList = [1, 2, 3]

# line 2: create an alias — another name for the exact same object as firstList
aliasOfFirst = firstList

# line 3: equality check — do they have the same values?
print(firstList == secondList)    # line 4: True  — same content
print(firstList == aliasOfFirst)  # line 5: True  — same content

# line 6: identity check — are they the exact same object in memory?
print(firstList is secondList)    # line 7: False — separate objects
print(firstList is aliasOfFirst)  # line 8: True  — same object

# line 9: confirm different memory addresses for firstList and secondList
print(id(firstList))              # line 10: e.g. 4391820352
print(id(secondList))             # line 11: e.g. 4391821504 (different address)
print(id(aliasOfFirst))           # line 12: same address as firstList
```

---

# Example 5.3 — The None Rule: is None vs == None

```python
# line 1: a safe function that checks its argument using is None
def processPayload(payloadData=None):
    # line 2: correct — uses is None for reliable identity check
    if payloadData is None:
        print("No payload provided.")   # line 3
        return                          # line 4

    # line 5: process normally if a real payload was passed
    print(f"Processing payload with {len(payloadData)} items.")  # line 6


# line 7: demonstrate why == None is dangerous
class UnreliableWrapper:
    # line 8: override __eq__ to always return True — any comparison returns True
    def __eq__(self, otherValue):
        return True  # line 9: dangerously broken equality

# line 10: create an instance of the unreliable wrapper
unreliableInstance = UnreliableWrapper()

# line 11: == None incorrectly returns True — the object is not None!
print(unreliableInstance == None)   # line 12: True  — WRONG and dangerous
# line 13: is None correctly returns False — the object is not the None singleton
print(unreliableInstance is None)   # line 14: False — correct
```

---

# Example 5.4 — Mutable Aliasing: The False Backup Pattern

```python
# line 1: create a list of exam scores
scoreList = [85, 90, 75]

# line 2: this looks like a backup but is only an alias
backupRef = scoreList

# line 3: append a new score through scoreList
scoreList.append(95)

# line 4: backupRef is NOT independent — it shows the updated list too
print(scoreList)   # line 5: [85, 90, 75, 95]
print(backupRef)   # line 6: [85, 90, 75, 95] — not a real backup

# line 7: confirm they are the same object
print(scoreList is backupRef)  # line 8: True

# line 9: to create a real independent backup, use a shallow copy
import copy
realBackup = copy.copy(scoreList)   # line 10: new list, independent outer container

scoreList.append(100)               # line 11: append another score

print(scoreList)   # line 12: [85, 90, 75, 95, 100]
print(realBackup)  # line 13: [85, 90, 75, 95] — realBackup was not affected
```

---

# Example 5.5 — Shallow vs Deep Copy on Nested Configuration

```python
import copy  # line 1: import the copy module

# line 2: define a nested configuration dictionary
originalConfig = {
    "settings": {"timeout": 30, "retries": 3},  # line 3: inner mutable dict
    "active": True                               # line 4: immutable boolean
}

# line 5: create a shallow copy — new outer dict, shared inner dict
shallowConfig = copy.copy(originalConfig)

# line 6: create a deep copy — fully independent at every level
deepConfig = copy.deepcopy(originalConfig)

# line 7: modify the nested timeout through shallowConfig
shallowConfig["settings"]["timeout"] = 60

# line 8: the original is also changed because shallow copy shared the inner dict
print(originalConfig["settings"]["timeout"])  # line 9: 60 — CHANGED unexpectedly

# line 10: modify the nested timeout through deepConfig
deepConfig["settings"]["timeout"] = 999

# line 11: the original is not affected because deep copy made a fully independent copy
print(originalConfig["settings"]["timeout"])  # line 12: 60 — unchanged by deepConfig
print(deepConfig["settings"]["timeout"])      # line 13: 999 — independent

# line 14: confirm inner dict identity for shallow copy vs deep copy
print(shallowConfig["settings"] is originalConfig["settings"])  # line 15: True  — shared
print(deepConfig["settings"] is originalConfig["settings"])     # line 16: False — independent
```

---

# Example 5.6 — Truth-Value Testing with __len__ on a Custom Class

```python
# line 1: define a task queue that supports truthiness via __len__
class TaskQueue:
    def __init__(self):
        self.taskList = []   # line 2: internal list to hold task names

    def addTask(self, taskName):
        self.taskList.append(taskName)   # line 3: add a task by name

    def removeNextTask(self):
        if self.taskList:                         # line 4: truthy check on built-in list
            return self.taskList.pop(0)           # line 5: remove and return first task
        return None                               # line 6: return None if empty

    def __len__(self):
        return len(self.taskList)   # line 7: Python uses this for bool context


# line 8: create an empty queue
processingQueue = TaskQueue()

# line 9: queue is empty — __len__ returns 0, which is falsy
if not processingQueue:
    print("Queue is empty, nothing to process.")   # line 10

# line 11: add tasks to the queue
processingQueue.addTask("sendWelcomeEmail")    # line 12
processingQueue.addTask("generateInvoice")    # line 13

# line 14: queue now has 2 items — __len__ returns 2, which is truthy
if processingQueue:
    print(f"Queue has {len(processingQueue)} tasks. Begin processing.")  # line 15

# line 16: process all tasks using truthiness as the loop condition
while processingQueue:                          # line 17: stops when __len__ returns 0
    nextTask = processingQueue.removeNextTask()
    print(f"Processing: {nextTask}")            # line 18
```

---

# Example 5.7 — The Sentinel Pattern for Unambiguous Defaults

```python
# line 1: create a unique sentinel object at module level
MISSING = object()

# line 2: define a configuration update function using MISSING as default
def updateCacheTimeout(newTimeout=MISSING):
    # line 3: check if no argument was provided at all
    if newTimeout is MISSING:
        print("No timeout argument provided. Retaining current value.")  # line 4
        return                                                             # line 5

    # line 6: check if None was explicitly passed to clear the setting
    elif newTimeout is None:
        print("Timeout explicitly set to None. Disabling cache timeout.")  # line 7
        return                                                               # line 8

    # line 9: an actual numeric value was provided
    else:
        print(f"Updating cache timeout to {newTimeout} seconds.")  # line 10


# line 11: call with no argument — triggers the MISSING branch
updateCacheTimeout()          # line 12: No timeout argument provided. Retaining current value.

# line 13: call with None explicitly — triggers the None branch
updateCacheTimeout(None)      # line 14: Timeout explicitly set to None. Disabling cache timeout.

# line 15: call with an actual value — triggers the update branch
updateCacheTimeout(120)       # line 16: Updating cache timeout to 120 seconds.

# line 17: confirm the sentinel's unique identity
anotherSentinel = object()
print(MISSING is anotherSentinel)  # line 18: False — every object() call produces a unique object
print(MISSING is MISSING)          # line 19: True  — same object, same identity
```
