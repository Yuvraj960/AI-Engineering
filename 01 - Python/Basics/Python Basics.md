# Module 01: Python Basics

Welcome to the **Python Basics** module of the AI Engineering track. This document covers the foundational concepts, core syntax, control structures, and environment setup required to build a strong base in Python.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:
1. **Describe** the history, core features, and real-world applications of Python in AI, Data Engineering, and Web Development.
2. **Explain** Python's fundamental syntax rules, indentation mechanics, dynamic typing, and execution model.
3. **Identify & Utilize** built-in data types, variables, and operators (arithmetic, comparison, logical).
4. **Apply** core control flow structures (`if-else`, `for`/`while` loops) and standard functions to solve computational tasks.
5. **Set Up & Execute** Python scripts using the Python REPL, VS Code, or PyCharm environments.

---

## 🧠 Core Topics & In-Depth Notes

### 1. Introduction to Python & Design Philosophy
* **History:** Created by **Guido van Rossum** in late 1989 and released in 1991. Named after *Monty Python's Flying Circus*.
* **Key Features:**
  * **Interpreted & Dynamically Typed:** Code is executed line-by-line; variable types are resolved at runtime.
  * **Readability First:** Follows the principles outlined in *The Zen of Python* (`import this`).
  * **Cross-Platform:** Runs natively on Linux, macOS, and Windows without platform-specific code changes.

```python
# The Zen of Python snippet
import this

```

---

### 2. Environment Setup & Execution Models

| Execution Mode | Best Used For | Key Command / Usage |
| --- | --- | --- |
| **Python REPL** | Quick testing, inspecting variables, debugging snippets | Run `python` or `python3` in terminal |
| **Script Execution** | Executing full Python files (`.py`) | `python main.py` |
| **IDE / Code Editor** | Full project development with debugging & linting | VS Code, PyCharm |

---

### 3. Python Syntax Rules & Mechanics

#### Indentation & Whitespace

Python uses **4 spaces per indentation level** (PEP 8 standard) instead of curly braces `{}` or `begin/end` blocks.

```python
# Correct Indentation
if True:
    print("Properly indented block")
else:
    print("Else block")

```

#### Variables & Dynamic Typing

Variables are references to objects stored in memory. You do not need to declare types explicitly.

```python
x = 10         # int
x = "Hello"    # x now references a string (Dynamic Typing)

```

---

### 4. Primitive Data Types & Built-In Operators

#### Data Types

* **Numeric:** `int` (e.g., `42`), `float` (e.g., `3.14`), `complex` (e.g., `2 + 3j`)
* **Text:** `str` (e.g., `"AI Engineering"`)
* **Boolean:** `bool` (`True`, `False`)
* **NoneType:** `None` (represents absence of value)

#### Operators Quick Reference

```python
# Arithmetic
a, b = 10, 3
print(a + b)   # 13 (Addition)
print(a // b)  # 3  (Floor Division)
print(a ** b)  # 1000 (Exponentiation)

# Comparison & Logical
print(a > 5 and b < 5)  # True
print(not (a == b))     # True

```

---

### 5. Control Flow & Basic Functions

#### Conditional Statements

```python
score = 85

if score >= 90:
    print("Grade: A")
elif score >= 80:
    print("Grade: B")
else:
    print("Grade: C")

```

#### Loops (`for` and `while`)

```python
# For Loop with range()
for i in range(3):
    print(f"Iteration {i}")

# While Loop
count = 3
while count > 0:
    print(f"Countdown: {count}")
    count -= 1

```

#### Basic Functions

```python
def greet_developer(name: str) -> str:
    """Returns a welcome message for a given developer."""
    return f"Welcome to AI Engineering, {name}!"

message = greet_developer("Alex")
print(message)

```

---

## 🛠️ Practical Hands-On Project: Number Guessing Game

Put these concepts into practice by building a basic interactive script.

```python
# filename: number_guessing_game.py
import random

def run_game():
    target_number = random.randint(1, 20)
    attempts = 0
    max_attempts = 5
    
    print("=== Welcome to the Number Guessing Game ===")
    print("Guess a number between 1 and 20.")
    
    while attempts < max_attempts:
        guess_input = input(f"Attempt {attempts + 1}/{max_attempts} - Enter your guess: ")
        
        # Basic validation
        if not guess_input.isdigit():
            print("Please enter a valid positive integer.")
            continue
            
        guess = int(guess_input)
        attempts += 1
        
        if guess == target_number:
            print(f"🎉 Correct! You guessed the number in {attempts} attempt(s).")
            break
        elif guess < target_number:
            print("Too low! Try higher.")
        else:
            print("Too high! Try lower.")
    else:
        print(f"❌ Game Over! The correct number was {target_number}.")

if __name__ == "__main__":
    run_game()

```

---

## ❓ Frequently Asked Exam / Research Questions

1. **Why is Python dynamically typed, and how does it differ from statically typed languages?**
* *Answer:* In Python, type checking occurs at runtime rather than compile time. Variables are tags bound to objects, allowing a variable to change types throughout execution.


2. **When should you use the REPL vs an IDE?**
* *Answer:* Use the REPL for quick expression evaluations, testing built-in methods, or simple debugging. Use an IDE for multi-file projects, writing modular code, running debuggers, and managing virtual environments.


3. **How does Python's reliance on indentation impact program execution?**
* *Answer:* Indentation defines code blocks. Mismatched or missing indentation causes `IndentationError` or `TabError` at execution time, enforcing consistent readability across codebases.



---

## 📚 Curated Resources & References

### Official Documentation & Tutorials

* [Python 3 Official Tutorial](https://docs.python.org/3/tutorial/index.html) – The definitive source for core syntax.
* [Real Python: Python 3 Basics](https://realpython.com/python-first-steps/) – In-depth guide with practical examples.
* [PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/) – Official code formatting standard.

### Interactive Learning Tools

* [W3Schools Python Tutorial](https://www.w3schools.com/python/) – Interactive code runner for testing basic constructs.
* [Codecademy Learn Python 3](https://www.codecademy.com/learn/learn-python-3) – Hands-on browser-based practice.

```

```