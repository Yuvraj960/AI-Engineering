# Module 07: Code Reusability - Functions and Modules

Welcome to the **Code Reusability - Functions and Modules** module of the AI Engineering track. In Python, functions encapsulate specific operations into reusable blocks, while modules organize related functions and constants into separate files (`.py`). Mastering functions and modular design enables you to write clean, maintainable, and scalable software.

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Describe** the structure and benefits of Python functions and modules for code organization and maintenance.

2. **Explain** key functional concepts, including parameters, positional vs. keyword arguments, default arguments, return values, and variable scoping rules (LEGB).

3. **Apply** function definitions using the `def` keyword, type hints, docstrings, and effective `return` statements.

4. **Analyze & Utilize** standard built-in modules (`math`, `os`, `random`, `sys`) using `import`, `from ... import`, and aliasing `as`.

5. **Design & Construct** custom multi-file Python projects using custom `.py` modules and the `if __name__ == "__main__":` idiom.

## 🧠 Core Topics & In-Depth Notes

### 1. Function Structure, Arguments & Parameters

Functions are declared using the `def` keyword, followed by the function name, parentheses `()`, and a colon `:`.

```
def calculate_accuracy(correct_predictions: int, total_samples: int) -> float:
    """Calculates model accuracy given correct count and total count."""
    if total_samples <= 0:
        return 0.0
    return correct_predictions / total_samples

```

#### Argument Passing Mechanics

* **Positional Arguments:** Passed in the order defined by the parameters.
* **Keyword Arguments:** Passed explicitly by naming parameters, ignoring order.
* **Default Arguments:** Assigned initial values in the parameter list; must follow positional parameters.

```
def train_model(model_name, epochs=10, learning_rate=0.001):
    print(f"Training {model_name} | Epochs: {epochs} | LR: {learning_rate}")

# Positional and Keyword usages
train_model("ResNet")                         # Uses default epochs (10) & LR (0.001)
train_model("ResNet", 25)                     # Overrides epochs (25)
train_model(learning_rate=0.01, model_name="BERT") # Keyword args (order ignored)

```

* **Mutable Default Argument Trap:** Never use mutable types (like lists or dicts) as default arguments because they are evaluated once at function definition, not invocation.

```
# BAD: Mutable default parameter retains state across calls
def append_to_list(val, target_list=[]):
    target_list.append(val)
    return target_list

# GOOD: Use None as default indicator
def append_to_list_safe(val, target_list=None):
    if target_list is None:
        target_list = []
    target_list.append(val)
    return target_list

```

### 2. Variable Scope and Lifetime (LEGB Rule)

Python resolves variable names using the **LEGB** lookup order:

1. **Local (L):** Defined inside the active function body.
2. **Enclosing (E):** Defined in enclosing/outer functions (nested scopes).
3. **Global (G):** Defined at the top level of the module file.
4. **Built-in (B):** Preloaded Python identifiers (e.g., `len`, `range`, `ValueError`).

```
global_var = "I am Global"

def scope_demo():
    local_var = "I am Local"
    print(global_var)  # Accesses Global
    print(local_var)   # Accesses Local

scope_demo()
# print(local_var)    # Raises NameError! local_var is destroyed after execution.

```

* **The `global` Keyword:** Used inside functions to modify global variables. Modifying global variables directly from functions is generally discouraged in clean code architecture.

### 3. Working with Built-in Python Modules

Modules are imported using `import` statements. Python locates modules via `sys.path`.

```
import math
import random as rnd
from os import path, getcwd

# Using math module
radius = 5.0
area = math.pi * (radius ** 2)

# Using random alias
sample_score = rnd.uniform(0.5, 1.0)

# Using selective imports
current_directory = getcwd()

```

### 4. Creating & Importing Custom Modules

A custom module is simply a standard Python file containing functions, classes, or constants.

#### Module Design Best Practice: The Entry Guard
When a `.py` file is imported, Python executes all top-level code. To prevent test scripts or execution blocks from running during an import, use `if __name__ == "__main__":`.

```
# file: metrics.py
def mean_squared_error(y_true, y_pred):
    return sum((t - p) ** 2 for t, p in zip(y_true, y_pred)) / len(y_true)

# Execution Guard
if __name__ == "__main__":
    print("Testing MSE function locally...")
    test_actual = [1.0, 2.0, 3.0]
    test_predicted = [1.1, 1.9, 3.2]
    print(f"Test MSE: {mean_squared_error(test_actual, test_predicted):.4f}")

```

## 🛠️ Practical Hands-On Project: Multi-File Data Processing Pipeline

This project demonstrates creating a modular utility engine for data generation, statistical processing, and pipeline reporting.

### Project Directory Structure
```
project/
│
├── stats_engine.py      # Custom Module with mathematical utilities
└── pipeline_runner.py   # Main application script importing stats_engine
```

### Module File: `stats_engine.py`

```
# filename: stats_engine.py
"""
Utility module for modular statistical computing.
"""
import math

def calculate_mean(data: list[float]) -> float:
    """Calculates arithmetic mean of a dataset."""
    if not data:
        raise ValueError("Data list cannot be empty.")
    return sum(data) / len(data)

def calculate_std_dev(data: list[float]) -> float:
    """Calculates population standard deviation of a dataset."""
    if len(data) < 2:
        return 0.0
    mean = calculate_mean(data)
    variance = sum((x - mean) ** 2 for x in data) / len(data)
    return math.sqrt(variance)

def normalize_dataset(data: list[float]) -> list[float]:
    """Applies Z-score normalization: (x - mean) / std_dev."""
    mean = calculate_mean(data)
    std_dev = calculate_std_dev(data)
    if std_dev == 0:
        return [0.0 for _ in data]
    return [(x - mean) / std_dev for x in data]

if __name__ == "__main__":
    # Local unit testing block
    test_data = [10.0, 12.0, 23.0, 23.0, 16.0, 23.0, 21.0, 16.0]
    print(f"[TEST] Mean: {calculate_mean(test_data):.2f}")
    print(f"[TEST] Std Dev: {calculate_std_dev(test_data):.2f}")

```

### Runner File: `pipeline_runner.py`

```
# filename: pipeline_runner.py
"""
Main entry point executing data processing workflow using stats_engine module.
"""
import random
import stats_engine as stats

def generate_sensor_readings(num_samples: int = 20) -> list[float]:
    """Generates synthetic temperature data samples."""
    random.seed(42)  # For reproducible results
    return [round(random.uniform(20.0, 35.0), 2) for _ in range(num_samples)]

def run_data_pipeline():
    print("=== Pipeline Step 1: Ingesting Sensor Data ===")
    raw_data = generate_sensor_readings(10)
    print(f"Raw Readings: {raw_data}")

    print("\n=== Pipeline Step 2: Computing Metrics via stats_engine ===")
    mean_val = stats.calculate_mean(raw_data)
    std_val = stats.calculate_std_dev(raw_data)
    print(f"Dataset Mean: {mean_val:.2f}°C")
    print(f"Dataset Standard Deviation: {std_val:.2f}°C")

    print("\n=== Pipeline Step 3: Normalizing Data ===")
    normalized_data = stats.normalize_dataset(raw_data)
    print("Z-Score Normalized Data:")
    for raw, norm in zip(raw_data, normalized_data):
        print(f"  Raw: {raw:5.2f}°C  ==> Normalized: {norm:6.3f}")

if __name__ == "__main__":
    run_data_pipeline()

```

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **What is the purpose of `if __name__ == "__main__":` in Python scripts?**

   * *Answer:* It checks whether the Python file is being executed directly as the main script or imported as a module into another file. When executed directly, `__name__` is set to `"__main__"`, running the guarded block. When imported, `__name__` takes the module's file name, suppressing execution of main blocks or tests.

2. **Why should you avoid using mutable default arguments in functions?**

   * *Answer:* Mutable default parameters (like `def func(lst=[])`) are evaluated once when the function is defined, not every time it is called. Consequently, mutations persist across successive calls to the function, causing unintended side effects.

3. **How does Python resolve variable lookup across scopes?**

   * *Answer:* Python follows the **LEGB rule**: Local scope first, followed by Enclosing (nested) scopes, Global scope (module level), and finally Built-in scope functions/constants.

## 📚 Curated Resources & References

### Official Documentation & Guides

* [Python Official Documentation – Defining Functions](https://docs.python.org/3/tutorial/controlflow.html#defining-functions) – Detailed syntax guide for functions and parameter options.

* [Python Official Documentation – Modules](https://docs.python.org/3/tutorial/modules.html) – Official reference for Python module importing and namespaces.

### Interactive Practice Platforms

* [Real Python: Defining Your Own Python Function](https://realpython.com/defining-your-own-python-function/) – Comprehensive guide covering functions, parameter passing, and return types.

* [GeeksforGeeks: Python Functions and Modules](https://www.geeksforgeeks.org/python-functions/) – Concise reference with code examples and exercise challenges.