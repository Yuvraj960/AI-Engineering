# Chapter 3: Structured Output & Output Parsers in LangChain

## 3.1 The Problem: Unstructured Text is Hard to Use

### Default LLM Output = Free Text
```python
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4o-mini")

result = model.invoke("Give me 3 facts about black holes")
print(result.content)
# Output:
# "1. Black holes are regions of spacetime where gravity is so strong...
# 2. The event horizon marks the boundary beyond which nothing can escape...
# 3. Stephen Hawking discovered that black holes emit radiation..."
```

**Problems with free text:**
| Issue | Impact |
|-------|--------|
| **No schema enforcement** | Keys missing, wrong types, extra fields |
| **Hard to parse programmatically** | Regex fragile, breaks on wording changes |
| **Cannot validate** | No guarantee of completeness/correctness |
| **Downstream systems need JSON/objects** | APIs, databases, UI components expect structured data |
| **Inconsistent formatting** | "Fact 1:" vs "1." vs "- " vs "• " |

### Solution: Structured Output
> **Structured Output = LLM returns data conforming to a predefined schema (JSON, Pydantic, TypedDict)**

```python
# Instead of parsing text, you get a validated object directly:
result = structured_model.invoke("Give me 3 facts about black holes")
# result = Review(key_themes=[...], summary="...", sentiment="pos", ...)
# type(result) → Review (Pydantic model) or dict (TypedDict/JSON)
```

---

## 3.2 Three Schema Definition Approaches

LangChain's `with_structured_output()` accepts **three schema formats**:

| Approach | Python Type | Validation | Best For |
|----------|-------------|------------|----------|
| **Pydantic** | `BaseModel` subclass | Full (types, constraints, defaults) | Production, complex schemas, IDE support |
| **TypedDict** | `TypedDict` class | Lightweight (type hints only) | Quick prototyping, simpler schemas |
| **JSON Schema** | `dict` | Standard JSON Schema | Dynamic schemas, cross-language, config-driven |

---

### 3.2.1 Pydantic Schema (Recommended for Production)

```python
# pydantic_demo.py
from pydantic import BaseModel, EmailStr, Field
from typing import Optional

class Student(BaseModel):
    name: str = "nitish"
    age: Optional[int] = None
    email: EmailStr
    cgpa: float = Field(gt=0, lt=10, default=5, description="CGPA (0-10)")

# Validation happens at instantiation
new_student = {"age": "32", "email": "abc@gmail.com"}  # age as string!
student = Student(**new_student)  # Auto-coerces "32" → 32

print(student.age)        # 32 (int)
print(student.model_dump_json())  # JSON string
```

**Full Structured Output Example:**
```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
from pydantic import BaseModel, Field
from typing import Optional, Literal

load_dotenv()
model = ChatOpenAI()

class Review(BaseModel):
    key_themes: list[str] = Field(description="Key themes discussed in the review")
    summary: str = Field(description="Brief summary of the review")
    sentiment: Literal["pos", "neg"] = Field(description="Sentiment: positive or negative")
    pros: Optional[list[str]] = Field(default=None, description="List of pros")
    cons: Optional[list[str]] = Field(default=None, description="List of cons")
    name: Optional[str] = Field(default=None, description="Reviewer name")

structured_model = model.with_structured_output(Review)

result = structured_model.invoke("""
I recently upgraded to the Samsung Galaxy S24 Ultra...
[long review text]
""")

print(result)
# Review(key_themes=['processor', 'camera', 'battery', 'bloatware', 'price'],
#        summary='The Samsung Galaxy S24 Ultra impresses with its powerful processor...',
#        sentiment='pos',
#        pros=['Insanely powerful processor', 'Stunning 200MP camera', ...],
#        cons=['Heavy for one-handed use', 'Bloatware in One UI', '$1,300 price tag'],
#        name='Nitish Singh')

# Access like object
print(result.sentiment)   # "pos"
print(result.pros)        # [...]
```

**Pydantic Advantages:**
- ✅ Runtime validation + coercion (string "32" → int 32)
- ✅ Field constraints (`gt`, `lt`, `min_length`, `pattern`)
- ✅ Default values + optional fields
- ✅ Rich metadata (`description` → sent to LLM as instruction)
- ✅ IDE autocomplete + type checking
- ✅ `model_dump()`, `model_dump_json()`, `model_validate()`
- ✅ Nested models, unions, enums, custom validators

---

### 3.2.2 TypedDict Schema (Lightweight)

```python
# typeddict_demo.py
from typing import TypedDict

class Person(TypedDict):
    name: str
    age: int

new_person: Person = {"name": "nitish", "age": "35"}  # Note: age is string!
print(new_person)  # {'name': 'nitish', 'age': '35'} — NO validation!
```

**With Structured Output:**
```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
from typing import TypedDict, Annotated, Optional, Literal

load_dotenv()
model = ChatOpenAI()

class Review(TypedDict):
    key_themes: Annotated[list[str], "Key themes discussed in the review"]
    summary: Annotated[str, "Brief summary of the review"]
    sentiment: Annotated[Literal["pos", "neg"], "Sentiment: positive or negative"]
    pros: Annotated[Optional[list[str]], "List of pros"]
    cons: Annotated[Optional[list[str]], "List of cons"]
    name: Annotated[Optional[str], "Reviewer name"]

structured_model = model.with_structured_output(Review)

result = structured_model.invoke("...")  # Same review text
print(result["name"])  # Dict access: result["name"]
```

**TypedDict Notes:**
- Uses `Annotated[type, "description"]` for field descriptions (sent to LLM)
- Returns **plain `dict`**, not a Pydantic model
- **No runtime validation** — trusts LLM output
- Lighter weight, no Pydantic dependency
- Good for: simple schemas, rapid prototyping, when validation not critical

---

### 3.2.3 JSON Schema (Dynamic/Config-Driven)

```python
# json_schema.json
{
    "title": "Review",
    "type": "object",
    "properties": {
        "key_themes": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Key themes discussed in the review"
        },
        "summary": {"type": "string", "description": "Brief summary"},
        "sentiment": {
            "type": "string",
            "enum": ["pos", "neg"],
            "description": "Sentiment: positive or negative"
        },
        "pros": {
            "type": ["array", "null"],
            "items": {"type": "string"},
            "description": "List of pros"
        },
        "cons": {
            "type": ["array", "null"],
            "items": {"type": "string"},
            "description": "List of cons"
        },
        "name": {"type": ["string", "null"], "description": "Reviewer name"}
    },
    "required": ["key_themes", "summary", "sentiment"]
}
```

```python
# with_structured_output_json.py
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI()

import json
with open("json_schema.json") as f:
    json_schema = json.load(f)

structured_model = model.with_structured_output(json_schema)

result = structured_model.invoke("...")  # Same review
print(result)  # Plain dict
```

**JSON Schema Advantages:**
- ✅ Language-agnostic (can share with non-Python services)
- ✅ Dynamic — load from file/database/config at runtime
- ✅ Standard format — compatible with OpenAPI, database schemas
- ✅ No Python class definition needed
- ✅ Used by OpenAI Function Calling natively

---

## 3.3 `with_structured_output()` — The Modern API

### Signature
```python
structured_model = model.with_structured_output(
    schema,                    # Pydantic class | TypedDict | JSON Schema dict
    method: str = "function_calling",  # "function_calling" | "json_mode" | "tool_calling"
    include_raw: bool = False,         # If True, returns (raw_message, parsed_object)
    strict: bool = True                # Strict schema enforcement (function_calling only)
)
```

### Method Comparison

| Method | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **function_calling** (default) | Uses OpenAI/Anthropic/Gemini native function calling | Most reliable, strict schema, parallel calls | Requires model supporting tools |
| **json_mode** | Uses `response_format={"type": "json_object"}` | Works on older models | No schema enforcement, may hallucinate keys |
| **tool_calling** | Similar to function_calling, newer API | Unified with tool calling | Same as function_calling |

### Provider Support Matrix

| Provider | Models Supporting Structured Output |
|----------|-------------------------------------|
| **OpenAI** | GPT-4o, GPT-4o-mini, GPT-4-turbo, GPT-3.5-turbo (1106+) |
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Haiku/Opus, Claude 3.7+ |
| **Google** | Gemini 1.5 Pro, Gemini 1.5 Flash, Gemini 1.0 Pro |
| **Open Source** | Via `ChatHuggingFace` — limited support, best with Pydantic |

### Open-Source Model Example
```python
from dotenv import load_dotenv
from pydantic import BaseModel, Field
from typing import Optional, Literal
from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint

load_dotenv()

llm = HuggingFaceEndpoint(
    repo_id="TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    task="text-generation"
)
model = ChatHuggingFace(llm=llm)

class Review(BaseModel):
    key_themes: list[str] = Field(description="Key themes...")
    summary: str = Field(description="Brief summary")
    sentiment: Literal["pos", "neg"] = Field(description="Sentiment")
    # ... other fields

structured_model = model.with_structured_output(Review)
result = structured_model.invoke("...")
```

> **Note:** Open-source models via HF Inference API have **limited structured output reliability**. For production local models, use **Ollama with `format="json"`** or **vLLM with guided decoding**.

---

## 3.4 Output Parsers — The Classic Approach

Before `with_structured_output()` (v0.1 era), output parsers were the primary way to get structured data. **Still useful for:**
- Models not supporting function calling
- Custom parsing logic
- Streaming parsing (parse partial output)
- Legacy code migration

### 3.4.1 StrOutputParser — Extract String Content

```python
# stroutputparser1.py — LCEL Chain with StrOutputParser
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()
model = ChatOpenAI()

template1 = PromptTemplate(
    template="Write a detailed report on {topic}",
    input_variables=["topic"]
)

template2 = PromptTemplate(
    template="Write a 5 line summary on the following text.\n{text}",
    input_variables=["text"]
)

parser = StrOutputParser()

# Chain: prompt → model → string → prompt → model → string
chain = template1 | model | parser | template2 | model | parser

result = chain.invoke({"topic": "black hole"})
print(result)  # Direct string, not AIMessage
```

**StrOutputParser simply does:** `lambda x: x.content` (extracts `.content` from AIMessage)

---

### 3.4.2 JsonOutputParser — Parse JSON from Text

```python
# jsonoutputparser.py
from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint
from dotenv import load_dotenv
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import JsonOutputParser

load_dotenv()

llm = HuggingFaceEndpoint(repo_id="google/gemma-2-2b-it", task="text-generation")
model = ChatHuggingFace(llm=llm)

parser = JsonOutputParser()

template = PromptTemplate(
    template="Give me 5 facts about {topic} \n {format_instruction}",
    input_variables=["topic"],
    partial_variables={"format_instruction": parser.get_format_instructions()}
)

chain = template | model | parser

result = chain.invoke({"topic": "black hole"})
# result = {"fact_1": "...", "fact_2": "...", ...}  (dict)
print(result)
```

**Key Points:**
- `parser.get_format_instructions()` → injects formatting instructions into prompt
- LLM must output valid JSON (fragile with smaller models)
- No schema validation — just JSON parsing

---

### 3.4.3 StructuredOutputParser — Schema via ResponseSchema (Legacy)

```python
# structuredoutputparser.py
from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint
from dotenv import load_dotenv
from langchain_core.prompts import PromptTemplate
from langchain.output_parsers import StructuredOutputParser, ResponseSchema

load_dotenv()

llm = HuggingFaceEndpoint(repo_id="google/gemma-2-2b-it", task="text-generation")
model = ChatHuggingFace(llm=llm)

schema = [
    ResponseSchema(name="fact_1", description="Fact 1 about the topic"),
    ResponseSchema(name="fact_2", description="Fact 2 about the topic"),
    ResponseSchema(name="fact_3", description="Fact 3 about the topic"),
]

parser = StructuredOutputParser.from_response_schemas(schema)

template = PromptTemplate(
    template="Give 3 facts about {topic} \n {format_instruction}",
    input_variables=["topic"],
    partial_variables={"format_instruction": parser.get_format_instructions()}
)

chain = template | model | parser

result = chain.invoke({"topic": "black hole"})
# result = {"fact_1": "...", "fact_2": "...", "fact_3": "..."}
print(result)
```

**ResponseSchema → Format Instructions:**
```
The output should be a markdown code snippet formatted in the following schema:
```json
{
    "fact_1": "string",  // Fact 1 about the topic
    "fact_2": "string",  // Fact 2 about the topic
    "fact_3": "string"   // Fact 3 about the topic
}
```

---

### 3.4.4 PydanticOutputParser — Pydantic Validation (Legacy)

```python
# pydanticoutputparser.py
from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint
from dotenv import load_dotenv
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

load_dotenv()

llm = HuggingFaceEndpoint(repo_id="google/gemma-2-2b-it", task="text-generation")
model = ChatHuggingFace(llm=llm)

class Person(BaseModel):
    name: str = Field(description="Name of the person")
    age: int = Field(gt=18, description="Age of the person")
    city: str = Field(description="City the person belongs to")

parser = PydanticOutputParser(pydantic_object=Person)

template = PromptTemplate(
    template="Generate the name, age and city of a fictional {place} person \n {format_instruction}",
    input_variables=["place"],
    partial_variables={"format_instruction": parser.get_format_instructions()}
)

chain = template | model | parser

final_result = chain.invoke({"place": "sri lankan"})
# final_result = Person(name="...", age=25, city="...")
print(final_result)
print(final_result.name)  # Attribute access
```

---

## 3.5 Comparison: `with_structured_output()` vs Output Parsers

| Aspect | `with_structured_output()` | Output Parsers |
|--------|---------------------------|----------------|
| **Schema Enforcement** | Native (function calling) | Prompt-based (fragile) |
| **Validation** | Automatic (Pydantic) | Manual (PydanticOutputParser only) |
| **Reliability** | High (model-native) | Medium (depends on model size) |
| **Streaming Support** | ✅ Native | ✅ Via `stream()` + parser |
| **Model Compatibility** | Requires tool-calling models | Works on any model |
| **Code Complexity** | One line | Prompt + parser + format_instructions |
| **Error Handling** | Built-in retries | Custom retry logic needed |
| **Performance** | Optimized | Extra prompt tokens |

### When to Use Each

```
USE with_structured_output() WHEN:
  ✅ Model supports function/tool calling (GPT-4o, Claude 3.5, Gemini 1.5)
  ✅ Production reliability matters
  ✅ Need validation, coercion, defaults
  ✅ Want cleanest code (1 line)

USE Output Parsers WHEN:
  ✅ Model doesn't support tools (small open-source, older models)
  ✅ Need custom parsing logic (regex, partial parsing)
  ✅ Streaming partial JSON (progressive UI updates)
  ✅ Legacy codebase migration
```

---

## 3.6 Advanced Patterns

### 3.6.1 Include Raw Response
```python
structured_model = model.with_structured_output(Review, include_raw=True)

raw_message, parsed = structured_model.invoke("...")
# raw_message = AIMessage(content='...', tool_calls=[...])
# parsed = Review(...)
```

### 3.6.2 Streaming Structured Output
```python
# Stream tokens, but get final parsed object
async for chunk in structured_model.astream("..."):
    # Chunks are partial parsed objects (Pydantic partial)
    pass

# Or stream raw then parse at end
async for chunk in model.astream("..."):
    print(chunk.content, end="")
```

### 3.6.3 Fallback Chain (Parser → Structured Output)
```python
from langchain_core.runnables import RunnableWithFallbacks

# Try structured output first, fall back to parser
primary = model.with_structured_output(Review)
fallback = template | model | JsonOutputParser()

chain = primary.with_fallbacks([fallback])
result = chain.invoke("...")
```

### 3.6.4 Nested/Recursive Schemas
```python
from pydantic import BaseModel, Field
from typing import List, Optional

class Author(BaseModel):
    name: str
    email: str

class Comment(BaseModel):
    author: Author
    text: str
    replies: List["Comment"] = Field(default_factory=list)

class BlogPost(BaseModel):
    title: str
    author: Author
    content: str
    comments: List[Comment] = []

# Works with with_structured_output()
structured_model = model.with_structured_output(BlogPost)
```

### 3.6.5 Enum with Descriptions
```python
from enum import Enum

class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

class Review(BaseModel):
    sentiment: Sentiment = Field(description="Overall sentiment")
    # Or use Literal for simpler:
    # sentiment: Literal["positive", "negative", "neutral"]
```

---

## 3.7 Real-World Use Cases

### Use Case 1: API Integration (Function Calling)
```python
class SearchQuery(BaseModel):
    query: str = Field(description="Search query for web search")
    num_results: int = Field(default=5, ge=1, le=20)

structured_model = model.with_structured_output(SearchQuery)
result = structured_model.invoke("Find recent papers on transformer architecture")
# result.query → "transformer architecture recent papers 2024"
# result.num_results → 5
# Pass directly to search API
```

### Use Case 2: Database Insertion
```python
class UserRecord(BaseModel):
    username: str = Field(pattern=r"^[a-zA-Z0-9_]+$")
    email: str = Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")
    age: int = Field(ge=13, le=120)
    preferences: dict[str, str] = Field(default_factory=dict)

structured_model = model.with_structured_output(UserRecord)
user_data = structured_model.invoke("Create user profile for john_doe, john@example.com, 28, likes python and hiking")

# Direct DB insert (validated!)
db.users.insert_one(user_data.model_dump())
```

### Use Case 3: Agent Tool Arguments
```python
class WeatherToolArgs(BaseModel):
    city: str
    date: str = Field(description="YYYY-MM-DD format")
    units: Literal["celsius", "fahrenheit"] = "celsius"

# Agent uses this schema for tool calling
@tool(args_schema=WeatherToolArgs)
def get_weather(city: str, date: str, units: str) -> dict:
    ...
```

### Use Case 4: Evaluation/Testing
```python
class Evaluation(BaseModel):
    score: int = Field(ge=1, le=10)
    reasoning: str
    passes: bool

evaluator = model.with_structured_output(Evaluation)
eval_result = evaluator.invoke(f"""
Evaluate this response: {response}
Criteria: {criteria}
""")
assert eval_result.passes == True
```

---

## 3.8 Integration with LCEL & Chains

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Summary(BaseModel):
    key_points: list[str]
    tldr: str
    word_count: int

# Chain: prompt → structured model → dict
prompt = ChatPromptTemplate.from_template(
    "Summarize this text in structured format:\n{text}"
)

structured_model = model.with_structured_output(Summary)

chain = prompt | structured_model

result = chain.invoke({"text": long_article})
# result = Summary(key_points=[...], tldr="...", word_count=150)
```

---

## 3.9 DeepLearning.AI Course Connections

### "Functions, Tools & Agents with LangChain" (Harrison Chase)

| Lesson | Chapter 3 Relevance |
|--------|---------------------|
| **OpenAI Function Calling** | Foundation of `with_structured_output()` — function calling = structured output |
| **Function Calling in LangChain** | `bind_tools()` + `with_structured_output()` for tool arguments |
| **Tagging and Extraction** | Direct application: classification → structured output |
| **Tools and Routing** | Tool schemas = Pydantic models = structured output |

### "Long-Term Agentic Memory with LangGraph"

| Memory Type | Structured Output Role |
|-------------|------------------------|
| **Semantic** (facts) | Extract facts → Pydantic → store in vector DB |
| **Episodic** (experiences) | Summarize interactions → structured → memory store |
| **Procedural** (rules) | Evolve system prompt → structured instructions |

---

## 3.10 Production Checklist

### Schema Design
- [ ] Use **Pydantic** for production (validation, coercion, docs)
- [ ] Add `description` to **every field** — guides LLM
- [ ] Use `Literal`/`Enum` for categorical fields
- [ ] Set `Optional` with `default=None` for truly optional fields
- [ ] Add constraints: `gt`, `lt`, `min_length`, `pattern`
- [ ] Test schema independently: `Model(**test_data)`

### Model Selection
- [ ] Use **GPT-4o-mini** for cost/performance balance
- [ ] Use **GPT-4o** for complex reasoning + structure
- [ ] Avoid JSON mode — prefer function calling
- [ ] For local: Ollama `format="json"` or vLLM guided decoding

### Error Handling
- [ ] Wrap in `RunnableWithFallbacks` for parser fallback
- [ ] Log raw output on parse failure
- [ ] Implement retry with exponential backoff
- [ ] Validate output before downstream use

### Observability
- [ ] LangSmith traces show structured output automatically
- [ ] Log schema version with each call
- [ ] Monitor parse failure rate
- [ ] Track token usage (structured output uses more tokens)

---

## 3.11 Quick Reference: Key Classes

| Category | Class/Function | Package | Purpose |
|----------|----------------|---------|---------|
| **Structured Output** | `model.with_structured_output()` | `langchain_core.language_models` | Modern structured output (v0.2+) |
| **Pydantic** | `BaseModel`, `Field` | `pydantic` | Schema definition + validation |
| **TypedDict** | `TypedDict`, `Annotated` | `typing` | Lightweight schema |
| **JSON Schema** | `dict` | builtin | Dynamic/config-driven schema |
| **StrOutputParser** | `StrOutputParser` | `langchain_core.output_parsers` | Extract `.content` from AIMessage |
| **JsonOutputParser** | `JsonOutputParser` | `langchain_core.output_parsers` | Parse JSON + inject format instructions |
| **StructuredOutputParser** | `StructuredOutputParser`, `ResponseSchema` | `langchain.output_parsers` | Legacy schema-based parsing |
| **PydanticOutputParser** | `PydanticOutputParser` | `langchain_core.output_parsers` | Legacy Pydantic parsing |

---

## 3.12 Summary: Decision Flowchart

```
Need structured output?
│
├─► Model supports function calling? (GPT-4o, Claude 3.5, Gemini 1.5)
│   │
│   ├─ YES → Use model.with_structured_output(PydanticModel)
│   │         ├─ Need validation/coercion? → Pydantic (recommended)
│   │         ├─ Quick prototype? → TypedDict
│   │         └─ Dynamic/config-driven? → JSON Schema
│   │
│   └─ NO (small local model, older model)
│       │
│       ├─ Simple JSON? → JsonOutputParser + format_instructions
│       ├─ Need Pydantic validation? → PydanticOutputParser
│       ├─ Simple key-value? → StructuredOutputParser + ResponseSchema
│       └─ Just extract string? → StrOutputParser
│
└─► Streaming needed?
    ├─ with_structured_output().astream() → partial objects
    └─ model.astream() + custom parser → progressive parsing
```

---

## 3.13 What's Next

| Chapter | Focus | Builds On |
|---------|-------|-----------|
| **4. Chains & LCEL Deep Dive** | `Runnable` protocol, streaming, batch, fallback, branching, custom Runnables | `prompt \| model \| parser` patterns |
| **5. RAG & Indexes** | Loaders, splitters, embeddings, vector stores, retrievers, hybrid search | Structured output for extraction + embedding models |
| **6. Memory** | ConversationBuffer, Summary, LangGraph state, LangMem | Output parsers for memory serialization |
| **7. Agents** | Tool calling, ReAct, LangGraph agents, multi-agent | `with_structured_output()` for tool schemas |

---

## 3.14 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **Structured Output:** https://python.langchain.com/docs/how_to/structured_output/
- **Output Parsers:** https://python.langchain.com/docs/concepts/output_parsers/
- **Function Calling:** https://python.langchain.com/docs/how_to/tool_calling/
- **Pydantic v2:** https://docs.pydantic.dev/

---

*End of Chapter 3 — Structured Output & Output Parsers*