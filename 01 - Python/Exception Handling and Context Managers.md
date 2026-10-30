# Module 09: Exception Handling and Context Managers

Welcome to **Module 09: Exception Handling and Context Managers**. As programs grow in complexity, handling runtime errors and managing system resources (such as database connections, file handles, and network sockets) deterministically becomes critical. This module covers advanced error handling patterns, custom exception hierarchies, and resource management via Python’s Context Manager protocol (`with` statement).

---

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Analyze** Python's exception propagation mechanism, stack unwinding, and runtime error execution flows.
2. **Implement** robust error-handling structures utilizing `try`, `except`, `else`, and `finally` blocks.
3. **Design** domain-specific custom exception hierarchies inheriting from standard base classes.
4. **Evaluate** resource lifecycle management using Context Managers and the Python Context Management Protocol (`__enter__` and `__exit__`).
5. **Construct** custom context managers using both class-based protocols and the `@contextmanager` generator decorator.
6. **Apply** defensive programming and clean teardown mechanics in production-ready software systems.

---

## 🧠 Core Topics & Deep-Dive Concepts

### 1. Python Exception Handling Mechanics

Exceptions are signals raised by the interpreter or program logic when an error alters the normal flow of execution. When an exception is raised, Python unwinds the call stack searching for an matching exception handler.

#### The Complete Control Flow (`try-except-else-finally`)

* **`try`**: Encapsulates code that may raise an exception.
* **`except`**: Intercepts specific exceptions. Multiple `except` blocks are evaluated top-to-bottom.
* **`else`**: Executes **only if** no exception was raised in the `try` block. Used for logic that depends on successful execution of the `try` block to avoid catching unwanted exceptions.
* **`finally`**: Guarantees execution regardless of whether an exception occurred, was caught, or was uncaught. Ideal for low-level manual cleanup.

```python
import logging

def safe_divide_and_store(numerator: float, denominator: float) -> float:
    result = None
    try:
        # Code that might fail
        result = numerator / denominator
    except ZeroDivisionError as e:
        # Specific exception handling
        logging.error(f"Division failed: Cannot divide {numerator} by zero.")
        raise ValueError("Invalid divisor provided") from e
    except TypeError as e:
        logging.error(f"Type error during calculation: {e}")
        return 0.0
    else:
        # Executes ONLY if try block succeeded
        logging.info("Division calculated successfully without errors.")
        return result
    finally:
        # Executes ALWAYS (even if return or raise occurred above)
        logging.info("Execution complete for safe_divide_and_store.")
```

---

### 2. Custom Exception Hierarchies

Inheriting from Python's built-in `Exception` class allows developers to define application-specific errors. Creating a structured hierarchy simplifies error catching at different boundary layers of a software application.

> **Note:** Custom exceptions should inherit from `Exception`, not `BaseException` (which includes `SystemExit` and `KeyboardInterrupt`).

```python
class ApplicationError(Exception):
    """Base class for all application-level errors."""
    pass

class DatabaseConnectionError(ApplicationError):
    """Raised when the application fails to connect to the database."""
    def __init__(self, host: str, port: int, reason: str):
        self.host = host
        self.port = port
        self.reason = reason
        super().__init__(f"Failed to connect to {host}:{port} - Reason: {reason}")

class RecordNotFoundError(ApplicationError):
    """Raised when a query returns no records."""
    def __init__(self, record_id: str):
        self.record_id = record_id
        super().__init__(f"Record with ID '{record_id}' was not found.")

# Usage Example:
try:
    raise DatabaseConnectionError("127.0.0.1", 5432, "Connection timed out")
except ApplicationError as err:
    # Catches DatabaseConnectionError, RecordNotFoundError, and any ApplicationError subclass
    print(f"Domain Error Captured: {err}")
```

---

### 3. Context Managers & The `with` Statement

A **Context Manager** manages resources by allocating them when entering a context and automatically releasing or cleaning them up upon exit, even if exceptions occur within the context block.

#### Context Management Protocol

An object implements the Context Management Protocol by defining two dunder methods:

1. `__enter__(self)`:
   * Called before entering the `with` block.
   * Returns the resource or object bound to the target variable in `as <variable>`.
2. `__exit__(self, exc_type, exc_val, exc_tb)`:
   * Called when exiting the `with` block.
   * Receives three arguments if an exception occurred (`exc_type`, `exc_val`, `exc_tb`); otherwise, all three are `None`.
   * Returning `True` suppresses the exception. Returning `False` (or `None`) allows the exception to propagate up the call stack.

```python
class ManagedFile:
    """Class-based custom context manager for safe file I/O."""
    def __init__(self, filename: str, mode: str):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        print(f"[ACQUIRE] Opening file: {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file  # Bound to the target in 'as target'

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"[RELEASE] Closing file: {self.filename}")
        if self.file:
            self.file.close()
        
        if exc_type is not None:
            print(f"[ERROR HANDLED] Exception detected: {exc_val}")
            # Returning True suppresses the exception from propagating
            return True 
        return False

# Usage:
with ManagedFile("example.txt", "w") as f:
    f.write("Hello, Context Manager!")
    # Simulating an inline error:
    # raise ValueError("Something went wrong inside the file operation!")
```

---

### 4. Generator-Based Context Managers (`contextlib`)

Python provides the `contextlib` module, which contains the `@contextmanager` decorator. This allows creating context managers using a single generator function instead of writing a full class with `__enter__` and `__exit__`.

```python
from contextlib import contextmanager
import time

@contextmanager
def execution_timer(label: str):
    """Generator-based context manager for timing block execution."""
    start_time = time.perf_counter()
    print(f"⏱️ [{label}] Starting timer...")
    try:
        yield  # Control yields to the body of the 'with' block
    finally:
        # Clean-up code runs after the 'with' block finishes or raises an exception
        elapsed = (time.perf_counter() - start_time) * 1000
        print(f"⏱️ [{label}] Finished in {elapsed:.2f} ms")

# Usage:
with execution_timer("Heavy Computing Phase"):
    sum([i ** 2 for i in range(1000000)])
```

---

## 🛠️ Practical Hands-On Project: Transactional Database & Audit Logger System

This system implements custom exceptions, a class-based database transaction manager with auto-rollback features, and a context-managed file logging system.

```python
# filename: transactional_system.py

import time
from typing import List, Dict, Any, Optional
from contextlib import contextmanager

# ==========================================
# 1. Custom Exception Hierarchy
# ==========================================

class SystemError(Exception):
    """Base exception for system operations."""
    pass

class TransactionError(SystemError):
    """Error raised during database transactions."""
    pass

class InsufficientFundsError(TransactionError):
    """Error raised when account balance is low."""
    def __init__(self, account_id: str, balance: float, required: float):
        self.account_id = account_id
        self.balance = balance
        self.required = required
        super().__init__(
            f"Account '{account_id}' insufficient funds: Available ${balance:.2f}, Required ${required:.2f}"
        )


# ==========================================
# 2. Database Session Context Manager (Class-Based)
# ==========================================

class MockDatabaseConnection:
    """Simulates a database connection with transaction state."""
    def __init__(self):
        self.in_transaction = False
        self.database = {"ACC_01": 500.0, "ACC_02": 150.0}
        self.snapshot = {}

    def begin_transaction(self):
        self.in_transaction = True
        # Take a snapshot for potential rollback
        self.snapshot = self.database.copy()
        print("  [DB] Transaction STARTED. Snapshot saved.")

    def commit(self):
        self.in_transaction = False
        self.snapshot = {}
        print("  [DB] Transaction COMMITTED successfully.")

    def rollback(self):
        if self.in_transaction:
            self.database = self.snapshot.copy()
            self.in_transaction = False
            print("  [DB] Transaction ROLLED BACK to previous snapshot state.")


class DatabaseTransactionManager:
    """Class-based context manager handling automatic commit and rollback."""
    def __init__(self, connection: MockDatabaseConnection):
        self.db = connection

    def __enter__(self) -> MockDatabaseConnection:
        self.db.begin_transaction()
        return self.db

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            print(f"  [TX ERROR] Exception detected ({exc_type.__name__}: {exc_val})")
            self.db.rollback()
            # Do not suppress the exception; return False to allow propagation
            return False
        else:
            self.db.commit()
            return True


# ==========================================
# 3. Audit Context Manager (Generator-Based)
# ==========================================

@contextmanager
def audit_session(session_id: str):
    """Generator context manager for auditing transaction execution status."""
    print(f"\n📋 [AUDIT] Starting Audit Session: {session_id}")
    start_time = time.perf_counter()
    try:
        yield
        status = "SUCCESS"
    except Exception as e:
        status = f"FAILED ({type(e).__name__})"
        raise
    finally:
        elapsed = (time.perf_counter() - start_time) * 1000
        print(f"📋 [AUDIT] Ended Session: {session_id} | Status: {status} | Duration: {elapsed:.3f} ms")


# ==========================================
# 4. Service Logic
# ==========================================

def transfer_funds(db: MockDatabaseConnection, sender: str, receiver: str, amount: float):
    if sender not in db.database or receiver not in db.database:
        raise TransactionError("Invalid sender or receiver account ID.")
    
    if db.database[sender] < amount:
        raise InsufficientFundsError(sender, db.database[sender], amount)
    
    db.database[sender] -= amount
    db.database[receiver] += amount
    print(f"  [TRANSFER] ${amount:.2f} transferred from {sender} to {receiver}.")


# ==========================================
# 5. Pipeline Execution Test
# ==========================================

if __name__ == "__main__":
    db = MockDatabaseConnection()

    print("--- Test Scenario 1: Successful Transfer ---")
    try:
        with audit_session("SESS_1001"):
            with DatabaseTransactionManager(db) as tx_db:
                transfer_funds(tx_db, sender="ACC_01", receiver="ACC_02", amount=100.0)
    except SystemError as err:
        print(f"Handled Session Error: {err}")

    print("\nCurrent Database State:", db.database)

    print("\n--- Test Scenario 2: Failed Transfer (Insufficient Funds -> Auto Rollback) ---")
    try:
        with audit_session("SESS_1002"):
            with DatabaseTransactionManager(db) as tx_db:
                # ACC_02 only has 250.0 available
                transfer_funds(tx_db, sender="ACC_02", receiver="ACC_01", amount=1000.0)
    except SystemError as err:
        print(f"Handled Session Error: {err}")

    print("\nFinal Database State (Unchanged due to rollback):", db.database)
```

---

## ❓ Technical Interview & Conceptual Exam Questions

1. **How does Python handle exceptions inside an `__exit__` method of a context manager?**
   * **Answer:** When an exception occurs inside a `with` block, Python passes its type, value, and traceback to the `__exit__(exc_type, exc_val, exc_tb)` method. If `__exit__` returns `True`, the exception is suppressed and program execution continues after the `with` block. If it returns `False` or `None`, the exception is re-raised and continues propagating up the call stack.

2. **What is Exception Chaining (`raise ... from ...`) and why is it useful?**
   * **Answer:** Exception chaining explicitly links a new exception to an underlying root cause exception using the `from` keyword. It preserves the original stack trace attached to the `__cause__` attribute of the new exception, allowing developers to debug high-level errors while preserving lower-level diagnostic information.

3. **When should you use `contextlib.contextmanager` instead of implementing a class with `__enter__` and `__exit__`?**
   * **Answer:** `@contextmanager` is best suited for simple, straightforward setup and teardown scenarios that require minimal internal state. A class-based implementation is preferred when the context manager requires complex internal state tracking, multiple reusable methods, or fine-grained control over exception suppression.

---

## 📚 Curated Resources & References

### Official Documentation & Technical Specifications

* [Python Docs – Errors and Exceptions](https://docs.python.org/3/tutorial/errors.html)
* [Python Docs – The `with` Statement Context Managers](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement)
* [Python Docs – `contextlib` Module](https://docs.python.org/3/library/contextlib.html)

### Technical Articles & Guides

* [Real Python: Python's Context Managers and the 'with' Statement](https://realpython.com/python-with-statement/)
* [GeeksforGeeks: Context Manager in Python](https://www.geeksforgeeks.org/context-manager-in-python/)