# Code — Lecture 33: Big Project Stage 1 — Pluggable Analytics Engine

---

## Example 33.1 — Source Protocol: Base interface and first implementations

```python
# Example 33.1
import csv                                             # line 1
import random                                          # line 2
import time                                            # line 3
from pathlib import Path                               # line 4

class CsvFileSource:                                   # line 5
    """Source protocol: read sensor records from a CSV file."""  # line 6
    def __init__(self, filePath, sensorId="unknown"):  # line 7
        self.filePath = Path(filePath)                 # line 8
        self.sensorId = sensorId                       # line 9

    def stream(self):                                  # line 10  generator method
        with open(self.filePath, newline="") as f:     # line 11
            reader = csv.DictReader(f)                 # line 12
            for row in reader:                         # line 13
                yield {                                # line 14
                    "sensorId": self.sensorId,         # line 15
                    "timestamp": float(row["timestamp"]),  # line 16
                    "value": float(row["value"]),       # line 17
                    "unit": row.get("unit", "unknown") # line 18
                }                                      # line 19

class SyntheticSource:                                 # line 20
    """Source protocol: generate synthetic sensor readings for testing."""  # line 21
    def __init__(self, sensorId, meanValue=50.0, stddev=10.0, numRecords=None):  # line 22
        self.sensorId = sensorId                       # line 23
        self.meanValue = meanValue                     # line 24
        self.stddev = stddev                           # line 25
        self.numRecords = numRecords                   # line 26  None = infinite

    def stream(self):                                  # line 27
        count = 0                                      # line 28
        while self.numRecords is None or count < self.numRecords:  # line 29
            yield {                                    # line 30
                "sensorId": self.sensorId,             # line 31
                "timestamp": time.time(),              # line 32
                "value": random.gauss(self.meanValue, self.stddev),  # line 33
                "unit": "celsius"                      # line 34
            }                                          # line 35
            count += 1                                 # line 36
            time.sleep(0.001)                          # line 37  simulate sensor polling rate

if __name__ == "__main__":                             # line 38
    print("--- SyntheticSource demonstration ---")     # line 39
    synth = SyntheticSource(sensorId="TEMP_01", numRecords=5)  # line 40
    for record in synth.stream():                      # line 41
        print(f"  {record['sensorId']}: {record['value']:.2f}{record['unit']} at {record['timestamp']:.3f}")  # line 42
    print("Source protocol demonstration complete")    # line 43
```

**Line-by-line explanation:**

- **Line 4:** `from pathlib import Path` — `Path` normalizes file paths across platforms and provides safer file handling than raw strings.
- **Line 10:** `def stream(self):` — the critical naming convention. Any class implementing the Source protocol must have a method called `stream`. This is not enforced by any base class; it is the informal contract.
- **Line 11:** `with open(self.filePath, newline="") as f:` — the context manager ensures the file is closed even if an exception occurs during iteration. `newline=""` is required for `csv.DictReader` to handle line endings correctly across platforms.
- **Line 12:** `csv.DictReader(f)` — reads each row as an `OrderedDict` keyed by column headers. Memory-constant: one row is loaded per iteration, not the entire file.
- **Line 13-19:** `for row in reader: yield {...}` — the `yield` turns `stream` into a generator method. Execution pauses here until the caller asks for the next record. The file stays open during all pauses because the generator's stack frame retains the `with` block's context.
- **Line 22:** `numRecords=None` — `None` means infinite: the source generates readings forever, simulating a live sensor feed. When `numRecords` is set, the source stops after exactly that many records — useful for testing.
- **Line 33:** `random.gauss(self.meanValue, self.stddev)` — Gaussian distribution simulates real sensor noise: values cluster around the mean with variability determined by standard deviation.
- **Line 37:** `time.sleep(0.001)` — 1ms sleep simulates a 1000 Hz sensor polling rate. In production, this would be replaced by an actual hardware read or an async socket receive.

---

## Example 33.2 — Pipeline stages: ThresholdFilter, NormalizeTransform, and MovingAverageTransform

```python
# Example 33.2
from collections import deque                         # line 1

class ThresholdFilter:                                 # line 2
    """Pipeline stage: drop records outside [minVal, maxVal]."""  # line 3
    def __init__(self, minVal, maxVal):                # line 4
        self.minVal = minVal                           # line 5
        self.maxVal = maxVal                           # line 6

    def process(self, inputStream):                    # line 7  generator method
        for record in inputStream:                     # line 8
            if self.minVal <= record["value"] <= self.maxVal:  # line 9
                yield record                           # line 10
            # records outside range are simply not yielded (silently dropped)

class NormalizeTransform:                              # line 11
    """Pipeline stage: scale values to [0, 1] range."""  # line 12
    def __init__(self, minVal, maxVal):                # line 13
        self.minVal = minVal                           # line 14
        self.range = maxVal - minVal                   # line 15

    def process(self, inputStream):                    # line 16
        for record in inputStream:                     # line 17
            normalized = (record["value"] - self.minVal) / self.range  # line 18
            yield {**record, "value": round(normalized, 6), "unit": "normalized"}  # line 19

class MovingAverageTransform:                          # line 20
    """Pipeline stage: add rolling average over last N records."""  # line 21
    def __init__(self, windowSize):                    # line 22
        self.windowSize = windowSize                   # line 23
        self.window = deque(maxlen=windowSize)         # line 24

    def process(self, inputStream):                    # line 25
        for record in inputStream:                     # line 26
            self.window.append(record["value"])        # line 27
            rollingAvg = sum(self.window) / len(self.window)  # line 28
            yield {**record, "rollingAvg": round(rollingAvg, 6)}  # line 29

def buildPipeline(source, stages):                     # line 30
    """Compose a source and a list of stages into a single lazy generator."""  # line 31
    stream = source.stream()                           # line 32
    for stage in stages:                               # line 33
        stream = stage.process(stream)                 # line 34
    return stream                                      # line 35

if __name__ == "__main__":                             # line 36
    source = SyntheticSource(sensorId="TEMP_01", numRecords=20)  # line 37
    stages = [                                         # line 38
        ThresholdFilter(minVal=30.0, maxVal=70.0),     # line 39
        NormalizeTransform(minVal=30.0, maxVal=70.0),  # line 40
        MovingAverageTransform(windowSize=5),          # line 41
    ]                                                  # line 42
    pipeline = buildPipeline(source, stages)           # line 43
    count = 0                                          # line 44
    for record in pipeline:                            # line 45
        count += 1                                     # line 46
        print(f"Record {count}: sensor={record['sensorId']}, value={record['value']:.4f}, rolling={record.get('rollingAvg', 'N/A')}")  # line 47
    print(f"Pipeline processed {count} records (after filtering)")  # line 48
```

**Line-by-line explanation:**

- **Line 1:** `from collections import deque` — `deque(maxlen=N)` is the standard Python structure for a fixed-size sliding window. When full, adding a new item automatically removes the oldest one — O(1) for both operations.
- **Lines 7-10:** `ThresholdFilter.process` is a generator method. The `for record in inputStream` loop drives the upstream generator one record at a time. Records that pass the threshold check are `yield`ed; others are silently skipped by the `if` statement without a `yield`.
- **Line 15:** `self.range = maxVal - minVal` — pre-compute the denominator once at construction rather than on every record. This is a minor performance optimization that becomes significant when processing millions of records.
- **Line 18:** `(record["value"] - self.minVal) / self.range` — linear normalization to [0, 1]. Values at or below `minVal` map to 0.0; values at or above `maxVal` map to 1.0. Values outside the range would normalize outside [0, 1]; the ThresholdFilter upstream prevents this.
- **Line 19:** `{**record, "value": ..., "unit": ...}` — dict unpacking creates a new dictionary with all existing keys from `record`, then overrides the "value" and "unit" keys. This is the standard Python pattern for creating a modified copy of a dictionary without mutating the original.
- **Line 24:** `self.window = deque(maxlen=windowSize)` — the window is instance state that persists between `process` generator yields. This is safe because generator execution is single-threaded and sequential — no two calls to `process` execute simultaneously.
- **Line 28:** `sum(self.window) / len(self.window)` — simple mean over the window. For the first `windowSize` records, the window is smaller than full, so the rolling average is computed over however many records are available. This warm-up behavior is correct: the average is accurate for the records seen so far.
- **Lines 30-35:** `buildPipeline` demonstrates the power of generator composition. The `stream` variable is reassigned on each loop iteration, wrapping the previous generator in the new stage. After the loop, `stream` is a generator that, when iterated, lazily pulls from all stages in order. This function makes the composition explicit and reusable.
- **Lines 38-42:** The stages list defines the pipeline declaratively. Changing the order of stages, adding stages, or removing stages is a change to this list — not a change to any stage class or to the buildPipeline function.
