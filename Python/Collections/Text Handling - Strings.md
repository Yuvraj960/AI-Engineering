# Module 02: Text Handling - Strings

Welcome to the **Text Handling - Strings** module of the AI Engineering track. Text manipulation is a fundamental skill in software engineering, natural language processing, and data pre-processing. This guide covers string immutability, indexing/slicing, essential string methods, formatting styles, and real-world parsing scenarios.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:
1. **Describe** string properties, memory models, immutability, and 0-based indexing.
2. **Explain** and execute slicing operations, sequence concatenation, repetition, and iteration.
3. **Apply** essential built-in string methods (`split`, `join`, `strip`, `replace`, `find`, `startswith`, `endswith`) to cleanse and transform raw text.
4. **Compare & Analyze** string formatting techniques (`%`, `.format()`, and `f-strings`) and select the most readable, performant approach.
5. **Implement** practical text processing scripts for tasks like log parsing, input validation, and dynamic report generation.

---

## 🧠 Core Topics & Deep Dive Notes

### 1. String Structure & Immutability

In Python, strings (`str`) are **immutable sequences of Unicode characters**. 

* **Immutability:** Once created in memory, a string object cannot be modified in place. Operations that appear to modify a string actually produce a *new* string instance.
* **Memory Implication:** Repeatedly modifying strings inside tight loops using `+=` creates multiple temporary string objects, leading to performance overhead. Joining a list of strings via `"".join()` is much more efficient ($O(N)$ vs $O(N^2)$).

```python
# Immutability in action
text = "hello"
# text[0] = "H"  # Raises TypeError: 'str' object does not support item assignment

# Correct approach: Create a new string
text = "H" + text[1:]
print(text)  # "Hello"
```

---

### 2. Indexing, Slicing, and Iteration

Strings use 0-indexed positioning from the start and negative indexing from the end.

```
 String:   P  y  t  h  o  n
 Index:    0  1  2  3  4  5
Neg Ind:  -6 -5 -4 -3 -2 -1
```

#### Syntax: `string[start:stop:step]`
* `start`: Inclusive starting index (default: 0).
* `stop`: Exclusive ending index (default: end of string).
* `step`: Index increment (default: 1).

```python
sample = "AI Engineering"

print(sample[0])       # 'A'
print(sample[-1])      # 'g'
print(sample[0:2])     # 'AI'
print(sample[3:])      # 'Engineering'
print(sample[::-1])    # 'gnireenignE IA' (Reversed string)
```

---

### 3. Essential Built-In String Methods

| Method | Description | Example Usage |
| :--- | :--- | :--- |
| `strip()`, `lstrip()`, `rstrip()` | Removes leading/trailing whitespace or target characters | `" hello ".strip() -> "hello"` |
| `split(sep)` | Splits string into a list of strings around `sep` | `"a,b,c".split(",") -> ['a', 'b', 'c']` |
| `join(iterable)` | Concatenates elements of an iterable using string as separator | `", ".join(['a', 'b']) -> "a, b"` |
| `replace(old, new)` | Replaces occurrences of a substring | `"py2".replace("2", "3") -> "py3"` |
| `find(sub)` / `index(sub)` | Returns index of substring (`find` returns -1 if not found; `index` raises `ValueError`) | `"abc".find("b") -> 1` |
| `startswith()` / `endswith()` | Checks if string begins or ends with a given prefix/suffix | `"test.py".endswith(".py") -> True` |

```python
# Data Cleaning Example
raw_input = "  USER_102, Active,  2026-10-05   \n"
cleaned_parts = [part.strip() for part in raw_input.strip().split(",")]
print(cleaned_parts)  # ['USER_102', 'Active', '2026-10-05']
```

---

### 4. String Formatting Techniques

Python supports three major approaches for string formatting:

#### A. `%` Formatting (Legacy)
```python
name = "Alice"
score = 95.5
print("Hello %s, your score is %.1f" % (name, score))
```

#### B. `str.format()` (Python 2.6+)
```python
print("Hello {}, your score is {:.1f}".format(name, score))
```

#### C. `f-strings` (Formatted String Literals - Python 3.6+ **Preferred**)
F-strings are evaluated at runtime, concise, fast, and allow embedding valid Python expressions directly.

```python
print(f"Hello {name.upper()}, your score is {score:.1f}")

# Escaping and Raw Strings
raw_path = r"C:\new_folder\test.txt"  # Prevents '\n' or '\t' escape interpretation
print(raw_path)
```

---

## 🛠️ Practical Hands-On Project: Log File Parser & Summary Generator

This practical script parses a raw server access log string, extracts essential fields, and formats an executive summary report.

```python
# filename: log_parser.py

raw_logs = """
2026-10-05 10:15:30 | INFO | User Login: admin_user
2026-10-05 10:16:05 | ERROR | Database Connection Failed
2026-10-05 10:17:12 | WARNING | High Memory Usage Detected
2026-10-05 10:18:00 | ERROR | Timeout on API Request
"""

def parse_logs(log_data: str):
    log_lines = log_data.strip().split("\n")
    stats = {"INFO": 0, "WARNING": 0, "ERROR": 0}
    error_messages = []

    for line in log_lines:
        if not line:
            continue
        parts = [p.strip() for p in line.split("|")]
        timestamp, log_level, message = parts[0], parts[1], parts[2]
        
        if log_level in stats:
            stats[log_level] += 1
            
        if log_level == "ERROR":
            error_messages.append(f"[{timestamp}] {message}")

    # Report Generation using F-Strings
    report = f"""
========================================
           LOG ANALYSIS REPORT
========================================
Total Entries Analyzed : {len(log_lines)}
INFO Logs              : {stats['INFO']}
WARNING Logs           : {stats['WARNING']}
ERROR Logs             : {stats['ERROR']}
----------------------------------------
Logged Errors:
"""
    for err in error_messages:
        report += f" - {err}\n"
        
    return report

if __name__ == "__main__":
    print(parse_logs(raw_logs))
```

---

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **Why are strings immutable in Python, and what are the advantages?**
   * *Answer:* Immutability ensures thread safety, allows strings to be used as hashable dictionary keys, and enables memory optimization via string interning (reusing identical immutable string instances in memory).

2. **What is the performance difference between `+=` in a loop and `str.join()`?**
   * *Answer:* `+=` creates a new string copy on every iteration ($O(N^2)$ time complexity for $N$ total characters). `str.join()` calculates total memory required first and constructs the output in a single pass ($O(N)$ time complexity).

3. **How do Raw Strings (`r"..."`) differ from standard strings?**
   * *Answer:* Raw strings treat backslashes `\` as literal characters rather than escape characters. For instance, `r"\n"` represents a two-character string containing `\` and `n`, whereas `"\n"` represents a newline character.

---

## 📚 Curated Resources & References

### Official Documentation & Guides
* [Python Official Documentation - Sequence Types (`str`)](https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str) – Complete reference of methods.
* [Real Python: Python Strings & Formatting](https://realpython.com/python-strings/) – Comprehensive breakdown of slicing and string methods.
* [PEP 498 – Formatted String Literals](https://peps.python.org/pep-0498/) – The original PEP proposing f-strings in Python.

### Interactive Tools
* [W3Schools Python Strings](https://www.w3schools.com/python/python_strings.asp) – Hands-on coding exercises for string operations.