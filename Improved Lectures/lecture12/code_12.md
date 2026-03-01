# Code — Lecture 12: Exception Model & EAFP — Errors as Alternate Control Flow

---

## Example 12.1 — LBYL vs EAFP File Reading

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

**Output:**
```
None
```

**Explanation:**
- Line 1: readFileLBYL defines the LBYL approach — precondition check before action
- Line 2: os.path.exists() checks whether the file is present at this exact moment
- Line 3: the open() call happens after the check — another process could delete the file between line 2 and line 3, creating a TOCTOU race condition
- Line 4: file contents are returned if the open succeeded
- Line 5: None is returned if the file was absent at check time, but the race window means the open on line 3 can still fail with an unhandled exception
- Line 6: readFileEAFP defines the EAFP approach — no precondition check
- Line 7: try block wraps the risky operation directly
- Line 8: open() is attempted immediately — the file is either opened or it is not, with no time gap
- Line 9: if open succeeded, the file contents are read and returned
- Line 10: FileNotFoundError is caught specifically — not OSError, not Exception, exactly the failure mode this function knows how to handle
- Line 11: None is returned for a missing file, matching the LBYL behavior without the race condition
- Line 12: readFileEAFP called with a file that does not exist in this environment
- Line 13: prints None because config.json was not found and the except clause returned None

---

## Example 12.2 — Multi-Exception Handling for User Data Parsing

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

**Output:**
```
{'id': 42, 'email': 'alice@example.com', 'age': 29}
Type conversion failed: invalid literal for int() with base 10: 'abc'
None
```

**Explanation:**
- Line 1: parseUserRecord receives a raw dictionary that may be missing fields or have wrong types
- Line 2: try block wraps all operations that can fail — field access and type conversions together
- Line 3: record["user_id"] raises KeyError if the key is absent; int() raises ValueError if the value is not numeric
- Line 4: .strip() and .lower() raise AttributeError if email is None instead of a string
- Line 5: record["age"] raises KeyError if the age key is missing
- Line 6: int(ageStr) raises ValueError if age is not a numeric string
- Line 7: only reached when all conversions succeeded; returns a fully typed, normalized dictionary
- Line 8: KeyError caught with "as missingField" — the exception object contains the missing key name as its value
- Line 9: the missing field name is printed directly from the exception object
- Line 11: ValueError caught with "as parseError" — the exception message describes exactly which conversion failed
- Line 14: AttributeError caught for the None-instead-of-string case; most specific clause for that failure mode
- Line 17: validRecord has all fields as valid numeric and string representations
- Line 18: badRecord has "abc" for user_id, which cannot be parsed as int — triggers ValueError on line 3
- Line 19: valid record returns a clean normalized dict with int ids and lowercase email
- Line 20: bad record prints the ValueError message and returns None

---

## Example 12.3 — Database Transaction with try/except/else/finally

```python
# Example 12.3
class MockDB:                                          # line 1
    def __init__(self, balance):                       # line 2
        self.balance = balance                         # line 3
        self.committed = False                         # line 4
    def begin(self):                                   # line 5
        print("Transaction begun")                     # line 6
    def query(self, sql):                              # line 7
        return self.balance                            # line 8
    def execute(self, sql):                            # line 9
        print("Update executed")                       # line 10
    def rollback(self):                                # line 11
        print("Rollback performed")                    # line 12
    def commit(self):                                  # line 13
        self.committed = True                          # line 14
        print("Commit performed")                      # line 15
    def close_cursor(self):                            # line 16
        print("Cursor closed")                         # line 17

def processTransaction(db, userId, amount):            # line 18
    try:                                               # line 19
        db.begin()                                     # line 20
        balance = db.query(f"SELECT balance FROM accounts WHERE id={userId}")  # line 21
        if balance < amount:                           # line 22
            raise ValueError(f"Insufficient funds: {balance} < {amount}")  # line 23
        db.execute(f"UPDATE accounts SET balance=balance-{amount} WHERE id={userId}")  # line 24
    except ValueError as fundError:                    # line 25
        db.rollback()                                  # line 26
        print(f"Transaction failed: {fundError}")      # line 27
        return False                                   # line 28
    except Exception as unexpectedError:               # line 29
        db.rollback()                                  # line 30
        print(f"Unexpected error, rolling back: {unexpectedError}")  # line 31
        raise                                          # line 32
    else:                                              # line 33
        db.commit()                                    # line 34
        print(f"Transaction of {amount} committed for user {userId}")  # line 35
        return True                                    # line 36
    finally:                                           # line 37
        db.close_cursor()                              # line 38

richDB = MockDB(balance=1000)                          # line 39
print(processTransaction(richDB, userId=1, amount=200))  # line 40
print()
poorDB = MockDB(balance=50)                            # line 41
print(processTransaction(poorDB, userId=2, amount=200))  # line 42
```

**Output:**
```
Transaction begun
Update executed
Commit performed
Transaction of 200 committed for user 1
Cursor closed
True

Transaction begun
Rollback performed
Transaction failed: Insufficient funds: 50 < 200
Cursor closed
False
```

**Explanation:**
- Lines 1-17: MockDB simulates a database connection with begin, query, execute, rollback, commit, and close_cursor operations
- Line 18: processTransaction takes a db object, a user id, and an amount to debit
- Line 19: try block wraps all operations that can fail — begin, query, and execute
- Line 20: begin() starts the transaction; in a real database this opens a transaction boundary
- Line 22-23: balance check raises ValueError intentionally — this is explicit control flow, not Python imposing an error
- Line 25: except ValueError catches the known business failure; rollback is called, False returned
- Line 29: except Exception catches anything unexpected — a broader net, but note line 32
- Line 32: bare raise re-raises the original exception with its original traceback intact; the unexpected failure is not swallowed
- Line 33: else runs only when no exception was raised in the try block — meaning the balance check passed and the update succeeded
- Line 34: commit is placed in else, not in try, because it should only run on a clean success path
- Line 37: finally runs unconditionally — after the except path, after the else path, even after the re-raise on line 32
- Line 38: close_cursor always executes; this is the guarantee context managers provide structurally, done manually here

---

## Example 12.4 — Custom Exception Hierarchy for a Payment Service

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

print()

try:                                                   # line 25
    result = chargeCustomer("cust_002", 15000, 20000)  # line 26
except InsufficientFundsError as fundError:            # line 27
    print(f"Fund error: need {fundError.required}, have {fundError.available}")  # line 28
except PaymentError as payError:                       # line 29
    print(f"Payment error: {payError}")                # line 30
```

**Output:**
```
Fund error: need 500, have 200

Payment error: Card declined: amount exceeds limit
```

**Explanation:**
- Line 1: PaymentError subclasses Exception; it is the base for all payment-related failures
- Line 2: pass — no additional behavior needed; the class exists to define the type
- Line 3: InsufficientFundsError subclasses PaymentError, not Exception directly; this makes it catchable both specifically and broadly
- Lines 4-8: __init__ stores required and available as instance attributes, then calls super().__init__ with a formatted message so str(error) works correctly
- Line 9: CardDeclinedError also subclasses PaymentError; callers catching PaymentError catch both subclasses
- Lines 10-12: same pattern — store the structured reason attribute, call super().__init__ with the formatted message
- Line 13: chargeCustomer encodes two distinct business failure modes as specific exception types
- Line 14-15: insufficient balance raises InsufficientFundsError with the exact numeric values
- Line 16-17: amount exceeding the limit raises CardDeclinedError with a reason string
- Line 18: only reached when both checks pass; the happy path returns a confirmation string
- Line 19-20: chargeCustomer called with balance 200 and amount 500 — insufficient funds
- Line 21: InsufficientFundsError caught first; it is more specific than PaymentError — ordering matters
- Line 22: fundError.required is 500, fundError.available is 200 — accessed as attributes, not parsed from a string
- Line 23: PaymentError as the fallback catches CardDeclinedError and any future PaymentError subclasses
- Line 25-26: second call with amount 15000 exceeding the 10000 limit — CardDeclinedError raised
- Line 27: InsufficientFundsError does not match CardDeclinedError; Python moves to the next clause
- Line 29: PaymentError matches CardDeclinedError because CardDeclinedError is a subclass; payError message is "Card declined: amount exceeds limit"
