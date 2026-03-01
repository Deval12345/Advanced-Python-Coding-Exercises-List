# Lecture 28: Sharing NumPy Data with Multiprocessing — Zero-Copy Design
## speech_28.md — Instructor Spoken Narrative

---

Last lecture we established the cost spectrum of shared-state mechanisms — from Manager objects at milliseconds per access to multiprocessing.Value at nanoseconds. Today we apply that same principle to the data structure that dominates scientific Python: the NumPy array. This lecture is about one thing: getting large arrays from one process to another without making any copies.

Let me start with the problem. You have a multiprocessing pipeline. Workers process NumPy arrays — images, signals, feature matrices, sensor readings. The natural instinct is to put the array into a Queue and let the worker read it. Let me tell you exactly what happens when you do that.

pickle.dumps takes your array — its dtype, shape, strides, and every byte of its data — and writes all of it into a bytes buffer. That is copy one: the data now exists twice in the sending process's memory. Those bytes are written through a kernel pipe into the OS's pipe buffer. That is copy two. The receiving process reads the bytes and pickle.loads reconstructs the full NumPy array in the worker's memory. That is copy three.

For a 100-megabyte array, you have three copies, peak memory pressure of 300 megabytes in transit, and potentially two seconds of serialization overhead — before the worker has done a single computation. At 30 arrays per second, you need 1.8 gigabytes per second of pickling throughput. Python cannot sustain that. The Queue is the wrong tool for large arrays.

The right tool is shared memory. The insight is simple: a NumPy array is just a block of bytes with some metadata attached — dtype, shape, strides. If you can put those bytes into a memory region that multiple processes can see simultaneously, each process can wrap those bytes in its own NumPy array object. The array object is lightweight — just metadata. The data never moves. That is zero-copy sharing.

Here is how it works with multiprocessing.Array. You create the array in shared memory: multiprocessing.Array with ctypes.c_double and a length. This allocates a block of raw bytes in a memory region backed by the OS and shared across all processes that receive this Array reference. In the main process, you call numpy.frombuffer on the Array, giving it the dtype. frombuffer says: take these bytes as-is and give me a NumPy array that uses them directly as its data buffer. No copy. The NumPy array's data pointer points into the shared memory.

Now you pass the Array reference to your workers. Not the data — the Array reference. Each worker calls numpy.frombuffer on the same Array, getting its own lightweight view of the same physical memory. A worker that writes to its view — say, arr[start:end] equals arr[start:end] squared — is writing directly into shared memory. When all workers have joined, the main process's view already reflects all the changes, because the main process's frombuffer view also points to the same shared bytes.

This is Example 28.1. One million float64 values, four workers, each squaring a non-overlapping chunk in-place. After all workers join, the main process reads the result sum directly from its view. No Queue needed for result transfer. No IPC at all for the data — only the tiny initial argument passing when spawning the processes.

Now, multiprocessing.Array has one awkward aspect: it requires ctypes type codes. If you think in dtypes — float64, int32, complex128 — you have to look up the corresponding ctypes types. Python 3.8 introduced multiprocessing.shared_memory, which solves this. shared_memory is simpler: it gives you a named block of raw bytes. You pass the name string to workers. Workers open the block by name, then wrap it in a NumPy ndarray using the dtype and shape you pass as arguments. No ctypes. No type code lookup. Pure NumPy semantics.

The name-based access is particularly powerful. In a multi-process server, a data loading process fills a shared memory block with a dataset. The block's name is published to worker processes. Workers attach to the block by name and read data directly, all without the data loading process being involved in any subsequent transfer. The dataset lives in memory once; all workers see it.

Example 28.2 demonstrates this. We load 2,000,000 float64 values into a named shared memory block. Four workers each open the block by name, create a NumPy view, and compute mean, standard deviation, and maximum for their chunk. They send only the tiny statistics dictionaries back through a Queue. After all workers join, we close and unlink the shared memory. Unlink is important — it tells the OS to release the underlying resource. Forgetting to unlink leaves a ghost in the OS shared memory registry.

Now I want to be precise about when you need locks and when you do not, because this is where shared memory gets delicate.

Concurrent reads are always safe. If all workers only read from the shared array — reading different elements or even the same elements — no lock is needed. The OS memory mapping is read-only for these workers, and two simultaneous reads of the same memory address are perfectly safe.

Writes to non-overlapping regions are safe without locks. If worker zero writes to indices 0 through 249,999 and worker one writes to indices 250,000 through 499,999, they are writing to completely different memory addresses. No conflict. No lock needed. This is the partitioned-write pattern, and it is the most common design for parallel in-place transforms.

Writes to overlapping regions require locks. If two workers might write to the same index, you need a lock — either the Array's built-in lock via the lock argument, or an external multiprocessing.Lock. Without a lock, you have a data race: both processes read the same memory location, compute independently, and write back, with only one write surviving and the other being silently lost.

The standard production design avoids this entirely through partitioning. Divide the array into non-overlapping chunks. Assign each chunk to exactly one worker. No locks needed anywhere in the hot path. For reduction — combining results — each worker writes to a separate output slot, and the main process aggregates after join. This is MapReduce at the shared-memory level, and it is the reason parallel numerical computing can be both fast and correct without lock contention.

The discipline for this lecture: never send large arrays through a Queue. Design your pipeline so that data lives in shared memory and IPC carries only indices, names, and statistics. The array never moves; only process views of it are created and destroyed.
