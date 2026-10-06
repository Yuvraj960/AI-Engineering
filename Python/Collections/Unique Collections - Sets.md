# Module 05: Unique Collections - Sets

Welcome to the **Unique Collections - Sets** module of the AI Engineering track. A Python set is an unordered collection of unique and mutable elements based on hash table mechanics. This module covers set characteristics, membership testing, mathematical operations (unions, intersections, differences), `frozenset` instances, and practical deduplication techniques.

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Describe** the key characteristics of Python sets, including their unordered nature, uniqueness constraints, and hashability requirements for elements.

2. **Explain** structural and behavioral differences between sets, lists, tuples, and dictionaries.

3. **Apply** essential set modification and inspection methods (`add`, `remove`, `discard`, `pop`, `update`, `in`, `not in`).

4. **Analyze & Perform** mathematical set operations including union ($\cup$), intersection ($\cap$), difference ($\setminus$), and symmetric difference ($\Delta$) using both operators and methods.

5. **Evaluate** use cases for `frozenset` and implement practical solutions for data deduplication and overlap analysis.

## 🧠 Core Topics & In-Depth Notes

### 1. Definition, Syntax & Hashability Requirements

A **set** is an unordered collection of unique elements enclosed in curly braces `{}` or created via the `set()` constructor.

* **Empty Set Syntax Trap:** An empty `{}` creates an empty dictionary (`dict`), NOT a set. You must use `set()` to instantiate an empty set.

  ```python
  empty_dict = {}       # Type: dict
  empty_set = set()     # Type: set
  ```

* **Uniqueness & Unordered Nature:** Sets automatically enforce duplicate suppression and do not preserve insertion order for indexing or slicing (i.e., `my_set[0]` raises a `TypeError`).

* **Hashability Requirement:** All elements stored inside a set must be **hashable** (immutable objects like `int`, `str`, `tuple`). Mutable objects like `list`, `dict`, or another `set` cannot be added to a set.

  ```python
  valid_set = {1, "AI", (10, 20)}  # Valid
  # invalid_set = {1, [10, 20]}    # Raises TypeError: unhashable type: 'list'
  ```

### 2. Modifying Sets & Safe Element Removal

| Method | Behavior | Error on Missing Element? |
| :--- | :--- | :--- |
| `add(elem)` | Adds a single element to the set. | N/A |
| `update(iterable)` | Adds all elements from an iterable (`list`, `tuple`, etc.) to the set. | N/A |
| `remove(elem)` | Removes `elem` from the set. | **Yes** (`KeyError`) |
| `discard(elem)` | Removes `elem` if present; does nothing if missing. | **No** (Safe) |
| `pop()` | Removes and returns an arbitrary element. | **Yes** (`KeyError` if empty) |
| `clear()` | Removes all items from the set. | No |

```python
tech_stack = {"Python", "PyTorch", "Docker"}

# Safe removal vs Strict removal
tech_stack.discard("Kubernetes")  # No error raised
# tech_stack.remove("Kubernetes") # Raises KeyError!

tech_stack.update(["FastAPI", "PostgreSQL"])
print(tech_stack)
```

### 3. Mathematical Set Operations

Sets excel at comparing collections and finding relationships between data subsets.

| Operation | Mathematical Symbol | Python Operator | Python Method | Description |
| :--- | :---: | :---: | :--- | :--- |
| **Union** | $A \cup B$ | `a \| b` | `a.union(b)` | Elements in $A$, $B$, or both. |
| **Intersection** | $A \cap B$ | `a & b` | `a.intersection(b)` | Elements common to both $A$ and $B$. |
| **Difference** | $A \setminus B$ | `a - b` | `a.difference(b)` | Elements in $A$ but NOT in $B$. |
| **Symmetric Difference** | $A \Delta B$ | `a ^ b` | `a.symmetric_difference(b)` | Elements in $A$ or $B$, but NOT both. |

```python
group_a = {"Python", "SQL", "Git"}
group_b = {"Python", "Docker", "AWS"}

# Union: All unique tools across both groups
print(group_a | group_b)  # {'Python', 'SQL', 'Git', 'Docker', 'AWS'}

# Intersection: Tools shared by both groups
print(group_a & group_b)  # {'Python'}

# Difference: Tools in group_a but not in group_b
print(group_a - group_b)  # {'SQL', 'Git'}

# Symmetric Difference: Tools exclusive to one group only
print(group_a ^ group_b)  # {'SQL', 'Git', 'Docker', 'AWS'}
```

#### Subset and Superset Relations

* `a.issubset(b)` or `a <= b`: Returns `True` if all elements of `a` exist in `b`.
* `a.issuperset(b)` or `a >= b`: Returns `True` if `a` contains all elements of `b`.
* `a.isdisjoint(b)`: Returns `True` if `a` and `b` have zero overlapping elements.

### 4. Immutable Sets (`frozenset`)

A `frozenset` is an immutable, hashable version of a set. Once created, its elements cannot be added or removed.

* **Use Case:** Because `frozenset` objects are hashable, they can be used as **keys in dictionaries** or as **elements of another set**.

```python
# Frozenset as a dictionary key
permissions_map = {
    frozenset(["read", "write"]): "Editor Role",
    frozenset(["read"]): "Viewer Role"
}

user_perms = frozenset(["read", "write"])
print(permissions_map[user_perms])  # "Editor Role"
```

## 🛠️ Practical Hands-On Project: User Permission & Dataset Overlap Analyzer

This project processes active user activity logs to detect user roles, analyze platform permissions, and filter distinct user groups.

```python
# filename: permissions_analyzer.py

def analyze_user_access():
    # Dataset 1: Registered users with active API keys
    api_users = ["usr_101", "usr_102", "usr_105", "usr_108", "usr_101", "usr_110"]
    
    # Dataset 2: Users flagged for administrative privileges
    admin_users = {"usr_102", "usr_108", "usr_112"}
    
    # Dataset 3: Users currently suspended due to security alerts
    suspended_users = {"usr_105", "usr_110", "usr_115"}

    # 1. Deduplicate API users list using set conversion
    unique_api_users = set(api_users)
    print("=== Unique API Users ===")
    print(f"Total entries: {len(api_users)} | Unique count: {len(unique_api_users)}")
    print(f"User IDs: {sorted(list(unique_api_users))}")

    # 2. Identify Active Admins (API Users who are Admins)
    active_admins = unique_api_users & admin_users
    print("\n=== Active Admin Users ===")
    print(f"Active Admins: {active_admins}")

    # 3. Identify Valid Non-Suspended Users (API Users excluding Suspended Users)
    valid_active_users = unique_api_users - suspended_users
    print("\n=== Valid Active (Non-Suspended) Users ===")
    print(f"Valid Users: {sorted(list(valid_active_users))}")

    # 4. Find Security Edge Cases (Suspended users who still have Admin role)
    suspended_admins = admin_users & suspended_users
    print("\n=== Security Alert: Suspended Admin Accounts ===")
    if suspended_admins:
        print(f"WARNING: Suspended Admin IDs detected: {suspended_admins}")
    else:
        print("No suspended admin accounts found.")

    # 5. Check if all active admins are part of the main API user set
    is_subset = active_admins.issubset(unique_api_users)
    print(f"\nAre active admins subset of API users? {is_subset}")

if __name__ == "__main__":
    analyze_user_access()
```

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **Why is membership testing (`item in container`) significantly faster in a set than in a list?**

   * *Answer:* Sets use a **hash table** structure under the hood, enabling average $O(1)$ time complexity for membership lookups. In contrast, checking membership in a standard list requires an $O(n)$ linear scan through elements.

2. **What is the difference between `set.remove(x)` and `set.discard(x)`?**

   * *Answer:* Both methods delete the specified item `x` from the set. However, if `x` does not exist in the set, `remove(x)` raises a `KeyError`, whereas `discard(x)` completes safely without raising an error.

3. **Can a set contain a list or another set as an element?**

   * *Answer:* No. Set elements must be hashable and immutable. Standard `list` and `set` types are mutable and unhashable, so attempting to add them raises a `TypeError`. However, you can add an immutable `tuple` or a `frozenset`.

## 📚 Curated Resources & References

### Official Documentation & Guides

* [Python Official Documentation - Set Types (`set`, `frozenset`)](https://docs.python.org/3/library/stdtypes.html#set-types-set-frozenset) – Official API reference for set methods and operators.
* [Real Python: Sets in Python](https://realpython.com/python-sets/) – In-depth guide covering set theory concepts and pythonic set operations.

### Interactive Practice Platforms

* [GeeksforGeeks: Python Sets](https://www.geeksforgeeks.org/python-sets/) – Examples, edge cases, and practice problems on sets.
* [W3Schools Python Sets Tutorial](https://www.w3schools.com/python/python_sets.asp) – Hands-on code playground for practice with set methods.