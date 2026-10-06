# Module 04: Dynamic Sequences - Lists

Welcome to the **Dynamic Sequences - Lists** module of the AI Engineering track. Python lists are ordered, mutable, and dynamic collections that store heterogeneous data types. This module covers essential list operations, memory mutability, list comprehensions, nested list processing, and sorting algorithms in Python.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:
1. **Describe** the characteristics, structural advantages, and mutability model of Python lists.
2. **Explain** the behavioral and structural differences between lists and other collections (`tuple`, `set`, `dict`).
3. **Apply** fundamental list operations (`append`, `extend`, `insert`, `pop`, `remove`, slicing, indexing) to manage dynamic collections.
4. **Evaluate & Implement** list comprehensions for concise, readable, and pythonic data filtering and transformations.
5. **Manipulate** multidimensional data structures (nested lists) and apply advanced sorting techniques using custom key functions.

---

## 🧠 Core Topics & In-Depth Notes

### 1. Structure & Mutability Model

Lists in Python are **dynamic mutable arrays**. Unlike fixed-length arrays in lower-level languages, Python lists automatically handle allocation and expansion as elements are added or removed.

* **Heterogeneous Elements:** A single list can hold elements of mixed data types (e.g., `[1, "AI", 3.14, True]`).
* **In-Place Mutation:** Modifying an element inside a list updates the list reference directly without creating a new object in memory.

```python
# Demonstrating Mutability
numbers = [10, 20, 30]
print(f"Original ID: {id(numbers)}")

numbers[1] = 99  # In-place modification
print(f"Updated List: {numbers}")
print(f"Same ID: {id(numbers)}")  # ID remains identical
```

---

### 2. Essential List Operations & Methods

| Method / Syntax | Operations & Description | Time Complexity |
| :--- | :--- | :--- |
| `append(x)` | Adds element `x` to the end of the list. | $O(1)$ amortized |
| `extend(iterable)` | Concatenates all elements of an iterable to the list. | $O(k)$ where $k$ is length of iterable |
| `insert(i, x)` | Inserts element `x` at index `i`. Shifts existing elements right. | $O(n)$ |
| `remove(x)` | Removes the *first* matching element equal to `x`. Raises `ValueError` if missing. | $O(n)$ |
| `pop([i])` | Removes and returns item at index `i` (defaults to last item if omitted). | $O(1)$ for last item, $O(n)$ for middle |
| `sort()` vs `sorted()` | `sort()` mutates list in-place ($O(n \log n)$). `sorted()` returns a new list. | $O(n \log n)$ |

```python
# Comparison of append() vs extend()
a = [1, 2]
a.append([3, 4])  # [1, 2, [3, 4]] -> Nested insertion

b = [1, 2]
b.extend([3, 4])  # [1, 2, 3, 4]   -> Flattened extension
```

---

### 3. List Comprehensions

List comprehensions provide a compact, expressive, and optimized syntax for creating lists from existing sequences.

#### Syntax Formula
`[expression for item in iterable if condition]`

```python
# Traditional Loop Approach
squares_even = []
for x in range(10):
    if x % 2 == 0:
        squares_even.append(x ** 2)

# Pythonic List Comprehension Equivalent
squares_even_comp = [x ** 2 for x in range(10) if x % 2 == 0]

print(squares_even_comp)  # [0, 4, 16, 36, 64]
```

---

### 4. Multidimensional Data & Nested Lists

A multidimensional list is a list containing other lists. Commonly used for matrices, grids, or tabular records before moving to frameworks like NumPy or Pandas.

```python
# 3x3 Matrix representation
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# Accessing row 2, column 3 (value 6)
print(matrix[1][2])

# Flattening a 2D matrix using double list comprehension
flattened = [val for row in matrix for val in row]
print(flattened)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

### 5. Custom Sorting with `key` Functions

The `sort()` method and `sorted()` function accept a `key` parameter (a callable function) to customize sorting logic (e.g., sorting objects by a specific dictionary field or string property).

```python
dataset = [
    {"name": "Model_A", "latency_ms": 120, "accuracy": 0.92},
    {"name": "Model_B", "latency_ms": 45,  "accuracy": 0.88},
    {"name": "Model_C", "latency_ms": 80,  "accuracy": 0.95}
]

# Sort models by lowest latency ascending
dataset.sort(key=lambda model: model["latency_ms"])
print("Sorted by Latency:", dataset)

# Sort models by highest accuracy descending
dataset_by_acc = sorted(dataset, key=lambda m: m["accuracy"], reverse=True)
print("Sorted by Accuracy:", dataset_by_acc)
```

---

## 🛠️ Practical Hands-On Project: Student Score Management & Filtering Engine

This project demonstrates processing, filtering, and reporting performance metrics across student grade records using dynamic lists and list comprehensions.

```python
# filename: score_management_engine.py

def analyze_student_records():
    # Raw incoming data: [Student ID, Name, Score]
    raw_records = [
        [101, "Alice", 88.5],
        [102, "Bob", 42.0],
        [103, "Charlie", 95.0],
        [104, "Diana", 67.5],
        [105, "Ethan", 53.0],
        [106, "Fiona", 79.0]
    ]

    print("=== Raw Records ===")
    for rec in raw_records:
        print(f"ID: {rec[0]} | Name: {rec[1]} | Score: {rec[2]}")

    # 1. Compute Average Score
    scores = [rec[2] for rec in raw_records]
    avg_score = sum(scores) / len(scores)
    print(f"\nClass Average Score: {avg_score:.2f}")

    # 2. Extract High Achievers (Score >= 75) sorted by score descending
    high_achievers = [rec for rec in raw_records if rec[2] >= 75.0]
    high_achievers.sort(key=lambda item: item[2], reverse=True)

    print("\n=== High Achievers (Score >= 75) ===")
    for rank, rec in enumerate(high_achievers, start=1):
        print(f"Rank {rank}: {rec[1]} - {rec[2]}")

    # 3. Categorize Students (Passing >= 50 vs Needs Improvement < 50)
    passing_names = [rec[1] for rec in raw_records if rec[2] >= 50.0]
    failing_names = [rec[1] for rec in raw_records if rec[2] < 50.0]

    print("\n=== Summary Categorization ===")
    print(f"Passing Students ({len(passing_names)}): {', '.join(passing_names)}")
    print(f"Needs Help ({len(failing_names)}): {', '.join(failing_names)}")

if __name__ == "__main__":
    analyze_student_records()
```

---

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **What is the difference between `list.pop()` and `list.remove()`?**
   * *Answer:* `list.pop([index])` removes and returns an element at a given position (defaulting to the last index). `list.remove(value)` searches for the first element matching `value` and removes it without returning the value.

2. **How do `sort()` and `sorted()` differ?**
   * *Answer:* `list.sort()` is an in-place method that modifies the original list and returns `None`. `sorted(iterable)` is a built-in function that leaves the original sequence untouched and returns a new sorted list.

3. **What is a potential risk of nested list initialization like `[[0]*3]*3`?**
   * *Answer:* This syntax creates three outer elements referencing the *same underlying list object in memory*. Modifying one row (e.g., `matrix[0][0] = 1`) alters every row (`[[1, 0, 0], [1, 0, 0], [1, 0, 0]]`). Use `[[0 for _ in range(3)] for _ in range(3)]` instead.

---

## 📚 Curated Resources & References

### Official Documentation & Tutorials
* [Python Official Documentation - More on Lists](https://docs.python.org/3/tutorial/datastructures.html#more-on-lists) – Core usage details and data structure behaviors.
* [Real Python: Python Lists & Tuples](https://realpython.com/python-lists-tuples/) – Comparative dynamic list features.

### Interactive Learning Tools
* [W3Schools Python Lists](https://www.w3schools.com/python/python_lists.asp) – Exercises on operations and comprehensions.
* [GeeksforGeeks: Python Lists](https://www.geeksforgeeks.org/python-lists/) – Code examples and method summaries.