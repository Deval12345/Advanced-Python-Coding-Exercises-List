Slide 0
Title: Previous Lecture Takeaway
Point 1: Context managers rely on exceptions; *exit* runs even when the block raises; Python treats exceptions as part of control flow.

Slide 1
Title: Two Styles: EAFP and LBYL
Point 1: EAFP: do the operation, handle the exception; LBYL: check first then act; EAFP keeps the happy path linear.
Point 2: EAFP can be safer under concurrency because state cannot change between check and use.

Slide 2
Title: Why EAFP Fits Python
Point 1: Operations raise exceptions on failure; catching in *except* keeps main flow clean; no need to pre-check every possibility.

Slide 3
Title: Defensive Code Versus EAFP
Point 1: EAFP inverts structure: common case in one block, rare case in *except*; matches production where most requests succeed.

Slide 4
Title: Exceptions as Control Flow
Point 1: Exceptions signal that an operation could not be completed; catch specific types for expected failures; let unexpected ones propagate.

Slide 5
Title: Final Takeaways
Point 1: EAFP keeps happy path linear and failure in *except*; avoids check-then-use races; use *try*/*except* for expected failures, specific types, let rest propagate.
