In the previous lecture, we looked at context managers — the mechanism that makes cleanup structural. When you write a with statement, Python guarantees that the exit method runs, no matter what happens inside the block. There is no execution path that skips it. The reason Python could design that guarantee is that exceptions are a first-class part of the language, not an afterthought. Exceptions are not just error signals. They are a control flow mechanism. And the design philosophy built on top of that mechanism has a name: EAFP — Easier to Ask Forgiveness than Permission. Today we examine that philosophy directly.

Let's start with the two competing philosophies every developer has to choose between when they write code that can fail.

The first is LBYL — Look Before You Leap. This is the C tradition. Before you do anything risky, check whether it is safe. Before you open a file, check that it exists. Before you access a dictionary key, check that the key is in there. Before you call a method on an object, check that the object is not None. This sounds sensible. It is defensive, explicit, and cautious. The problem is that it has a fundamental flaw called TOCTOU — Time Of Check, Time Of Use. The file might exist when you check it, and disappear before you open it. Another thread might delete the dictionary key between your check and your access. In single-threaded scripts on a quiet machine, TOCTOU is rare. In production systems, in multi-threaded applications, in containerized environments where files appear and disappear — TOCTOU is a real bug class. The check gives you false confidence.

The second philosophy is EAFP — Easier to Ask Forgiveness than Permission. Python's official style. Try the operation. If it fails, catch the exception and handle it. There is no check to go stale because there is no check. You either open the file or you get FileNotFoundError, and those two outcomes are separated by no time gap at all. The try block describes what you want. The except clause handles what went wrong. Your code reads like a description of intent, not a defensive gauntlet of preconditions.

Now let's look at these two approaches side by side in code.

The LBYL version of reading a file calls os.path.exists before attempting to open. It looks safe. But between the exists check on line two and the open call on line three, another process could delete the file. The if block passes, Python enters it, the open fails, and the exception is unhandled. The EAFP version on lines six through eleven skips the check entirely. It opens the file directly inside a try block. If the file is not there, FileNotFoundError is raised and caught cleanly on line ten, returning None. Notice how the EAFP version is also shorter. It reads: try to open the file; if not found, return None. That is exactly the intent, with no noise around it.

This is why Python's own documentation, Django, SQLAlchemy, and virtually every major Python library recommend EAFP. The code is cleaner and the behavior is more correct.

Now let's talk about how exceptions work structurally.

When Python raises an exception, execution jumps out of the current block, out of any nested blocks, up the call stack, until it finds a matching except clause. This is non-local control flow. It crosses function boundaries. A FileNotFoundError raised in open() inside a helper function can be caught five levels up in a request handler, if nothing in between catches it. This is powerful, but it requires discipline. You need to catch the right exception at the right level.

Python's exception types form a class hierarchy. At the root is BaseException, which includes system-level things like KeyboardInterrupt and SystemExit. Below it is Exception, the base for all ordinary application errors. From Exception descend all the types you use daily: ValueError, TypeError, KeyError, AttributeError, OSError. And under OSError sit more specific types: FileNotFoundError, PermissionError, IsADirectoryError. This hierarchy is not decoration. An except clause catches the named type and all its subclasses. If you catch OSError, you also catch FileNotFoundError. That can be what you want — or it can be a mistake that silently swallows errors you should be seeing.

The rule is: always catch the most specific exception you can actually handle.

Let's look at a real example where multiple things can go wrong independently. We have a function that parses a user record dictionary into a typed, normalized result.

Three different failure modes exist here, and they mean different things. Line three attempts to convert user_id from a string to an integer. That conversion can fail with ValueError if the string is not a valid integer, or it can fail with KeyError if the user_id key is missing entirely. Line four calls .strip() and .lower() on the email value, which raises AttributeError if the email field was None rather than a string. Lines five and six do the same integer conversion for age.

Each failure mode gets its own except clause. KeyError on line eight catches missing fields. ValueError on line eleven catches bad string-to-integer conversions. AttributeError on line fourteen catches None where a string was expected. Each clause uses as to bind the exception to a name, so the print statement can include the actual failure detail. Common mistake here is to put a broad except Exception at the end that catches everything. That collapses three distinct failure modes into one branch, losing the specific information about what actually went wrong.

The valid record on line seventeen produces a clean normalized dictionary. The bad record on line eighteen has user_id as "abc", which cannot be parsed as an integer, so the ValueError branch fires.

Now let's look at two underused clauses of the try statement: else and finally.

Most developers know try and except. Fewer use else and finally correctly.

The else clause runs only when the try block completed without raising any exception. This is the right place for success-only logic. Consider what happens if you put your success logic inside the try block itself: if that success logic raises an exception, your except clauses will catch it. Those except clauses were written to handle failures in the original risky operation, not failures in the success response. The else clause separates them cleanly. If the try block raises, else never runs. If the try block succeeds, else runs. The except clauses never see errors from the else block.

The finally clause runs unconditionally — whether try succeeded, try raised, an except ran, or an except re-raised. It is the unconditional cleanup guarantee. Before context managers, finally was the only way to ensure a resource was released. In practice, finally handles what context managers cannot: cleanup that spans a structure too complex to wrap in a single with block, or cleanup that must happen after a sequence of conditional operations.

In our database transaction example, look at how all four clauses work together. The try block begins the transaction and executes the business logic. If the balance is insufficient, line six raises a ValueError deliberately — this is intentional use of exceptions as control flow, not Python complaining about something. The except ValueError clause on line eight catches that business failure, rolls back, and returns False. The except Exception clause on line twelve catches anything unexpected — network errors, query errors, anything — rolls back, prints the error, and then re-raises on line fifteen. Notice that bare raise, with no argument. That re-raises the original exception with its original traceback preserved. It does not swallow the unexpected failure. The else clause on line sixteen only runs when nothing raised, meaning the balance check passed and the update executed. It commits the transaction. The finally clause on line twenty closes the cursor no matter what path was taken.

This four-clause pattern is the database transaction pattern used in Django, SQLAlchemy, and every serious Python data layer.

Now let's talk about custom exceptions.

Every mature Python library defines its own exception hierarchy. This is not vanity. It is API design. When requests raises HTTPError, the caller writes except requests.HTTPError. When SQLAlchemy raises IntegrityError, the caller writes except sqlalchemy.exc.IntegrityError. These exception types are part of the library's contract with its callers. They tell you exactly what can go wrong and give you a precise target to catch.

The alternative — raising raw Exception("message") — forces the caller to match on a string. That is fragile, invisible to static analysis tools, and breaks silently when the message changes.

Custom exceptions also carry structured data. Look at InsufficientFundsError in our payment example. It stores required and available as attributes. The caller accesses fundError.required and fundError.available directly, getting the actual numeric values, not a string they have to parse. The hierarchy has PaymentError as the base, with InsufficientFundsError and CardDeclinedError as subclasses. The caller can catch InsufficientFundsError specifically if they want to handle insufficient funds differently from card declines, or catch PaymentError broadly if any payment failure gets the same response.

Notice the order of the except clauses on lines twenty-one and twenty-three. InsufficientFundsError comes first, PaymentError second. Python checks except clauses top to bottom and stops at the first match. Since InsufficientFundsError is a subclass of PaymentError, putting PaymentError first would catch InsufficientFundsError before the specific branch got a chance. Most specific first — always.

Let me pull everything together.

EAFP is Python's idiomatic philosophy. Attempt the operation; handle failure if it occurs. No stale precondition checks, no TOCTOU race windows. The try block describes what you want. The except clauses describe what you know how to handle. The else clause acts on success. The finally clause guarantees cleanup. These four clauses together form a complete, layered contract between an operation and its outcome.

Catch specific exceptions. Never use a bare except. Never catch Exception blindly unless you immediately re-raise. When you catch broadly and swallow, you are hiding bugs that will surface as mysterious behavior later.

Define custom exception hierarchies for any library or service boundary where failure modes have distinct meanings. Treat your exception types as part of your public API. They are documentation that runs.

This is why Python code reads naturally. You describe what you want to happen. The exception model handles the rest. In the next lecture, we move to descriptors — Python's mechanism for programmable attribute access — which is another case where Python deliberately exposes its internals so you can customize behavior that other languages keep locked away.
