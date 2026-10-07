# Unit Testing: Robust Testing Strategies with Pytest and Mocking

Unit testing is a foundational practice in software engineering and AI systems development. Reliable test suites allow teams to validate discrete software components in isolation, refactor models and data pipelines safely, prevent regressions, and ensure deterministic behavior across production deployments.

This module explores Python’s built-in `unittest` framework, dependency isolation with `unittest.mock`, and modern testing practices using **Pytest**—including fixtures, parameterization, and edge-case validation.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

1. **Understand** the role of unit testing, mocking, and isolated execution in producing reliable Python applications.
2. **Implement** object-oriented test suites using Python's standard `unittest` library.
3. **Isolate** external dependencies (APIs, databases, hardware, file I/O) using `unittest.mock.Mock` and `@patch`.
4. **Leverage** Pytest features including custom fixtures (`@pytest.fixture`), test parameterization (`@pytest.mark.parametrize`), and native Python `assert` statements.
5. **Analyze** failure outputs, inspect assertion introspection, and debug failing test cases efficiently.
6. **Design** clean, maintainable, and scalable test suites following the **Arrange-Act-Assert (AAA)** pattern.

---

## 🧠 Core Topics & Deep-Dive Concepts

### 1. Fundamentals of Unit Testing & The AAA Pattern

A **Unit Test** checks the smallest testable part of an application (a single function, class method, or module) in complete isolation from external components.

#### The Arrange-Act-Assert (AAA) Pattern
Every well-structured test follows three clear phases:
* **Arrange**: Set up the necessary objects, mock responses, or data inputs.
* **Act**: Execute the function or method under test.
* **Assert**: Verify that the actual output matches the expected behavior or outcome.

```
       +------------------------------------+
       | 1. Arrange                         |
       |    Set up inputs & dependencies    |
       +-----------------+------------------+
                         |
                         v
       +-----------------+------------------+
       | 2. Act                             |
       |    Invoke target function/method   |
       +-----------------+------------------+
                         |
                         v
       +-----------------+------------------+
       | 3. Assert                          |
       |    Verify output / side-effects    |
       +------------------------------------+
```

---

### 2. Standard Testing: Python `unittest`

Python’s built-in `unittest` module uses an object-oriented paradigm inspired by JUnit.

```python
import unittest

def calculate_discount(price: float, discount_pct: float) -> float:
    if not (0 <= discount_pct <= 1):
        raise ValueError("Discount percentage must be between 0.0 and 1.0")
    return price * (1 - discount_pct)


class TestDiscountCalculator(unittest.TestCase):

    def test_valid_discount(self):
        # Arrange & Act
        result = calculate_discount(100.0, 0.2)
        # Assert
        self.assertEqual(result, 80.0)

    def test_invalid_discount_raises_error(self):
        # Assert Exception
        with self.assertRaises(ValueError):
            calculate_discount(100.0, 1.5)

if __name__ == "__main__":
    unittest.main()
```

---

### 3. Dependency Isolation with `unittest.mock`

When unit testing code that interacts with external resources (such as database queries, external LLM endpoints, or filesystem I/O), we use **Mocking** to replace real calls with controllable substitutes.

#### Key Mock Tools
* `Mock` / `MagicMock`: Dummy objects that intercept calls and allow predefined return values or side effects.
* `patch`: Decorator or context manager used to intercept and replace real module/class references during test execution.

```python
import unittest
from unittest.mock import patch, MagicMock

# Source function under test
def fetch_user_role(api_client, user_id: int) -> str:
    response = api_client.get(f"/users/{user_id}")
    if response.status_code == 200:
        return response.json().get("role", "guest")
    raise ConnectionError("API Unavailable")


class TestAPIIntegration(unittest.TestCase):

    @patch("requests.get")  # Intercepting external call
    def test_fetch_user_role_success(self, mock_get):
        # Arrange Mock Response
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"role": "admin"}
        mock_get.return_value = mock_response

        # Act
        role = fetch_user_role(mock_response, user_id=42)

        # Assert
        self.assertEqual(role, "admin")
        mock_response.get.assert_called_once_with("/users/42")
```

---

### 4. Pytest: Modern, Pythonic Testing

`pytest` simplifies testing by removing boilerplate code. It allows testing with standard Python `assert` statements instead of verbose helper methods.

#### Benefits of Pytest over `unittest`
* **Plain Assert Statements**: Detailed assertion failure introspection without extra class boilerplate.
* **Fixtures (`@pytest.fixture`)**: Clean, reusable dependency injection for test setup and teardown.
* **Parameterization (`@pytest.mark.parametrize`)**: Easily run a single test function across multiple input/output scenarios.

---

## 📊 Comparison Matrix: `unittest` vs `pytest`

| Feature | `unittest` (Standard Library) | `pytest` (Third-Party / Industry Standard) |
| :--- | :--- | :--- |
| **Paradigm** | Object-Oriented (`TestCase` classes) | Functional or Object-Oriented |
| **Assertions** | Class methods (`self.assertEqual(a, b)`) | Native Python `assert a == b` with rich introspection |
| **Setup / Teardown** | `setUp()` / `tearDown()` methods | Reusable, composable `@pytest.fixture` functions |
| **Test Discovery** | Requires explicit runner setup or CLI flags | Automatic discovery of `test_*.py` and `*_test.py` files |
| **Parameterization** | Complex subtests (`self.subTest()`) | Concise `@pytest.mark.parametrize` decorator |
| **Ecosystem & Plugins**| Limited extension points | Rich ecosystem (`pytest-cov`, `pytest-mock`, etc.) |

---

## 🛠️ Practical Hands-On Project: AI Data Pipeline Test Suite

Below is a complete, executable Python test module built with **Pytest** and `unittest.mock`. It demonstrates testing a real-world scenario: an **AI Model Data Preprocessor and Inference Engine**.

```python
# filename: test_ai_pipeline.py

import pytest
from unittest.mock import Mock, patch
from typing import Dict, Any

# =====================================================================
# Application Code (Code Under Test)
# =====================================================================

class DataPipelineError(Exception):
    """Custom exception raised when data pipeline validation fails."""
    pass


class LLMClient:
    """Simulates an external LLM API client."""
    def query_model(self, prompt: str) -> Dict[str, Any]:
        # Real implementation would make an HTTP request here
        raise NotImplementedError("Real network calls forbidden in unit tests!")


class TextProcessingPipeline:
    """Preprocesses text and invokes an external LLM model."""
    def __init__(self, llm_client: LLMClient):
        self.llm_client = llm_client

    def preprocess_input(self, text: str) -> str:
        if not text or not text.strip():
            raise DataPipelineError("Input text cannot be empty or whitespace.")
        return text.strip().lower()

    def process_and_predict(self, raw_text: str) -> str:
        cleaned_prompt = self.preprocess_input(raw_text)
        response = self.llm_client.query_model(cleaned_prompt)
        return response.get("generated_text", "").upper()


# =====================================================================
# Pytest Test Suite
# =====================================================================

# 1. Pytest Fixture for Shared Dependencies
@pytest.fixture
def mock_llm_client() -> Mock:
    """Fixture providing a mocked LLM API client."""
    client = Mock(spec=LLMClient)
    client.query_model.return_value = {"generated_text": "processed response"}
    return client


@pytest.fixture
def pipeline(mock_llm_client: Mock) -> TextProcessingPipeline:
    """Fixture providing an initialized pipeline with a mocked client."""
    return TextProcessingPipeline(llm_client=mock_llm_client)


# 2. Parameterized Tests for Input Normalization
@pytest.mark.parametrize(
    "raw_input, expected_cleaned",
    [
        ("  Hello World  ", "hello world"),
        ("PyTest ENGINE ", "pytest engine"),
        ("  TRANSFORM\n ", "transform"),
    ]
)
def test_preprocess_input_normalization(pipeline: TextProcessingPipeline, raw_input: str, expected_cleaned: str):
    """Validates text normalization across various formatting edge cases."""
    assert pipeline.preprocess_input(raw_input) == expected_cleaned


# 3. Testing Exception Handling
@pytest.mark.parametrize("invalid_input", ["", "   ", "\t\n"])
def test_preprocess_input_raises_error_on_empty(pipeline: TextProcessingPipeline, invalid_input: str):
    """Ensures empty strings or whitespace raise DataPipelineError."""
    with pytest.raises(DataPipelineError, match="Input text cannot be empty"):
        pipeline.preprocess_input(invalid_input)


# 4. Testing External Dependency Mocking and Assertions
def test_process_and_predict_flow(pipeline: TextProcessingPipeline, mock_llm_client: Mock):
    """Verifies that processing coordinates properly with mocked LLM client."""
    raw_text = "  Analyze Sentiment  "
    
    # Act
    result = pipeline.process_and_predict(raw_text)

    # Assert Output Transformation
    assert result == "PROCESSED RESPONSE"

    # Assert Call Verification on Mock
    mock_llm_client.query_model.assert_called_once_with("analyze sentiment")


# 5. Simulating External Failures with Mock Side Effects
def test_process_and_predict_handles_api_failure(pipeline: TextProcessingPipeline, mock_llm_client: Mock):
    """Ensures system properly surfaces lower-level API errors."""
    # Configure mock to raise an error
    mock_llm_client.query_model.side_effect = ConnectionError("API connection lost")

    with pytest.raises(ConnectionError, match="API connection lost"):
        pipeline.process_and_predict("Valid Prompt")
```

---

## ❓ Technical Interview & Conceptual Exam Questions

1. **What is the difference between `@pytest.fixture` and standard `setUp()` in `unittest`?**
   * **Answer:** In `unittest`, `setUp()` runs implicitly before every single test method in a `TestCase` class, which can lead to redundant setup logic. In `pytest`, fixtures are explicitly injected into specific tests via parameter names, allowing modular reuse, composability, and configurable scopes (`function`, `class`, `module`, `session`).

2. **What is "over-mocking," and why should developers avoid it?**
   * **Answer:** Over-mocking occurs when tests mock out internal implementation details, helper functions, or core components rather than just external boundary interfaces (APIs, databases). This makes tests fragile—they break whenever internal code is refactored, even if the user-facing behavior remains correct.

3. **How does Pytest perform assertion introspection without requiring special method assertions?**
   * **Answer:** Pytest rewrites Python's native `assert` statements at import time. When an assertion fails, Pytest inspects the expression tree and prints detailed context about intermediate variable values, list differences, or dictionary keys without needing custom `assert*` methods.

4. **When should you use `side_effect` instead of `return_value` on a `Mock` object?**
   * **Answer:** `return_value` provides a static response every time the mock is called. Use `side_effect` when you need to raise an exception, return dynamic outputs based on inputs, or return different values across sequential calls.

---

## 📚 Curated Resources & References

### Official Documentation
* [Python Docs – `unittest` Framework](https://docs.python.org/3/library/unittest.html)
* [Python Docs – `unittest.mock` Library](https://docs.python.org/3/library/unittest.mock.html)
* [Pytest Official Documentation](https://docs.pytest.org/en/stable/)

### Technical Guides & Deep Dives
* [Real Python – Getting Started with Pytest](https://realpython.com/pytest-python-testing/)
* [Real Python – Understanding Pytest Fixtures and Parametrization](https://realpython.com/pytest-iterable/)
* [Pytest-Mock Plugin Documentation](https://pytest-mock.readthedocs.io/en/latest/)