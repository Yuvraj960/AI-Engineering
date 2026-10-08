# Module 04: Output Formatting, Structured Generation, JSON & Pydantic

## 1. Executive Summary & Core Concepts

In software engineering, LLM outputs must interface cleanly with databases, microservices, frontend applications, and downstream APIs. Free-form text responses are inherently probabilistic and unformatted, making direct programmatic ingestion risky. 

**Structured Generation** transforms non-deterministic language model output into deterministic, schema-compliant data structures (primarily JSON) that adhere strictly to pre-defined constraints.

### 1.1 Structural Mechanics: How Models Output JSON

When an LLM generates structured data, it can be guided through two distinct mechanisms:

1. **Prompt-Based Formatting (Soft Constraints)**: The prompt instructs the model to adhere to a JSON schema (e.g., providing a JSON schema in text or TypeScript interfaces). The model relies on instruction tuning to format the output string correctly.
2. **Grammar-Guided / Constrained Decoding (Hard Constraints)**: At the decoding layer during token sampling, the inference engine (e.g., vLLM, OpenAI Structured Outputs, Llama.cpp) applies a context-free grammar (CFG) or regex mask over the model's vocabulary. Tokens that violate the target JSON schema are assigned a probability of zero ($P(\text{token}) = 0$), mathematically guaranteeing schema compliance.

---

## 2. Prompt Engineering for JSON Output

### 2.1 Schema Specification Strategies

To maximize schema compliance without constrained decoding, schemas can be represented in prompts using standard formal notation:

* **JSON Schema (Draft-07 / 2020-12)**: The formal standard used natively by OpenAI, Anthropic, and Pydantic.
* **TypeScript Interface Definitions**: Frequently consumed more efficiently by models due to code pre-training density.
* **XML-Wrapped Templates**: Useful for hybrid outputs (e.g., reasoning inside `<thinking>` tags, followed by JSON in `<json>` tags).

### 2.2 Common Failure Modes in Text-Based Structured Output

* **Trailing Commas**: `{"keys": ["val1", "val2",]}` breaking standard JSON parsers.
* **Markdown Wrapper Pollution**: Enclosing output in ```json ... ``` blocks when raw JSON is expected.
* **Hallucinated Keys**: Generating extra fields not defined in the target schema.
* **Truncated JSON**: Reaching max token limits mid-object, leaving unclosed brackets or strings.

---

## 3. Pydantic v2 & Type-Safe Schema Enforcement

Pydantic v2 is the industry standard for schema definition and runtime data validation in Python AI applications. It converts Python class models into valid JSON Schemas and parses runtime strings into strongly typed Python objects.

### 3.1 Pydantic Model Design for LLM Schemas

```python
from typing import List, Optional, Literal
from pydantic import BaseModel, Field, FieldValidationInfo, field_validator

class EntityExtraction(BaseModel):
    entity_name: str = Field(description="The primary name of the identified entity.")
    category: Literal["ORGANIZATION", "PERSON", "LOCATION", "PRODUCT"] = Field(
        description="The category classification of the entity."
    )
    confidence_score: float = Field(
        ge=0.0, le=1.0, description="Confidence score between 0.0 and 1.0"
    )
    metadata: Optional[List[str]] = Field(
        default=None, description="Optional key attributes associated with the entity."
    )

    @field_validator("entity_name")
    @classmethod
    def name_must_be_capitalized(cls, v: str) -> str:
        if not v[0].isupper():
            raise ValueError("Entity name must start with a capital letter.")
        return v
```

---

## 4. Production Architectural Patterns

### 4.1 Hybrid Chain-of-Thought with Structured Output

To preserve reasoning capabilities, models should perform Chain-of-Thought (CoT) *before* emitting structured JSON. Forcing an LLM to immediately start with `{` suppresses intermediate reasoning space and degrades performance on complex tasks.

**Pattern Template:**
```text
Analyze the text step-by-step. First, write your reasoning inside <thinking> tags. 
Then, provide the final structured result inside <json_output> tags matching the JSON schema below.

Schema:
{schema_json}
```

### 4.2 Retry & Self-Correction Loops

When runtime parsing fails (e.g., Pydantic raises a `ValidationError` or `JSONDecodeError`), the exception message can be fed back into the LLM context to prompt self-correction.

```text
[User Prompt]: Extract user data.
[Assistant Response]: {"name": "Alice", "age": "twenty"}
[System Validation Error]: ValidationError: Field 'age' must be a valid integer.
[Correction Prompt]: Your previous JSON response failed validation with the following error:
ValidationError: Field 'age' must be a valid integer.
Please fix the error and output only valid JSON.
```

---

## 5. Production Python Implementation

The following production script demonstrates schema generation, manual JSON extraction with validation, and an automated retry/self-correction pipeline using Pydantic v2.

```python
import json
import re
from typing import List, Literal, Optional, Type, TypeVar
from pydantic import BaseModel, Field, ValidationError

T = TypeVar("T", bound=BaseModel)

# ------------------------------------------------------------------
# 1. Target Pydantic Schema
# ------------------------------------------------------------------
class ActionItem(BaseModel):
    task: str = Field(description="Clear action statement describing the task.")
    assignee: str = Field(description="Person responsible for completing the task.")
    priority: Literal["HIGH", "MEDIUM", "LOW"] = Field(description="Task priority level.")
    due_date: Optional[str] = Field(default=None, description="ISO formatted date string (YYYY-MM-DD) if specified.")

class MeetingSummary(BaseModel):
    summary: str = Field(description="Concise summary of meeting discussions.")
    key_decisions: List[str] = Field(description="List of finalized decisions made during the meeting.")
    action_items: List[ActionItem] = Field(description="List of actionable tasks assigned.")

# ------------------------------------------------------------------
# 2. Parsing & Self-Correction Utilities
# ------------------------------------------------------------------
def extract_json_from_response(raw_response: str) -> str:
    """
    Extracts raw JSON string from responses containing markdown code blocks or thought tags.
    """
    # Strips markdown code blocks if present
    json_match = re.search(r"```(?:json)?\s*([\s\S]*?)\s*```", raw_response)
    if json_match:
        return json_match.group(1).strip()
    
    # Searches for outermost braces
    brace_match = re.search(r"\{[\s\S]*\}", raw_response)
    if brace_match:
        return brace_match.group(0).strip()
        
    return raw_response.strip()

def parse_and_validate(raw_response: str, model_cls: Type[T]) -> T:
    """
    Parses and validates LLM output string against a Pydantic model class.
    """
    clean_json_str = extract_json_from_response(raw_response)
    parsed_dict = json.loads(clean_json_str)
    return model_cls.model_validate(parsed_dict)

def generate_prompt_with_schema(user_input: str, model_cls: Type[BaseModel]) -> str:
    """
    Builds a prompt embedding the Pydantic JSON schema into the prompt.
    """
    schema_json = json.dumps(model_cls.model_json_schema(), indent=2)
    return f"""Analyze the provided meeting notes and extract structured data matching the JSON Schema below.

[JSON SCHEMA]
{schema_json}

[INSTRUCTIONS]
1. Ensure all required fields are present.
2. Return ONLY the valid JSON object inside a ```json ``` block.

[INPUT TEXT]
{user_input}
"""

# ------------------------------------------------------------------
# 3. Simulated LLM Calling Pipeline with Self-Correction Loop
# ------------------------------------------------------------------
def mock_llm_call(prompt: str, simulate_error: bool = False) -> str:
    """
    Simulates an API call to an LLM endpoint.
    """
    if simulate_error:
        # Invalid payload: missing required field 'assignee' inside action_items
        return """
        ```json
        {
            "summary": "Discussed Q3 product rollout and infrastructure planning.",
            "key_decisions": ["Deploy model to vLLM cluster"],
            "action_items": [
                {
                    "task": "Set up monitoring dashboard",
                    "priority": "HIGH"
                }
            ]
        }
        ```
        """
    else:
        # Valid payload
        return """
        ```json
        {
            "summary": "Discussed Q3 product rollout and infrastructure planning.",
            "key_decisions": ["Deploy model to vLLM cluster"],
            "action_items": [
                {
                    "task": "Set up monitoring dashboard",
                    "assignee": "Alex",
                    "priority": "HIGH",
                    "due_date": "2026-10-15"
                }
            ]
        }
        ```
        """

def execution_pipeline_with_retry(meeting_notes: str, max_retries: int = 2):
    prompt = generate_prompt_with_schema(meeting_notes, MeetingSummary)
    print("--- GENERATED PROMPT ---")
    print(prompt)
    
    for attempt in range(1, max_retries + 2):
        print(f"\n--- ATTEMPT {attempt} ---")
        # First attempt simulates error to demonstrate correction loop
        simulate_err = (attempt == 1) 
        raw_response = mock_llm_call(prompt, simulate_error=simulate_err)
        
        try:
            validated_obj = parse_and_validate(raw_response, MeetingSummary)
            print("\n✅ Parsing & Validation Successful!")
            print(f"Validated Object Type: {type(validated_obj)}")
            print(json.dumps(validated_obj.model_dump(), indent=2))
            return validated_obj
        except (json.JSONDecodeError, ValidationError) as e:
            print(f"❌ Validation Failed on Attempt {attempt}: {e}")
            if attempt <= max_retries:
                # Constructing correction prompt append
                correction_msg = f"\n\n[SYSTEM ERROR]: Your previous output failed validation:\n{str(e)}\nFix the error and output valid JSON."
                prompt += correction_msg

# ------------------------------------------------------------------
# 4. Entrypoint
# ------------------------------------------------------------------
if __name__ == "__main__":
    sample_notes = "Alex will set up the monitoring dashboard for the high priority vLLM rollout by Oct 15."
    execution_pipeline_with_retry(sample_notes)
```

---

## 6. Summary Comparison Matrix

| Feature / Dimension | Prompt-Based Formatting | Constrained Decoding (Grammar/CFG) | Pydantic Post-Validation Loop |
| ----- | ----- | ----- | ----- |
| **Enforcement Layer** | Soft (Model Attention) | Hard (Token Sampling Probability Mask) | Runtime Execution / Exception Loop |
| **Schema Latency Overhead** | Zero | Negligible to Moderate (Grammar Compilation) | Low (Python Validation Cost) |
| **Portability** | Works across all API providers | Requires supporting backend (vLLM, Outlines, Ollama) | Framework agnostic |
| **Guarantee of Compliance** | ~85%–98% depending on model scale | 100% mathematically constrained | Guarantees catching errors via retry |
| **Handling Intermediary Reasoning** | Easy (CoT before JSON) | Requires specialized multi-syntax grammar | Easy (Extract JSON via Regex/XML) |
