Slide 0
Title: Previous Lecture Takeaway
Point 1: Decorators add behavior around function calls; context managers add guaranteed setup and cleanup around a block.

Slide 1
Title: Why Context Managers Exist
Point 1: Cleanup must run on every path (success, failure, early return); context managers make cleanup structural.
Point 2: *with* ensures a designated exit method runs when leaving the block; used for files, sessions, locks.

Slide 2
Title: How with Works
Point 1: *with expr as var* evaluates expr, calls *enter*, binds return value to var, runs block, then always calls *exit*.
Point 2: *exit* receives exception info; return True to suppress, False or None to propagate.

Slide 3
Title: A Timing Context Manager
Point 1: Context managers can measure time, switch config, or log; same pattern: *enter* for setup, *exit* for cleanup.

Slide 4
Title: Files and Built-in Support
Point 1: *open()* returns a context manager; *with open(...) as f* closes the file when the block ends.

Slide 5
Title: Final Takeaways
Point 1: *enter* and *exit* guarantee cleanup; *exit* always runs; use *with* for resources and setup/cleanup pairs.
