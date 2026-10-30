# Module 08: Iterators, Generators, and Decorators

Welcome to the **Iterators, Generators, and Decorators** module of the AI Engineering track. This module explores advanced Python constructs for efficient data processing and dynamic code extension. You will learn how iterators control collection traversal, how generators enable memory-efficient lazy evaluation for large data streams, and how decorators dynamically modify function behavior without altering source code.

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Describe** the purpose, structure, and operational mechanics of Python iterators, generators, and function decorators.

2. **Explain** how custom iterators traverse sequences and how generator functions implement lazy evaluation to reduce memory consumption.

3. **Apply** Python's built-in iterator functions (`iter()`, `next()`), generator functions (`yield`), generator expressions, and decorator syntax (`@decorator_name`).

4. **Analyze** the operational differences between lazy evaluation (generators) and eagerly evaluated data structures (lists, tuples).

5. **Design & Implement** robust Python applications featuring custom iterators, stream generators, and custom decorators for logging, execution timing, and authentication.

## 🧠 Core Topics & In-Depth Notes

### 1. Python Iterators & The Iteration Protocol

An **iterable** is any Python object capable of returning its members one at a time (e.g., `list`, `tuple`, `str`). An **iterator** is the underlying stateful object that performs the actual traversal.

#### The Iterator Protocol

To construct a custom iterator class, two dunder methods must be implemented:

1. `__iter__()`: Returns the iterator object itself.

2. `__next__()`: Returns the next value in the sequence. When no elements remain, it must raise a `StopIteration` exception.

```
class Countdown:
    """Custom iterator counting down from start to 1."""
    def __init__(self, start: int):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        val = self.current
        self.current -= 1
        return val

# Traversal using manual next() calls
counter = Countdown(3)
print(next(counter))  # 3
print(next(counter))  # 2
print(next(counter))  # 1
# print(next(counter)) # Raises StopIteration!

```

### 2. Python Generators & Lazy Evaluation

A **generator** is a simplified function-based iterator created using the `yield` keyword instead of `return`. 

* **Lazy Evaluation:** When a generator function is invoked, it returns a generator object without executing the function body immediately. Values are computed on demand when `next()` is called.

* **State Preservation:** Upon encountering `yield`, Python saves the function's execution state, local variables, and instruction pointer, resuming immediately after the `yield` line on the next call.

```
def FibonacciGenerator(limit: int):
    """Generates Fibonacci numbers up to a given limit lazily."""
    a, b = 0, 1
    count = 0
    while count < limit:
        yield a
        a, b = b, a + b
        count += 1

# Consuming the generator via for-loop
for num in FibonacciGenerator(5):
    print(num, end=" ")  # Output: 0 1 1 2 3

```

#### Generator Expressions vs List Comprehensions

Generator expressions use parentheses `()` instead of square brackets `[]`. They do not store the full dataset in RAM.

```
import sys

# List Comprehension (Eager Evaluation - High Memory Allocation)
list_comp = [x * 2 for x in range(1000000)]
print(f"List Memory Size: {sys.getsizeof(list_comp)} bytes")  # ~8 MB

# Generator Expression (Lazy Evaluation - Constant Memory Footprint)
gen_exp = (x * 2 for x in range(1000000))
print(f"Gen Memory Size: {sys.getsizeof(gen_exp)} bytes")    # ~200 bytes

```

### 3. Python Decorators

A **decorator** is a higher-order function that takes another function as an argument, wraps or modifies its behavior, and returns the modified wrapper function without permanently altering the original function's source code.

#### Basic Decorator Structure

```
from functools import wraps

def my_logger(func):
    @wraps(func)  # Preserves original function name and docstrings
    def wrapper(*args, **kwargs):
        print(f"[LOG] Calling function: '{func.__name__}'")
        result = func(*args, **kwargs)
        print(f"[LOG] Execution completed for: '{func.__name__}'")
        return result
    return wrapper

@my_logger
def process_data(dataset_id: str):
    print(f"Processing dataset: {dataset_id}")

process_data("DS_9012")

```

#### Built-in Decorators

* `@staticmethod`: Defines a method inside a class that does not require access to instance (`self`) or class (`cls`) state.

* `@classmethod`: Defines a method that accepts the class (`cls`) as its first argument instead of an instance.

* `@property`: Turns a method into a read-only getter attribute.

## 🛠️ Practical Hands-On Project: Streaming Data Pipeline with Timing & Logging

This project demonstrates a data-processing system that uses custom decorators for execution timing/logging, custom iterators for sequence navigation, and generators for memory-safe stream filtering.

```
# filename: data_stream_processor.py

import time
from functools import wraps
from typing import Callable, Any, Generator

# ==========================================
# 1. Custom Decorators
# ==========================================

def performance_timer(func: Callable) -> Callable:
    """Decorator to measure and log the execution time of a function."""
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = (time.perf_counter() - start_time) * 1000
        print(f"⏱️ [PERFORMANCE] '{func.__name__}' executed in {elapsed:.3f} ms")
        return result
    return wrapper


def audit_logger(func: Callable) -> Callable:
    """Decorator to log argument details during execution."""
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        print(f"🔍 [AUDIT] Executing '{func.__name__}' with args={args}, kwargs={kwargs}")
        return func(*args, **kwargs)
    return wrapper


# ==========================================
# 2. Custom Iterator
# ==========================================

class BatchReader:
    """Custom Iterator to traverse a fixed sequence of raw sensor IDs."""
    def __init__(self, start_id: int, total_count: int):
        self.current = start_id
        self.end_id = start_id + total_count

    def __iter__(self):
        return self

    def __next__(self) -> str:
        if self.current >= self.end_id:
            raise StopIteration
        sensor_id = f"SENSOR_RAW_{self.current:04d}"
        self.current += 1
        return sensor_id


# ==========================================
# 3. Stream Generator Functions
# ==========================================

def raw_data_stream(sensor_iterator: BatchReader) -> Generator[dict, None, None]:
    """Simulates an infinite/large stream of sensor data records lazily."""
    for sensor in sensor_iterator:
        # Yields fake telemetry payloads lazily
        val = (hash(sensor) % 100) + 1.5
        yield {"sensor_id": sensor, "temperature": val, "status": "OK" if val < 80 else "CRITICAL"}


def filter_critical_events(stream: Generator[dict, None, None]) -> Generator[dict, None, None]:
    """Pipeline filter generator that extracts only critical readings."""
    for record in stream:
        if record["status"] == "CRITICAL" or record["temperature"] > 65.0:
            yield record


# ==========================================
# 4. Main Processing Execution
# ==========================================

@performance_timer
@audit_logger
def run_telemetry_pipeline(batch_size: int):
    """Executes the pipeline by pulling stream data through generator stages."""
    print("=== Step 1: Initializing Custom Iterator Batch ===")
    batch = BatchReader(start_id=101, total_count=batch_size)

    print("\n=== Step 2: Constructing Generator Processing Pipeline ===")
    stream = raw_data_stream(batch)
    filtered_stream = filter_critical_events(stream)

    print("\n=== Step 3: Consuming Generator Stream Outputs ===")
    critical_count = 0
    for alert in filtered_stream:
        critical_count += 1
        print(f"  🚨 ALERT DETECTED -> {alert['sensor_id']} | Temp: {alert['temperature']:.1f}°C")

    print(f"\nProcessing Complete. Total Critical Alerts Processed: {critical_count}")


if __name__ == "__main__":
    run_telemetry_pipeline(batch_size=15)

```

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **How does `yield` differ from `return` in a Python function?**

   * *Answer:* `return` terminates execution, destroys local variable state, and returns a single value to the caller. `yield` pauses function execution, saves state, returns a value, and allows the function to resume execution right after the `yield` statement when `next()` is invoked again.

2. **What is the difference between an iterable and an iterator?**

   * *Answer:* An **iterable** is an object that implements `__iter__()` to return an iterator (e.g., lists, strings). An **iterator** is an object that implements both `__iter__()` and `__next__()` to manage traversal state and raise `StopIteration` when complete.

3. **Why are decorators wrapped with `@functools.wraps`?**

   * *Answer:* When a function is decorated, its reference is replaced by the inner wrapper function, losing its original `__name__` and `__doc__` attributes. `@functools.wraps(func)` copies these metadata attributes from the original function to the wrapper.

## 📚 Curated Resources & References

### Official Documentation & Guides

* [Python Official Documentation – Iterators and Generators](https://docs.python.org/3/tutorial/classes.html#iterators) – Language specifications for iterator classes and generator expressions.

* [Python Official Documentation – Decorators (`functools.wraps`)](https://docs.python.org/3/library/functools.html#functools.wraps) – Official guide for creating robust wrapper decorators.

### Interactive Practice Platforms

* [Real Python: Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/) – Complete guide to decorators, syntax, argument passing, and stateful wrappers.

* [GeeksforGeeks: Python Generators](https://www.geeksforgeeks.org/generators-in-python/) – Practical examples comparing list comprehensions and generator functions.