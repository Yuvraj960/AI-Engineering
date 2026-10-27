# Chapter 2: Deep Dive — Models & Prompts

## 2.1 Models Component — Universal LLM Interface

### 2.1.1 Mental Model: Why Models Component Exists

```
Problem: Every LLM provider has different API signatures
┌─────────────────────────────────────────────────────────────────┐
│  OpenAI                    Anthropic (Claude)                   │
│  client.chat.completions   client.messages.create               │
│  .create(model="gpt-4o",   (model="claude-3-5-sonnet",          │
│   messages=[...])          max_tokens=1024,                     │
│                            messages=[...])                      │
│                                                                 │
│  Google (Gemini)           Cohere, Mistral, etc.                │
│  genai.generate_content()  All different!                       │
└─────────────────────────────────────────────────────────────────┘

LangChain Solution: Unified Interface (BaseChatModel / BaseLLM)
```

**Key Insight:** The Models component is **just an interface layer**. It standardizes:
- Invocation: `.invoke()`, `.stream()`, `.batch()`, `.abatch()`
- Parameters: `temperature`, `max_tokens`, `model_name`
- Output: `AIMessage` / `BaseMessage` objects
- Streaming, async, batch — all uniform

---

### 2.1.2 Two Fundamental Model Types

| Type | Input | Output | Primary Use Case | LangChain Class |
|------|-------|--------|------------------|-----------------|
| **Language Models (LLMs / Chat Models)** | Text / Messages | Text / Messages | Chatbots, agents, generation, reasoning, code | `ChatOpenAI`, `ChatAnthropic`, `ChatGoogleGenerativeAI`, `ChatHuggingFace` |
| **Embedding Models** | Text | Vector (`List[float]`) | Semantic search, RAG retrieval, clustering, classification | `OpenAIEmbeddings`, `HuggingFaceEmbeddings`, `CohereEmbeddings` |

> **Critical Distinction:**
> - **LLM (BaseLLM):** String in → String out. No conversation memory, no roles. Legacy. Use for: text generation, summarization, translation, code gen.
> - **Chat Model (BaseChatModel):** Messages in → Message out. Supports **System/Human/AI roles**, **conversation history**, **tool calling**. **Modern default** — industry shifting here.

---

### 2.1.3 Closed-Source Models (API-Based)

#### OpenAI (GPT Family)
```python
# 1.LLMs/1_llm_demo.py — Legacy LLM (completion endpoint)
from langchain_openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
llm = OpenAI(model="gpt-3.5-turbo-instruct")
result = llm.invoke("What is the capital of India")
print(result)  # String output

# 2.ChatModels/1_chatmodel_openai.py — Modern Chat Model (RECOMMENDED)
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI(
    model="gpt-4o",           # or gpt-4o-mini, gpt-4-turbo
    temperature=0.7,          # 0=deterministic, 2=creative
    max_completion_tokens=500 # Output token limit
)
result = model.invoke("Write a 5-line poem on cricket")
print(result.content)  # AIMessage.content → string
```

**Key Parameters:**
| Parameter | Range | Effect | Recommended For |
|-----------|-------|--------|-----------------|
| `temperature` | 0.0 – 2.0 | Randomness/creativity | 0–0.3: code, math, facts; 0.5–0.7: general QA; 0.9–1.2: writing, stories; 1.5+: brainstorming |
| `max_completion_tokens` | int | Hard output limit | Cost control, prevent runaway |
| `model` | str | Model variant | `gpt-4o` (best), `gpt-4o-mini` (cheap/fast), `gpt-4-turbo` |
| `streaming` | bool | Token-by-token output | UX for chat interfaces |
| `seed` | int | Reproducibility | Testing, deterministic outputs |

#### Anthropic (Claude)
```python
# 2.ChatModels/2_chatmodel_anthropic.py
from langchain_anthropic import ChatAnthropic
from dotenv import load_dotenv

load_dotenv()
model = ChatAnthropic(
    model="claude-3-5-sonnet-20241022",  # or claude-3-haiku, claude-3-opus
    temperature=0,
    max_tokens=1024
)
result = model.invoke("What is the capital of India?")
print(result.content)
```

#### Google (Gemini)
```python
# 2.ChatModels/3_chatmodel_google.py
from langchain_google_genai import ChatGoogleGenerativeAI
from dotenv import load_dotenv

load_dotenv()
model = ChatGoogleGenerativeAI(
    model="gemini-1.5-pro",  # or gemini-1.5-flash, gemini-1.0-pro
    temperature=0.7
)
result = model.invoke("What is the capital of India?")
print(result.content)
```

**Provider Switching = 1-2 Line Changes:**
```python
# Same code structure, different import + class
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_google_genai import ChatGoogleGenerativeAI

# Swap provider in ONE line:
model = ChatOpenAI(model="gpt-4o")
model = ChatAnthropic(model="claude-3-5-sonnet-20241022")
model = ChatGoogleGenerativeAI(model="gemini-1.5-pro")

# Everything else identical:
result = model.invoke("Hello")
print(result.content)
```

---

### 2.1.4 Open-Source Models (Local / Self-Hosted)

#### Why Open-Source?
| Advantage | Description |
|-----------|-------------|
| **Zero API Cost** | Run on own hardware (after GPU investment) |
| **Data Privacy** | Confidential docs never leave your server |
| **Customization** | Fine-tune on domain data (LoRA, QLoRA) |
| **No Rate Limits** | Your hardware, your throughput |
| **Offline Capability** | Air-gapped deployments |

| Disadvantage | Mitigation |
|--------------|------------|
| Hardware Requirements | Quantization (4-bit, 8-bit), smaller models (7B, 1.5B) |
| Setup Complexity | `llama.cpp`, `Ollama`, `vLLM`, `TGI` simplify |
| Less Refined (RLHF) | Community fine-tunes (Zephyr, OpenHermes, NeuralHermes) |
| Limited Multimodal | Improving (LLaVA, Qwen-VL, Phi-3-Vision) |

#### Option A: Hugging Face Inference API (Cloud, Free Tier)
```python
# 2.ChatModels/4_chatmodel_hf_api.py
from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint
from dotenv import load_dotenv

load_dotenv()  # Requires HUGGINGFACEHUB_API_TOKEN

llm = HuggingFaceEndpoint(
    repo_id="TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    task="text-generation",
    max_new_tokens=512,
    temperature=0.7,
    huggingfacehub_api_token="your_token"  # or from .env
)

model = ChatHuggingFace(llm=llm)
result = model.invoke("What is the capital of India?")
print(result.content)
```

**Popular HF Models for Chat:**
| Model | Size | Notes |
|-------|------|-------|
| `TinyLlama/TinyLlama-1.1B-Chat-v1.0` | 1.1B | Tiny, fast, good for testing |
| `mistralai/Mistral-7B-Instruct-v0.3` | 7B | Strong general purpose |
| `meta-llama/Meta-Llama-3-8B-Instruct` | 8B | Requires HF access request |
| `microsoft/Phi-3-mini-4k-instruct` | 3.8B | Excellent quality/size ratio |
| `Qwen/Qwen2.5-7B-Instruct` | 7B | Strong multilingual, coding |

#### Option B: Local Execution (Ollama / llama.cpp / vLLM)
```python
# 2.ChatModels/5_chatmodel_hf_local.py (conceptual — requires local setup)
from langchain_huggingface import ChatHuggingFace, HuggingFacePipeline
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline
import torch

model_id = "microsoft/Phi-3-mini-4k-instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.float16,
    device_map="auto",
    load_in_4bit=True  # Requires bitsandbytes
)
pipe = pipeline("text-generation", model=model, tokenizer=tokenizer, max_new_tokens=512)
llm = HuggingFacePipeline(pipeline=pipe)
chat_model = ChatHuggingFace(llm=llm)

result = chat_model.invoke("What is the capital of India?")
```

> **Production Recommendation:** Use **Ollama** (easiest local), **vLLM** (high-throughput serving), or **TGI** (Hugging Face's production server) rather than raw transformers pipeline.

**Ollama Integration (Easiest Local):**
```python
# pip install langchain-ollama
from langchain_ollama import ChatOllama

model = ChatOllama(model="llama3.1:8b", temperature=0.7)  # Runs locally via Ollama
result = model.invoke("What is the capital of India?")
```

---

### 2.1.5 Embedding Models — Text to Vectors

**Purpose:** Convert text → numerical vectors for semantic similarity, RAG retrieval, clustering.

```python
# 3.EmbeddingModels/1_embedding_openai_query.py
from langchain_openai import OpenAIEmbeddings
from dotenv import load_dotenv

load_dotenv()
embedding = OpenAIEmbeddings(
    model="text-embedding-3-large",  # or text-embedding-3-small, text-embedding-ada-002
    dimensions=1536  # Optional: truncate dimensions (only for v3 models)
)

# Single query embedding
vector = embedding.embed_query("Delhi is the capital of India")
print(f"Dimension: {len(vector)}")  # 1536 (or 3072 for large)
print(vector[:5])  # First 5 dimensions

# 3.EmbeddingModels/2_embedding_openai_docs.py — Batch documents
documents = [
    "Delhi is the capital of India",
    "Kolkata is the capital of West Bengal",
    "Paris is the capital of France"
]
vectors = embedding.embed_documents(documents)
print(f"Batch shape: {len(vectors)} x {len(vectors[0])}")
```

**Open-Source Embeddings (Local):**
```python
# 3.EmbeddingModels/3_embedding_hf_local.py
from langchain_huggingface import HuggingFaceEmbeddings

# sentence-transformers models — optimized for embeddings
embedding = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",  # 384 dim, fast
    # model_name="sentence-transformers/all-mpnet-base-v2",  # 768 dim, better quality
    # model_name="BAAI/bge-large-en-v1.5",  # 1024 dim, SOTA open
    model_kwargs={"device": "cuda"},  # or "cpu"
    encode_kwargs={"normalize_embeddings": True}
)

documents = ["Delhi is the capital of India", "Kolkata is the capital of West Bengal"]
vectors = embedding.embed_documents(documents)
```

**Embedding Model Selection Guide:**
| Model | Dim | Speed | Quality | Best For |
|-------|-----|-------|---------|----------|
| `text-embedding-3-large` | 3072 | Medium | Highest (OpenAI) | Production RAG, high accuracy |
| `text-embedding-3-small` | 1536 | Fast | High | Cost-sensitive production |
| `text-embedding-ada-002` | 1536 | Fast | Good | Legacy, compatible |
| `all-MiniLM-L6-v2` | 384 | Very Fast | Good | Local, real-time, edge |
| `all-mpnet-base-v2` | 768 | Fast | Better | Local, better quality |
| `BAAI/bge-large-en-v1.5` | 1024 | Medium | SOTA Open | Local, highest quality |
| `nomic-embed-text-v1.5` | 768 | Fast | Excellent | Local, long context (8k) |

---

### 2.1.6 Practical Application: Document Similarity Search

```python
# 3.EmbeddingModels/4_document_similarity.py
from langchain_openai import OpenAIEmbeddings
from dotenv import load_dotenv
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

load_dotenv()
embedding = OpenAIEmbeddings(model="text-embedding-3-large", dimensions=300)

documents = [
    "Virat Kohli is an Indian cricketer known for his aggressive batting and leadership.",
    "MS Dhoni is a former Indian captain famous for his calm demeanor and finishing skills.",
    "Sachin Tendulkar, also known as the 'God of Cricket', holds many batting records.",
    "Rohit Sharma is known for his elegant batting and record-breaking double centuries.",
    "Jasprit Bumrah is an Indian fast bowler known for his unorthodox action and yorkers."
]

query = "tell me about bumrah"

doc_embeddings = embedding.embed_documents(documents)
query_embedding = embedding.embed_query(query)

# Cosine similarity
scores = cosine_similarity([query_embedding], doc_embeddings)[0]
best_idx, best_score = max(enumerate(scores), key=lambda x: x[1])

print(f"Query: {query}")
print(f"Best match: {documents[best_idx]}")
print(f"Similarity: {best_score:.4f}")
```

**Output:**
```
Query: tell me about bumrah
Best match: Jasprit Bumrah is an Indian fast bowler known for his unorthodox action and yorkers.
Similarity: 0.8234
```

> **Production Note:** In real RAG systems, use **vector databases** (Chroma, FAISS, Pinecone, Qdrant, Weaviate) instead of in-memory cosine similarity. They handle indexing, filtering, persistence, and scale.

---

### 2.1.7 Model Invocation Patterns (LCEL / Runnable Interface)

All models implement the **Runnable Protocol** — uniform interface:

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

model = ChatOpenAI(model="gpt-4o-mini")

# 1. Single invocation (sync)
result = model.invoke("What is 2+2?")
print(result.content)

# 2. Streaming (token by token)
for chunk in model.stream("Write a haiku about coding"):
    print(chunk.content, end="", flush=True)
print()

# 3. Batch (parallel multiple inputs)
results = model.batch([
    "What is the capital of France?",
    "What is the capital of Japan?",
    "What is the capital of Brazil?"
])
for r in results:
    print(r.content)

# 4. Async (non-blocking)
import asyncio
async def main():
    results = await model.abatch([
        "Hello in French",
        "Hello in Spanish",
        "Hello in Japanese"
    ])
    for r in results:
        print(r.content)
asyncio.run(main())

# 5. With messages (chat format)
messages = [
    SystemMessage(content="You are a helpful assistant"),
    HumanMessage(content="What is LangChain?")
]
result = model.invoke(messages)
```

---

### 2.1.8 Advanced Model Features (v0.2+)

#### Structured Output (Pydantic / JSON Schema)
```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
from typing import Literal

class TicketClassification(BaseModel):
    category: Literal["billing", "technical", "general"] = Field(description="Ticket category")
    urgency: Literal["low", "medium", "high"] = Field(description="Urgency level")
    summary: str = Field(description="One-sentence summary")

model = ChatOpenAI(model="gpt-4o-mini")
structured_model = model.with_structured_output(TicketClassification)

result = structured_model.invoke("""
User: I've been charged twice for my subscription this month!
I need this fixed immediately as it's affecting my budget.
""")

print(result.category)    # "billing"
print(result.urgency)     # "high"
print(result.summary)     # "User reports duplicate subscription charge requiring urgent fix"
```

#### Tool Binding (Function Calling)
```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    # In reality, call weather API
    return f"Weather in {city}: 22°C, Sunny"

@tool
def calculate(expression: str) -> float:
    """Evaluate a mathematical expression."""
    return eval(expression)  # Note: unsafe in production!

model = ChatOpenAI(model="gpt-4o-mini")
model_with_tools = model.bind_tools([get_weather, calculate])

# Model decides which tool to call
result = model_with_tools.invoke("What's 25 * 3 + weather in Delhi?")
print(result.tool_calls)
# [{'name': 'calculate', 'args': {'expression': '25 * 3'}, 'id': '...'},
#  {'name': 'get_weather', 'args': {'city': 'Delhi'}, 'id': '...'}]
```

#### Multimodal (Vision)
```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
import base64

model = ChatOpenAI(model="gpt-4o")  # Vision-capable model

with open("chart.png", "rb") as f:
    image_data = base64.b64encode(f.read()).decode()

message = HumanMessage(content=[
    {"type": "text", "text": "Analyze this chart and summarize key insights"},
    {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{image_data}"}}
])

result = model.invoke([message])
print(result.content)
```

---

## 2.2 Prompts Component — Structuring Inputs to LLMs

### 2.2.1 Why Prompts Need Abstraction

**Problem with Raw Strings:**
```python
# ❌ Brittle, non-reusable, hard to maintain
prompt = f"Summarize {paper_name} in {style} style with {length} length. Include math and analogies."
result = model.invoke(prompt)
```

**LangChain Solution: PromptTemplate System**
- **Variables** → Separated from template logic
- **Validation** → Catch missing variables at template creation
- **Reusability** → Same template, different inputs
- **Composition** → Chain templates, partial filling
- **Serialization** → Save/load templates as JSON
- **Output Parsing** → Built-in structure enforcement

---

### 2.2.2 Static vs Dynamic Prompts

| Aspect | Static Prompt | Dynamic Prompt (Template) |
|--------|---------------|---------------------------|
| **Definition** | Hardcoded string | Template with placeholders |
| **Flexibility** | None | Runtime variable injection |
| **User Control** | Full prompt = user writes everything | User provides *values only* |
| **Consistency** | Varies wildly | Guaranteed structure |
| **Security** | Prompt injection risk | Controlled variables only |
| **Use Case** | Prototyping, fixed tasks | Production applications |

**Example — Research Paper Summarizer:**

```python
# Static: User types entire prompt (inconsistent, error-prone)
# "Summarize Attention Is All You Need in beginner-friendly style, short, with analogies"

# Dynamic: User selects from dropdowns → Template fills variables
paper_input = "Attention Is All You Need"
style_input = "Beginner-Friendly"
length_input = "Short (1-2 paragraphs)"

# Template ensures consistent output structure every time
```

---

### 2.2.3 PromptTemplate — Single-Turn (String-Based)

**For LLMs (string in, string out) — Legacy but still useful:**
```python
# prompt_template.py
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI()

# Method 1: Detailed constructor
template = PromptTemplate(
    template="Greet this person in 5 languages. The name of the person is {name}",
    input_variables=["name"],
    validate_template=True  # Validates variables exist in template
)

# Method 2: from_template (shorthand)
template = PromptTemplate.from_template(
    "Translate '{text}' to {language}. Provide formal and informal versions."
)

# Fill placeholders → Returns a PromptValue (wraps string + metadata)
prompt_value = template.invoke({"name": "Nitish"})

# Pass to model
result = model.invoke(prompt_value)
print(result.content)

# Chain style (LCEL)
chain = template | model
result = chain.invoke({"name": "Nitish"})
```

**Advanced Template Features:**
```python
from langchain_core.prompts import PromptTemplate

# Partial variables (pre-fill some values)
template = PromptTemplate(
    template="You are a {role}. Answer the question: {question}",
    input_variables=["question"],
    partial_variables={"role": "senior python developer"}
)

# Only need to provide 'question' at runtime
prompt = template.invoke({"question": "How do I optimize a slow query?"})

# Template validation (catches typos)
template = PromptTemplate(
    template="Hello {nam}",  # Typo: 'nam' vs 'name'
    input_variables=["name"],
    validate_template=True  # Raises ValueError at creation!
)

# Save/Load templates (version control, sharing)
template.save("greeting_prompt.json")
loaded_template = PromptTemplate.load("greeting_prompt.json")
```

---

### 2.2.4 ChatPromptTemplate — Multi-Turn (Message-Based)

**For Chat Models (messages in, message out) — MODERN STANDARD:**
```python
# chat_prompt_template.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# List of (role, template) tuples
chat_template = ChatPromptTemplate([
    ("system", "You are a helpful {domain} expert"),
    ("human", "Explain in simple terms, what is {topic}")
])

# Fill variables → Returns ChatPromptValue (list of messages)
prompt = chat_template.invoke({"domain": "cricket", "topic": "Dusra"})
print(prompt.to_messages())
# [SystemMessage(content="You are a helpful cricket expert"),
#  HumanMessage(content="Explain in simple terms, what is Dusra")]

# Direct to model
result = model.invoke(prompt)
print(result.content)

# LCEL Chain
chain = chat_template | model
result = chain.invoke({"domain": "python", "topic": "decorators"})
```

**Message Roles:**
| Role | Purpose | Typical Content |
|------|---------|-----------------|
| `system` | Instructions, persona, rules | "You are a helpful coding assistant. Always explain with examples." |
| `human` | User input, questions | "How do I fix this bug?" |
| `ai` | Model responses (for few-shot/history) | "Here's the fix: ..." |
| `tool` | Tool results (for agents) | `{"temperature": 22, "unit": "celsius"}` |

---

### 2.2.5 Messages & Chat History (Manual Construction)

```python
# messages.py
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI()

# Build conversation manually
messages = [
    SystemMessage(content="You are a helpful assistant"),
    HumanMessage(content="Tell me about LangChain")
]

result = model.invoke(messages)

# Append AI response for next turn
messages.append(AIMessage(content=result.content))

# Continue conversation
messages.append(HumanMessage(content="What are its main components?"))
result = model.invoke(messages)
print(result.content)
```

**Simple Chatbot Loop:**
```python
# chatbot.py
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI()

chat_history = [SystemMessage(content="You are a helpful AI assistant")]

while True:
    user_input = input("You: ")
    if user_input.lower() == "exit":
        break
    
    chat_history.append(HumanMessage(content=user_input))
    result = model.invoke(chat_history)
    chat_history.append(AIMessage(content=result.content))
    
    print("AI:", result.content)

print("Full history:", chat_history)
```

---

### 2.2.6 Few-Shot Prompting (In-Context Learning)

```python
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# Examples for few-shot
examples = [
    {"ticket": "My bill is wrong!", "category": "Billing"},
    {"ticket": "App crashes on login", "category": "Technical"},
    {"ticket": "What are your hours?", "category": "General"},
    {"ticket": "Password reset not working", "category": "Technical"},
    {"ticket": "Refund for canceled order", "category": "Billing"},
]

# Format for each example
example_prompt = PromptTemplate(
    input_variables=["ticket", "category"],
    template="Ticket: {ticket}\nCategory: {category}"
)

# Few-shot template
few_shot_template = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="Classify the support ticket into: Billing, Technical, or General",
    suffix="Ticket: {input}\nCategory:",
    input_variables=["input"],
    example_separator="\n\n"
)

chain = few_shot_template | model
result = chain.invoke({"input": "I want to cancel my subscription"})
print(result.content)  # "Billing"
```

---

### 2.2.7 MessagePlaceholder — Dynamic History Injection

**Critical for conversational apps — injects variable-length history at runtime:**
```python
# message_placeholder.py
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# Template with placeholder for chat history
chat_template = ChatPromptTemplate([
    ("system", "You are a helpful customer support agent"),
    MessagesPlaceholder(variable_name="chat_history"),  # ← Dynamic insertion point
    ("human", "{query}")
])

# Simulated history from database
chat_history = [
    HumanMessage(content="I want to request a refund for my order 12345"),
    AIMessage(content="Your refund request for order 12345 has been initiated. You'll receive it in 3-5 business days.")
]

# At runtime, inject history + current query
prompt = chat_template.invoke({
    "chat_history": chat_history,
    "query": "Where is my refund?"
})

print(prompt.to_messages())
# [SystemMessage(...),
#  HumanMessage("I want to request a refund..."),
#  AIMessage("Your refund request..."),
#  HumanMessage("Where is my refund?")]

result = model.invoke(prompt)
print(result.content)
```

**Real-World Pattern:**
```python
# Production: Load history from DB/Redis → inject via MessagesPlaceholder
def get_chat_prompt(session_id: str):
    history = load_history_from_db(session_id)  # List[BaseMessage]
    
    template = ChatPromptTemplate([
        ("system", "You are a helpful assistant with access to conversation history."),
        MessagesPlaceholder("history"),
        ("human", "{input}")
    ])
    
    return template.invoke({"history": history, "input": user_input})
```

---

### 2.2.8 Prompt Persistence & Versioning

```python
# prompt_generator.py — Save complex templates as JSON
from langchain_core.prompts import PromptTemplate

template = PromptTemplate(
    template="""
Please summarize the research paper titled "{paper_input}" with the following specifications:
Explanation Style: {style_input}
Explanation Length: {length_input}
1. Mathematical Details:
   - Include relevant mathematical equations if present in the paper.
   - Explain the mathematical concepts using simple, intuitive code snippets where applicable.
2. Analogies:
   - Use relatable analogies to simplify complex ideas.
If certain information is not available in the paper, respond with: "Insufficient information available" instead of guessing.
Ensure the summary is clear, accurate, and aligned with the provided style and length.
""",
    input_variables=["paper_input", "style_input", "length_input"],
    validate_template=True
)

# Save to file (version control, team sharing, deployment)
template.save("template.json")

# Load in production / another service
from langchain_core.prompts import load_prompt
loaded_template = load_prompt("template.json")

# Use in Streamlit UI (prompt_ui.py)
import streamlit as st
from langchain_openai import ChatOpenAI

model = ChatOpenAI()
template = load_prompt("template.json")

paper = st.selectbox("Paper", ["Attention Is All You Need", "BERT", "GPT-3", "Diffusion Models"])
style = st.selectbox("Style", ["Beginner-Friendly", "Technical", "Code-Oriented", "Mathematical"])
length = st.selectbox("Length", ["Short", "Medium", "Long"])

if st.button("Summarize"):
    chain = template | model
    result = chain.invoke({
        "paper_input": paper,
        "style_input": style,
        "length_input": length
    })
    st.write(result.content)
```

**Saved Template (template.json):**
```json
{
  "input_variables": ["length_input", "paper_input", "style_input"],
  "template": "\nPlease summarize the research paper titled \"{paper_input}\" with the following specifications:\nExplanation Style: {style_input}  \nExplanation Length: {length_input}  \n1. Mathematical Details:  \n   - Include relevant mathematical equations if present in the paper.  \n   - Explain the mathematical concepts using simple, intuitive code snippets where applicable.  \n2. Analogies:  \n   - Use relatable analogies to simplify complex ideas.  \nIf certain information is not available in the paper, respond with: \"Insufficient information available\" instead of guessing.  \nEnsure the summary is clear, accurate, and aligned with the provided style and length.\n",
  "template_format": "f-string",
  "validate_template": true,
  "_type": "prompt"
}
```

---

### 2.2.9 Prompt Engineering Best Practices

| Technique | Description | LangChain Implementation |
|-----------|-------------|--------------------------|
| **Role Prompting** | Assign persona via system message | `("system", "You are a {role}")` |
| **Few-Shot** | Provide examples in prompt | `FewShotPromptTemplate` |
| **Chain-of-Thought** | "Think step by step" | Add to system/template: "Reason step by step before answering." |
| **Structured Output** | Enforce JSON/Pydantic schema | `model.with_structured_output(PydanticClass)` |
| **Partial Variables** | Pre-fill context (domain, rules) | `partial_variables={"role": "doctor"}` |
| **MessagePlaceholder** | Inject dynamic history | `MessagesPlaceholder("history")` |
| **Template Versioning** | Save/load JSON for CI/CD | `template.save()`, `load_prompt()` |
| **Variable Validation** | Catch errors early | `validate_template=True` |

---

### 2.2.10 Prompt → Model → Output Parser Chain (LCEL)

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

# String output (default)
str_chain = (
    ChatPromptTemplate.from_template("Tell me a joke about {topic}")
    | model
    | StrOutputParser()
)

# JSON output
json_chain = (
    ChatPromptTemplate.from_template("Output JSON: {\"joke\": \"...\", \"topic\": \"{topic}\"}")
    | model
    | JsonOutputParser()
)

# Structured Pydantic output
class Joke(BaseModel):
    setup: str = Field(description="The setup of the joke")
    punchline: str = Field(description="The punchline")
    rating: int = Field(description="Funny rating 1-10", ge=1, le=10)

structured_chain = (
    ChatPromptTemplate.from_template("Tell me a joke about {topic} in structured format")
    | model.with_structured_output(Joke)
)

# Usage
print(str_chain.invoke({"topic": "programming"}))
print(json_chain.invoke({"topic": "AI"}))
print(structured_chain.invoke({"topic": "debugging"}))
```

---

## 2.3 Integration: Models + Prompts → Chains

The power emerges when **Models + Prompts compose via LCEL**:

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

model = ChatOpenAI(model="gpt-4o-mini")

# Prompt with history placeholder
prompt = ChatPromptTemplate([
    ("system", "You are a helpful assistant. Answer concisely."),
    MessagesPlaceholder("history"),
    ("human", "{input}")
])

# Base chain
chain = prompt | model | StrOutputParser()

# Add message history management
store = {}  # In production: Redis, Postgres, etc.

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

conversational_chain = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history"
)

# Use — history auto-managed!
config = {"configurable": {"session_id": "user_123"}}
print(conversational_chain.invoke({"input": "Hi, I'm Alice"}, config=config))
print(conversational_chain.invoke({"input": "What's my name?"}, config=config))  # Remembers!
```

---

## 2.4 Connection to DeepLearning.AI Courses

### From "Functions, Tools & Agents with LangChain" (Harrison Chase)

| Course Lesson | Chapter 2 Relevance |
|---------------|---------------------|
| **OpenAI Function Calling** | `model.bind_tools()`, structured output via `with_structured_output()` |
| **LangChain Expression Language (LCEL)** | All chains above use `|` operator, Runnable protocol |
| **Function Calling in LangChain** | Tool binding, Pydantic schemas, parallel tool calls |
| **Tagging and Extraction** | Few-shot + structured output for classification/extraction |
| **Tools and Routing** | `RunnableBranch`, conditional chains based on model output |
| **Conversational Agent** | `RunnableWithMessageHistory` + tools = full agent |

### From "Long-Term Agentic Memory with LangGraph" (Harrison Chase + Team)

| Concept | Chapter 2 Foundation |
|---------|---------------------|
| **Semantic Memory** (facts) | Embedding models → vector storage → retrieval |
| **Episodic Memory** (experiences) | Chat history via `MessagesPlaceholder` + `RunnableWithMessageHistory` |
| **Procedural Memory** (rules) | System prompts, prompt templates as evolving instructions |
| **LangMem Library** | Builds on embedding models + vector stores covered here |

---

## 2.5 Production Checklist: Models & Prompts

### Models
- [ ] **Provider abstraction** — Code works with OpenAI/Anthropic/Google/local via 1-line swap
- [ ] **Error handling** — Retries, fallbacks, timeouts via `with_fallbacks()`, `with_retry()`
- [ ] **Streaming** — Token-by-token for UX, `astream()` for async
- [ ] **Structured output** — Pydantic schemas for reliable parsing
- [ ] **Tool binding** — `bind_tools()` for agent workflows
- [ ] **Cost control** — `max_completion_tokens`, model selection (mini vs pro)
- [ ] **Observability** — LangSmith tracing auto-enabled on LCEL chains
- [ ] **Local fallback** — Ollama/vLLM for privacy/cost scenarios

### Prompts
- [ ] **No raw f-strings** — All prompts via `PromptTemplate` / `ChatPromptTemplate`
- [ ] **Variable validation** — `validate_template=True` catches typos
- [ ] **Version control** — Templates saved as JSON, committed to git
- [ ] **Partial variables** — System context pre-filled, not repeated
- [ ] **Few-shot examples** — Curated, representative, updated via eval
- [ ] **MessagePlaceholder** — History injected dynamically, not hardcoded
- [ ] **Output parsers** — Structured output for downstream consumption
- [ ] **Prompt testing** — Evaluation harness (LangSmith, custom) for regression

---

## 2.6 Summary: Key Classes Reference

### Models
| Category | Class | Package | Purpose |
|----------|-------|---------|---------|
| **Chat (OpenAI)** | `ChatOpenAI` | `langchain_openai` | GPT-4o, GPT-4o-mini, etc. |
| **Chat (Anthropic)** | `ChatAnthropic` | `langchain_anthropic` | Claude 3.5 Sonnet, Haiku, Opus |
| **Chat (Google)** | `ChatGoogleGenerativeAI` | `langchain_google_genai` | Gemini 1.5 Pro/Flash |
| **Chat (HF API)** | `ChatHuggingFace` + `HuggingFaceEndpoint` | `langchain_huggingface` | Any HF-hosted model |
| **Chat (Local)** | `ChatOllama` / `ChatHuggingFace` + `HuggingFacePipeline` | `langchain_ollama` / `langchain_huggingface` | Local GGUF/transformers |
| **Legacy LLM** | `OpenAI` | `langchain_openai` | Completion endpoint (deprecated path) |
| **Embeddings (OpenAI)** | `OpenAIEmbeddings` | `langchain_openai` | text-embedding-3-large/small |
| **Embeddings (Local)** | `HuggingFaceEmbeddings` | `langchain_huggingface` | sentence-transformers, BGE, Nomic |

### Prompts
| Class | Package | Purpose |
|-------|---------|---------|
| `PromptTemplate` | `langchain_core.prompts` | String templates (LLMs, single-turn) |
| `ChatPromptTemplate` | `langchain_core.prompts` | Message templates (Chat Models, multi-turn) |
| `FewShotPromptTemplate` | `langchain_core.prompts` | In-context learning with examples |
| `MessagesPlaceholder` | `langchain_core.prompts` | Dynamic history injection |
| `load_prompt` | `langchain_core.prompts` | Load saved template from JSON |
| `StrOutputParser` | `langchain_core.output_parsers` | Extract `.content` from AIMessage |
| `JsonOutputParser` | `langchain_core.output_parsers` | Parse JSON from model output |
| `PydanticOutputParser` | `langchain_core.output_parsers` | Parse into Pydantic model (legacy) |
| `RunnableWithMessageHistory` | `langchain_core.runnables` | Auto-manage conversation history |

---

## 2.7 What's Next

| Chapter | Focus | Builds On |
|---------|-------|-----------|
| **3. Chains & LCEL** | Pipeline composition, streaming, batch, fallback, branching, custom Runnables | Models + Prompts → `prompt \| model \| parser` |
| **4. RAG & Indexes** | Loaders, splitters, embeddings, vector stores, retrievers, hybrid search | Embedding models + Prompt templates for RAG |
| **5. Memory** | ConversationBuffer, Summary, LangGraph state, LangMem (semantic/episodic/procedural) | MessagesPlaceholder + ChatMessageHistory |
| **6. Agents** | Tool calling, ReAct, LangGraph agents, multi-agent, human-in-the-loop | Model.bind_tools() + structured output + prompt templates |

---

## 2.8 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### DeepLearning.AI Courses
- **Functions, Tools & Agents:** https://learn.deeplearning.ai/courses/functions-tools-agents-langchain
- **Long-Term Agentic Memory:** https://learn.deeplearning.ai/courses/long-term-agentic-memory-with-langgraph

### Official Documentation
- **LangChain Models:** https://python.langchain.com/docs/concepts/chat_models/
- **LangChain Prompts:** https://python.langchain.com/docs/concepts/prompt_templates/
- **LCEL/Runnables:** https://python.langchain.com/docs/concepts/lcel/
- **Structured Output:** https://python.langchain.com/docs/how_to/structured_output/
- **Tool Calling:** https://python.langchain.com/docs/how_to/tool_calling/

---

*End of Chapter 2 — Models & Prompts Deep Dive*