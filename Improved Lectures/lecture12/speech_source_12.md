# Speech Source — Lecture 12: Exception Model & EAFP — Errors as Alternate Control Flow

---

## CONCEPT 0.1 — Transition from Context Managers

In the previous lecture, we saw that context managers guarantee cleanup by making the __exit__ call structural — the language enforces it, not programmer discipline. The reason Python could design context managers this way is that the exception system is a first-class citizen of the language, not an afterthought bolted on. Exceptions in Python are not just error signals; they are a control flow mechanism that the entire standard library is built around. That design philosophy has a name: EAFP — Easier to Ask Forgiveness than Permission. Context managers are one expression of it; today we examine the philosophy itself.

---

## CONCEPT 1.1 — The Two Philosophies: LBYL vs EAFP

Every programming language community has to decide how to handle uncertain operations — file reads that might fail, dictionary lookups for keys that might not exist, network calls that might time out. The C tradition answers with LBYL: Look Before You Leap. Check that the file exists before you open it. Check that the key is in the dictionary before you access it. Check that the user is authorized before you execute the operation. This defensive style puts precondition checks before every action. The problem is that precondition checks can become stale. The file might exist when you check and disappear before you open it — a race condition known as TOCTOU: Time Of Check, Time Of Use. Distributed systems and multi-threaded applications make TOCTOU failures far more likely and far harder to debug. Python's answer is EAFP: attempt the operation, then handle the failure if it occurs. The try block contains the optimistic path; the except clause handles the failure. There are no stale checks because there are no checks at all — you simply attempt the thing and deal with what happens.

---

## CONCEPT 1.2 — Exceptions as Alternate Control Flow

Python's exception system is not just a mechanism for crashing with an informative message. Any code can raise any exception, and any except clause anywhere in the call stack can catch and handle it. This makes exceptions a form of non-local control flow — raising an exception jumps execution out of the current frame, out of any nested frames, up the call stack until a matching except clause is found. When the standard library's open() call cannot find a file, it raises FileNotFoundError. When a dictionary is accessed with a missing key, it raises KeyError. These are not crashes waiting to happen; they are structured signals that calling code can choose to handle or allow to propagate. Multiple except clauses on the same try block allow different failure modes to be handled differently — a missing key is handled one way, a type conversion failure another way, a permission error a third way. Specificity in exception handling is a professional discipline: catching the exact exception type you can handle, and letting everything else propagate for a higher layer to deal with.

---

## EXAMPLE 1.1 — LBYL vs EAFP File Reading

This example puts both philosophies side by side on the same problem: read a file and return its contents, or return None if the file does not exist.

The LBYL version calls os.path.exists() before attempting to open the file. This looks safe, but there is a window between the exists check and the open call — another process could delete the file in that window, and the open would fail with an unhandled exception despite the check passing.

The EAFP version skips the check entirely. It attempts the open inside a try block. If the file is not there, Python raises FileNotFoundError, which the except clause catches cleanly and returns None. There is no race window because there is no pre-check — the file is either opened or it is not, atomically, from the caller's perspective.

Notice how the EAFP version is also shorter and reads more like a description of intent: try to open it, and if it is not there, return None.

```python
# Example 12.1
import os

# LBYL approach — problematic
def readFileLBYL(filePath):                            # line 1
    if os.path.exists(filePath):                       # line 2
        with open(filePath) as fileHandle:             # line 3
            return fileHandle.read()                   # line 4
    return None                                        # line 5

# EAFP approach — Pythonic
def readFileEAFP(filePath):                            # line 6
    try:                                               # line 7
        with open(filePath) as fileHandle:             # line 8
            return fileHandle.read()                   # line 9
    except FileNotFoundError:                          # line 10
        return None                                    # line 11

result = readFileEAFP("config.json")                   # line 12
print(result)                                          # line 13
```

---

## CONCEPT 2.1 — Exception Hierarchy and Specificity

Python's exception types are organized in a class hierarchy. At the root is BaseException, which covers system-level exits and keyboard interrupts. Below it sits Exception, which covers all ordinary application-level errors. From Exception descend the specific types your code deals with daily: ValueError for bad values, TypeError for wrong types, KeyError for missing dictionary keys, AttributeError for missing attributes, OSError for operating system failures — and under OSError sit FileNotFoundError and PermissionError, which are more specific still. This hierarchy matters because an except clause catches the named type and all its subclasses. Catching OSError catches FileNotFoundError too. Catching Exception catches almost everything. The rule is always to catch the most specific exception you can actually handle. If you catch OSError when you only know how to handle FileNotFoundError, you will silently swallow PermissionError, disk full errors, and network filesystem failures that need different handling or need to propagate. Catching a bare Exception — or worse, a bare except with no type — is almost always wrong outside of top-level framework code. The multiple except clause pattern lets you be precise: list FileNotFoundError first, PermissionError second, OSError last as a fallback, each with its own handling logic.

---

## EXAMPLE 2.1 — Multi-Exception Handling for User Data Parsing

This example parses a dictionary representing a user record into a typed, normalized result. The record must have three fields — user_id, email, and age — and each must be of the right type.

Three different things can go wrong independently, and each failure mode deserves its own handling. A missing field is a KeyError — the key was never in the dictionary. A bad value that cannot be parsed as an integer is a ValueError — the key was there but the value is wrong. A None value where a string method is expected produces an AttributeError — the email field existed but was None rather than a string.

Each except clause names the exception it handles and binds it to a variable using as, so the error message can include the actual failure detail. This is the professional pattern: catch specifically, report informatively, and return a clean sentinel value so the caller does not have to deal with a partially constructed record.

The valid record produces a clean normalized result. The bad record — where user_id is "abc" which cannot be converted to int — triggers the ValueError branch.

```python
# Example 12.2
def parseUserRecord(record):                           # line 1
    try:                                               # line 2
        userId = int(record["user_id"])                # line 3
        email = record["email"].strip().lower()        # line 4
        ageStr = record["age"]                         # line 5
        age = int(ageStr)                              # line 6
        return {"id": userId, "email": email, "age": age}  # line 7
    except KeyError as missingField:                   # line 8
        print(f"Missing required field: {missingField}")  # line 9
        return None                                    # line 10
    except ValueError as parseError:                   # line 11
        print(f"Type conversion failed: {parseError}")# line 12
        return None                                    # line 13
    except AttributeError as attrError:                # line 14
        print(f"Unexpected None value: {attrError}")   # line 15
        return None                                    # line 16

validRecord = {"user_id": "42", "email": "Alice@Example.COM", "age": "29"}  # line 17
badRecord = {"user_id": "abc", "email": "bob@example.com", "age": "30"}     # line 18
print(parseUserRecord(validRecord))                    # line 19
print(parseUserRecord(badRecord))                      # line 20
```

---

## CONCEPT 3.1 — The else and finally Clauses

The try statement has two optional clauses that most developers underuse. The else clause runs only when the try block completed without raising any exception. This is the right place for success-only logic — code that should run when the operation succeeded but does not itself need to be inside the protection of the try block. The common mistake is to put success logic inside the try block. That works, but it means an exception raised by the success logic is caught by your except clauses, which were written to handle failure in the original operation, not failure in the success response. The else clause isolates success logic cleanly. The finally clause runs unconditionally — whether the try succeeded, the try raised, an except ran, or even an except re-raised. It is the cleanup guarantee. Before context managers, finally was the only way to ensure a resource was released. In combination, try handles the attempt, except handles specific failures, else handles success, and finally handles cleanup. The industry pattern for database transactions maps perfectly: the query goes in try, rollback logic goes in except, commit goes in else because you only commit on success, and cursor close goes in finally because the cursor must always be closed.

---

## EXAMPLE 3.1 — Database Transaction with try/except/else/finally

This example shows a processTransaction function that debits an amount from a user's account using all four clauses of the try statement.

The try block begins the transaction and queries the balance. If the balance is insufficient, it raises a ValueError explicitly — this is intentional use of exceptions as control flow, not an error condition Python imposed.

The except ValueError clause handles the known business failure: not enough funds. It rolls back and returns False. The except Exception clause catches anything unexpected — a network error, a syntax error in the query — rolls back, prints the error, and crucially re-raises with a bare raise statement. Re-raising preserves the original exception and traceback; it does not swallow the unexpected failure.

The else clause runs only when no exception was raised — meaning the balance check passed and the update executed — and commits the transaction. The finally clause closes the cursor whether anything succeeded or failed. This is the complete, correct pattern.

```python
# Example 12.3
def processTransaction(db, userId, amount):            # line 1
    try:                                               # line 2
        db.begin()                                     # line 3
        balance = db.query(f"SELECT balance FROM accounts WHERE id={userId}")  # line 4
        if balance < amount:                           # line 5
            raise ValueError(f"Insufficient funds: {balance} < {amount}")  # line 6
        db.execute(f"UPDATE accounts SET balance=balance-{amount} WHERE id={userId}")  # line 7
    except ValueError as fundError:                    # line 8
        db.rollback()                                  # line 9
        print(f"Transaction failed: {fundError}")      # line 10
        return False                                   # line 11
    except Exception as unexpectedError:               # line 12
        db.rollback()                                  # line 13
        print(f"Unexpected error, rolling back: {unexpectedError}")  # line 14
        raise                                          # line 15
    else:                                              # line 16
        db.commit()                                    # line 17
        print(f"Transaction of {amount} committed for user {userId}")  # line 18
        return True                                    # line 19
    finally:                                           # line 20
        db.close_cursor()                              # line 21
```

---

## CONCEPT 4.1 — Custom Exception Classes and Exception Hierarchies

Every mature Python library defines its own exception hierarchy. The requests library has HTTPError, ConnectionError, and Timeout. SQLAlchemy has OperationalError and IntegrityError. Django has ObjectDoesNotExist and MultipleObjectsReturned. These are not arbitrary. They give callers the ability to write precise except clauses that match the specific failure modes the library can produce, while still allowing callers to catch the common base class if they want broad handling. Defining your own exception hierarchy is the professional alternative to raising raw Exception("message"). When you raise Exception("Insufficient funds"), the caller has to match on the message string to distinguish it from every other Exception. When you raise InsufficientFundsError, the caller writes except InsufficientFundsError, which is unambiguous and stable across refactors. Custom exception classes can also carry structured data as attributes — not just a message string, but the actual values that explain the failure: how much was required, how much was available. This turns exception handling into structured error reporting rather than string parsing.

---

## EXAMPLE 4.1 — Custom Exception Hierarchy for a Payment Service

This example defines a two-level exception hierarchy for a payment service. PaymentError is the base, subclassing Exception. InsufficientFundsError and CardDeclinedError both subclass PaymentError.

InsufficientFundsError carries two attributes: required and available. Its __init__ receives the actual values, stores them, then calls super().__init__ with a formatted message. The caller can access fundError.required and fundError.available directly — no string parsing required.

CardDeclinedError carries a reason attribute, initialized similarly.

The chargeCustomer function raises these specific types based on business rules. The calling code has two except clauses: InsufficientFundsError first, because it is more specific, and PaymentError second as a fallback for CardDeclinedError and any future payment subclasses. This ordering matters — Python checks except clauses top to bottom and stops at the first match, so the more specific type must come first.

```python
# Example 12.4
class PaymentError(Exception):                         # line 1
    pass                                               # line 2

class InsufficientFundsError(PaymentError):            # line 3
    def __init__(self, required, available):           # line 4
        self.required = required                       # line 5
        self.available = available                     # line 6
        super().__init__(                              # line 7
            f"Need {required}, only {available} available"  # line 8
        )

class CardDeclinedError(PaymentError):                 # line 9
    def __init__(self, reason):                        # line 10
        self.reason = reason                           # line 11
        super().__init__(f"Card declined: {reason}")   # line 12

def chargeCustomer(customerId, amount, balance):       # line 13
    if balance < amount:                               # line 14
        raise InsufficientFundsError(amount, balance)  # line 15
    if amount > 10000:                                 # line 16
        raise CardDeclinedError("amount exceeds limit")  # line 17
    return f"Charged {amount} to customer {customerId}"  # line 18

try:                                                   # line 19
    result = chargeCustomer("cust_001", 500, 200)      # line 20
except InsufficientFundsError as fundError:            # line 21
    print(f"Fund error: need {fundError.required}, have {fundError.available}")  # line 22
except PaymentError as payError:                       # line 23
    print(f"Payment error: {payError}")                # line 24
```

---

## CONCEPT 5.1 — Final Takeaway

EAFP is not just a style preference — it is a design philosophy that the Python language was built around. The try block describes what you want to happen. The except clauses describe what you know how to handle. The else clause acts on success. The finally clause guarantees cleanup. This is a complete, layered contract between the attempt and its outcome. Custom exception hierarchies are the vocabulary you use to communicate failure modes precisely across module boundaries. When a caller catches InsufficientFundsError, they are reading your API documentation in executable form. The alternative — checking every precondition, returning None or -1 for errors, passing error codes through return values — produces code that buries intent under defensive checks and makes it impossible to distinguish "the operation succeeded and returned None" from "the operation failed." Python chose the exception model deliberately. In the next lecture, we turn to descriptors — the mechanism behind programmable attribute access — which is another place Python exposes its internals for intentional use.

---
