# 🐍 Python Mastery for AI Engineering & Distributed Systems

Welcome to the **Python for AI Engineering** curriculum repository. This repository contains a production-grade, end-to-end curriculum designed to transition software engineers into specialized **AI Engineers** capable of building, scaling, testing, and optimizing high-throughput AI/ML systems, data pipelines, and LLM integrations.

---

## 📌 Repository Overview & Index of Modules

Below is the complete index of all 12 modules included in this learning directory. Select a file name to open the module.

| Module # | File Name | Primary Scope & Focus Areas |
| :--- | :--- | :--- |
| **Module 01** | [Python Basics.md](Basics/Python%20Basics.md) | Python fundamentals, syntax, variables, data types, operators, control flow, and environment setup |
| **Module 02** | [Text Handling - Strings.md](Collections/Text%20Handling%20-%20Strings.md) | String immutability, indexing, slicing, methods, formatting, and practical text processing |
| **Module 03** | [Immutable Sequences - Tuples.md](Collections/Immutable%20Sequences%20-%20Tuples.md) | Tuple immutability, operations, packing and unpacking, multiple returns, and fixed data |
| **Module 04** | [Dynamic Sequences - Lists.md](Collections/Dynamic%20Sequences%20-%20Lists.md) | List mutability, operations, comprehensions, nested lists, and sorting |
| **Module 05** | [Unique Collections - Sets.md](Collections/Unique%20Collections%20-%20Sets.md) | Set uniqueness, hashability, membership, mathematical operations, and deduplication |
| **Module 06** | [Key Value Pairs - Dictionaries.md](Collections/Key%20Value%20Pairs%20-%20Dictionaries.md) | Dictionary keys and values, lookup and update operations, views, and nested mappings |
| **Module 07** | [Code Reusability - Functions and Modules.md](Code%20Reusability%20-%20Functions%20and%20Modules.md) | Functions, arguments, return values, scope, imports, built-in modules, and project structure |
| **Module 08** | [Iterators, Generators and Decorators.md](Iterators,%20Generators%20and%20Decorators.md) | Iteration protocol, lazy generators, memory-efficient processing, and decorators |
| **Module 09** | [Exception Handling and Context Managers.md](Exception%20Handling%20and%20Context%20Managers.md) | Exception flow, custom exceptions, `try`/`except` patterns, and context-managed resources |
| **Module 10** | [Object Oriented Programming OOPS.md](Object%20Oriented%20Programming%20OOPS.md) | Classes and objects, encapsulation, inheritance, polymorphism, abstraction, and dunder methods |
| **Module 11** | [Concurrency, Multithreading, Multiprocessing and AsyncIO.md](Concurrency,%20Multithreading,%20Multiprocessing%20and%20AsyncIO.md) | Concurrency and parallelism, GIL, threads, processes, synchronization, IPC, and AsyncIO |
| **Module 12** | [Unit Testing with PyTest and Mocking.md](Unit%20Testing%20with%20PyTest%20and%20Mocking.md) | Unit testing, `unittest`, mocking, Pytest fixtures, parameterization, and test design |

---

## 🚀 Key Curriculum Themes & Capabilities

This curriculum goes beyond foundational Python to address the specific performance and architectural requirements of modern AI engineering:

### 1. High-Throughput & Async AI Workloads
* **AsyncIO & Network I/O**: Stream tokens efficiently from LLM endpoints (OpenAI, Anthropic) using non-blocking event loops without thread context-switching overhead.
* **Multiprocessing**: Bypass CPython's **Global Interpreter Lock (GIL)** to parallelize CPU-heavy feature engineering, tokenization, matrix manipulations, and data transformations.

### 2. Type Safety & Dynamic Data Validation
* **Static Typing**: Utilize Python's `typing` system (`Generics`, `Protocol`, `TypeVar`) to maintain large codebases.
* **Pydantic V2**: Enforce strict runtime data validation and serialization schemas for incoming API payloads and structured LLM outputs.

### 3. Reliability, Testing & Production Observability
* **Pytest & Dependency Isolation**: Build test suites following the **Arrange-Act-Assert (AAA)** pattern using custom fixtures and `unittest.mock` to isolate external services.
* **Structured Logging & Error Handling**: Implement structured JSON logging and robust exception handling for observability in production environments.

---

## 🛠️️ Environment Setup & Installation

To run and experiment with the scripts across these modules, set up a local Python environment using standard tooling.

### 1. Prerequisites
* **Python 3.10+** (Python 3.11 or 3.12 recommended for async and typing improvements)
* `pip` and `venv` (or `uv` / `poetry`)

### 2. Virtual Environment Setup

```bash
# Clone or navigate to directory
cd python-ai-engineering

# Create virtual environment
python3 -m venv .venv

# Activate virtual environment
# On Linux/macOS:
source .venv/bin/activate
# On Windows:
# .venv\Scripts\activate

# Upgrade installer tools
pip install --upgrade pip setuptools
```

### 3. Install Dependencies

The code samples across these modules utilize standard libraries (`threading`, `multiprocessing`, `asyncio`, `unittest`, `json`, `math`) alongside key third-party packages:

```bash
pip install pytest pytest-mock pydantic typing-extensions
```

---

## 🧪 Running Tests & Examples

### Running Individual Module Scripts
To execute any hands-on module pipeline (such as the hybrid concurrency pipeline in Module 11):

```bash
python hybrid_concurrency_pipeline.py
```

### Executing Pytest Test Suites
To run unit tests and inspect detailed assertion introspection:

```bash
# Run all tests in the folder
pytest -v

# Run a specific test module
pytest test_ai_pipeline.py -v

# Run with stdout output enabled
pytest -s
```

---

## 📚 General References & Further Reading

* [Python Official Documentation](https://docs.python.org/3/)
* [Pytest Framework Documentation](https://docs.pytest.org/en/stable/)
* [Pydantic V2 Documentation](https://docs.pydantic.dev/latest/)
* [Real Python – Advanced Python Tutorials](https://realpython.com/)

---

*Congratulations on completing the **Python for AI Engineering** module! You now have a solid foundation in Python's advanced concurrency models, type systems, performance tuning, and unit testing frameworks.*