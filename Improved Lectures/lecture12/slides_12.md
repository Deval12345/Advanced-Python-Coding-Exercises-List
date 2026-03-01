# Slides — Lecture 12: Exception Model & EAFP — Errors as Alternate Control Flow

---

## Slide 1
**Title:** From Context Managers to EAFP

- L11 guaranteed cleanup via __exit__ — the exception system made that guarantee possible
- Python exceptions are a first-class control flow mechanism, not just error signals
- EAFP: Easier to Ask Forgiveness than Permission — Python's official style
- Today: the philosophy, the hierarchy, the full try/except/else/finally contract, and custom exceptions

---

## Slide 2
**Title:** LBYL vs EAFP — Two Philosophies

- LBYL (Look Before You Leap): check preconditions before acting — the C tradition
- EAFP (Easier to Ask Forgiveness): attempt the operation; catch failures
- LBYL has TOCTOU race conditions: the file may disappear between the check and the open
- EAFP eliminates the race window — there is no check to go stale
- Python's documentation, Django, and SQLAlchemy all recommend EAFP

---

## Slide 3
**Title:** Example 12.1 — LBYL vs EAFP File Reading

- LBYL: os.path.exists() before open() — TOCTOU race between check and open
- EAFP: open() directly in try block — no check, no race window
- except FileNotFoundError handles the missing-file case cleanly
- EAFP version is shorter and reads like a description of intent
- This is the canonical example Python documentation uses to introduce EAFP

---

## Slide 4
**Title:** Exceptions as Alternate Control Flow

- Raising an exception exits the current frame and travels up the call stack
- Python tests each except clause top to bottom; first match wins
- Uncaught exceptions propagate all the way to the top-level runtime
- Multiple except clauses handle different failure modes independently
- Best practice: catch the most specific exception you can actually handle

---

## Slide 5
**Title:** Exception Hierarchy

- BaseException → Exception → (ValueError, TypeError, KeyError, AttributeError, OSError)
- OSError → FileNotFoundError, PermissionError, IsADirectoryError
- An except clause catches the named type and all its subclasses
- Catching OSError also catches FileNotFoundError — may be intended or a bug
- Never catch Exception blindly unless you immediately re-raise

---

## Slide 6
**Title:** Example 12.2 — Multi-Exception Handling for User Data Parsing

- KeyError caught first: the field was missing from the record entirely
- ValueError caught second: the field existed but could not be parsed as int
- AttributeError caught third: the field was None, not a string — .strip() fails
- Each clause uses "as" to bind the exception and include the detail in the message
- Most specific first; each branch handles exactly what it understands

---

## Slide 7
**Title:** The else and finally Clauses

- else: runs only when try completed without raising — the success-only path
- Putting success logic in else prevents except clauses from catching its errors
- finally: runs unconditionally — try succeeded, try raised, except ran, except re-raised
- Industry pattern: query in try, rollback in except, commit in else, close cursor in finally
- Before context managers, finally was the only unconditional cleanup mechanism

---

## Slide 8
**Title:** Example 12.3 — Database Transaction with try/except/else/finally

- try: begin transaction, query balance, raise ValueError if insufficient funds
- except ValueError: rollback, print reason, return False — known business failure
- except Exception: rollback, re-raise with bare raise — unknown failures propagate
- else: commit — only runs when no exception occurred; commit only on success
- finally: close cursor — always, regardless of which path was taken

---

## Slide 9
**Title:** Custom Exception Classes

- Subclass Exception (or a domain base) to define your own exception types
- Custom exceptions carry structured attributes, not just a message string
- Callers write precise except clauses rather than matching on message strings
- Every mature library has its own hierarchy: requests, SQLAlchemy, Django
- Anti-pattern: raise Exception("message") loses specificity and breaks callers

---

## Slide 10
**Title:** Example 12.4 — Custom Exception Hierarchy for a Payment Service

- PaymentError subclasses Exception — the base for all payment failures
- InsufficientFundsError carries required and available as numeric attributes
- CardDeclinedError carries a reason string attribute
- except clauses ordered most-specific first: InsufficientFundsError before PaymentError
- Caller accesses fundError.required and fundError.available — no string parsing needed

---

## Slide 11
**Title:** Custom Exception Design Rules

- Most specific except clause first — Python stops at the first match
- Re-raise unexpected exceptions with bare raise to preserve the original traceback
- Never bare except — it catches SystemExit and KeyboardInterrupt too
- Exception types are API: they document your failure modes in executable form
- Catch at the level where you have enough context to handle the failure meaningfully

---

## Slide 12
**Title:** Summary

- EAFP: attempt the operation and handle failure — no TOCTOU race conditions
- Exception hierarchy: catch the most specific type you can actually handle
- else runs on success only; finally runs unconditionally — together they complete the pattern
- Custom exception hierarchies are part of your public API and enable precise handling by callers
- Next lecture: Descriptors — programmable attribute access, another place Python exposes its internals

---
