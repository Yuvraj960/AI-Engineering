# Module 03: Immutable Sequences - Tuples

Welcome to the **Immutable Sequences - Tuples** module of the AI Engineering track. Tuples are fixed-length, ordered, and immutable collection types in Python. This guide covers tuple characteristics, comparison with lists, packing and unpacking mechanics, function returning strategies, and real-world scenarios like storing coordinates and configuration settings.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:
1. **Describe** the key characteristics of Python tuples, including immutability, ordering, indexing, and syntax variations (e.g., single-element tuples).
2. **Explain** how tuples differ from lists in terms of memory usage, performance, hashability, and intentional design semantics.
3. **Apply** tuple operations such as indexing, slicing, concatenation, repetition, and positional/starred unpacking.
4. **Analyze & Utilize** tuples in function signatures for packing arbitrary positional arguments (`*args`) and returning multiple return values.
5. **Create** robust Python scripts using tuples for fixed, read-only data assets such as system configurations, dictionary keys, and geospatial coordinates.

---

## 🧠 Core Topics & Deep Dive Notes

### 1. Definition, Syntax & Immutability

A **tuple** is an ordered, immutable collection of elements enclosed by parentheses `()` (or created by comma separation).

* **Syntax Note for Single-Element Tuples:** To define a single-element tuple, a trailing comma is required. Without it, Python evaluates the expression as a parenthesized value.
  ```python
  not_a_tuple = ("config")   # Type: str
  is_a_tuple  = ("config",)  # Type: tuple
  ```

* **Immutability Behavior:** You cannot append, remove, or reassign elements inside a tuple after creation.
  ```python
  point = (10, 20)
  # point[0] = 15  # Raises TypeError: 'tuple' object does not support item assignment
  ```

* **Mutability inside Immutability (Gotcha):** If a tuple contains mutable objects (such as lists), those nested mutable objects *can* still be modified in place!
  ```python
  nested = (1, 2, [3, 4])
  nested[2].append(5)
  print(nested)  # (1, 2, [3, 4, 5])
  ```

---

### 2. Tuples vs Lists: Structural & Performance Comparison

| Feature | Tuple (`tuple`) | List (`list`) |
| :--- | :--- | :--- |
| **Mutability** | Immutable | Mutable |
| **Syntax** | Parentheses `()` or comma-separated | Square brackets `[]` |
| **Memory Allocation** | Fixed size; memory optimization (smaller footprint) | Over-allocates memory to support dynamic append/extend |
| **Hashability** | Hashable (if all elements are hashable) $\rightarrow$ Can be Dict key or Set element | Unhashable $\rightarrow$ Cannot be used as Dict key |
| **Primary Use Case** | Heterogeneous data structures, fixed schemas, records | Homogeneous sequences, dynamic lists of items |

---

### 3. Tuple Operations & Built-In Methods

Because tuples are immutable, they only support two built-in lookup methods:

* `count(value)`: Returns the number of times `value` appears in the tuple.
* `index(value)`: Returns the first 0-based index of `value` (raises `ValueError` if missing).

```python
data = (10, 20, 30, 20, 40)

print(data.count(20))  # 2
print(data.index(30))  # 2

# Indirect modification via list conversion
temp_list = list(data)
temp_list.append(50)
data = tuple(temp_list)
print(data)  # (10, 20, 30, 20, 40, 50)
```

---

### 4. Tuple Unpacking & Starred Expressions

Tuple unpacking allows assigning sequence elements directly to variables in a single expression.

```python
# Basic Unpacking
coordinates = (37.7749, -122.4194)
lat, lon = coordinates

# Extended Unpacking with Starred Expressions (*)
record = ("Model_V1", "Accuracy", 0.94, 0.92, 0.96, "PASSED")
model_name, metric, *scores, status = record

print(model_name) # "Model_V1"
print(scores)     # [0.94, 0.92, 0.96] (extracted as a list)
print(status)     # "PASSED"
```

---

### 5. Tuples in Functions & Dictionary Keys

#### Multiple Return Values
Functions returning multiple values implicitly pack them into a tuple.

```python
def get_min_max(numbers: list[int]) -> tuple[int, int]:
    return min(numbers), max(numbers)  # Returns a tuple

minimum, maximum = get_min_max([45, 12, 89, 3, 67])
```

#### Dictionary Keys (Tuples as Compound Keys)
Because tuples are hashable, they are frequently used as multi-dimensional keys in dictionaries or matrix lookups:

```python
# Grid coordinates mapping to terrain type
world_map = {
    (0, 0): "Grassland",
    (0, 1): "Forest",
    (1, 0): "Mountain"
}

print(world_map[(0, 1)])  # "Forest"
```

---

## 🛠️ Practical Hands-On Project: System Config & Geofencing Evaluator

This project demonstrates how tuples protect read-only system settings and coordinate records from accidental mutation during execution.

```python
# filename: config_geofence_checker.py
import math

# Fixed system configuration stored as a tuple of tuples
DATABASE_NODES = (
    ("Primary", "192.168.1.10", 5432),
    ("Replica1", "192.168.1.11", 5432),
    ("Replica2", "192.168.1.12", 5432),
)

# Target geofence boundary center (Latitude, Longitude, Radius in KM)
SAFE_ZONE_CENTER = (37.7749, -122.4194)  # San Francisco
MAX_RADIUS_KM = 15.0

def calculate_distance(coord1: tuple[float, float], coord2: tuple[float, float]) -> float:
    """Calculates approximate Euclidean distance in KM between two coordinates."""
    lat1, lon1 = coord1
    lat2, lon2 = coord2
    
    # Simple conversion factor for local distance estimation
    km_per_deg_lat = 111.0
    km_per_deg_lon = 88.0
    
    d_lat = (lat2 - lat1) * km_per_deg_lat
    d_lon = (lon2 - lon1) * km_per_deg_lon
    
    return math.sqrt(d_lat**2 + d_lon**2)

def evaluate_device_location(device_id: str, location: tuple[float, float]) -> tuple[bool, float]:
    """Checks if device is within safe geofence zone and returns status with distance."""
    distance = calculate_distance(SAFE_ZONE_CENTER, location)
    is_safe = distance <= MAX_RADIUS_KM
    return is_safe, distance

if __name__ == "__main__":
    print("=== System Database Node Registry ===")
    for name, ip, port in DATABASE_NODES:
        print(f"Node: {name} | Host: {ip}:{port}")

    print("\n=== Device Geofence Evaluation ===")
    test_device_loc = (37.8044, -122.2712)  # Oakland, CA
    in_zone, dist_km = evaluate_device_location("DEV_4092", test_device_loc)
    
    status_str = "INSIDE" if in_zone else "OUTSIDE"
    print(f"Device Location: {test_device_loc}")
    print(f"Status: {status_str} safe zone ({dist_km:.2f} km away)")
```

---

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **Why are tuples slightly more memory-efficient and faster than lists in Python?**
   * *Answer:* Lists are dynamic and over-allocate extra memory buffer space to handle future `append()` calls in $O(1)$ amortized time. Tuples are immutable and fixed size at creation; Python allocates exact memory without overhead. Python also caches small tuples internally to optimize allocation.

2. **Can a tuple be used as a key in a Python dictionary?**
   * *Answer:* Yes, but **only if all elements within the tuple are also immutable/hashable**. A tuple containing a mutable object like a list (e.g., `(1, [2, 3])`) is unhashable and raises a `TypeError: unhashable type: 'list'`.

3. **What happens if you omit parentheses when returning multiple items in a function?**
   * *Answer:* Python automatically performs **tuple packing**. Writing `return x, y, z` is syntactically equivalent to `return (x, y, z)`.

---

## 📚 Curated Resources & References

### Official Documentation & Guides
* [Python Official Documentation - Tuples and Sequences](https://docs.python.org/3/tutorial/datastructures.html#tuples-and-sequences) – Official language specifications.
* [Real Python: Understanding Tuples in Python](https://realpython.com/python-tuples/) – Detailed exploration of tuple use cases and immutability mechanics.

### Interactive Learning Tools
* [GeeksforGeeks: Python Tuples](https://www.geeksforgeeks.org/python-tuples/) – Quick reference guide with examples and operations.
* [W3Schools Python Tuples](https://www.w3schools.com/python/python_tuples.asp) – Hands-on code practice environment.