# Chapter 5: LangChain Runnables — The Universal Interface

## 5.1 The Origin Story: Why Runnables Exist

### The Historical Problem (2022-2023)

When LangChain launched (late 2022), the team built **components** for every LLM application need:
- **LLMs** — `predict(prompt)` 
- **Prompt Templates** — `format(input_dict)`
- **Retrievers** — `get_relevant_documents(query)`
- **Output Parsers** — `parse(output)`
- **Memory** — `load_memory_variables()`, `save_context()`

**The Fatal Flaw:** Each component had a **different interface**. No standardization.

```
LLMChain          →  llm.predict(prompt)
PromptTemplate    →  prompt.format(variables)
Retriever         →  retriever.get_relevant_documents(query)
Parser            →  parser.parse(llm_output)
Memory            →  memory.load_memory_variables({})
```

### The Chain Explosion

To connect these mismatched components, LangChain created **specialized Chain classes** for every use case:
- `LLMChain` — LLM + Prompt
- `RetrievalQA` — Retriever + LLM + Prompt
- `SimpleSequentialChain` — Chain1 → Chain2
- `SequentialChain` — Multi-input/output chains
- `ConversationChain` — LLM + Memory
- `SQLDatabaseChain` — LLM + SQL
- `APIChain` — LLM + API
- `LLMMathChain` — LLM + Calculator
- ... **50+ chain classes**

**Two Disasters:**
1. **Codebase Bloat** — Massive maintenance burden
2. **Learning Cliff** — New users couldn't know which chain for which use case

### The Root Cause

> **Components weren't designed to connect.** They were built independently with different method names (`predict`, `format`, `get_relevant_documents`, `parse`).

The LangChain team realized: **Standardize the interface, eliminate the chains.**

---

## 5.2 The Runnable Protocol — The Universal Interface

### The Insight

> **Every component should be a "unit of work" with a common interface.** Like LEGO blocks — different shapes, same connectors.

### The Runnable Abstract Base Class

```python
# Simplified from langchain_core.runnables.base.Runnable
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, Optional, Iterator, AsyncIterator, List, Any

Input = TypeVar("Input")
Output = TypeVar("Output")

class Runnable(Generic[Input, Output], ABC):
    """A unit of work that can be invoked, batched, streamed, and composed."""
    
    # ============ REQUIRED (must implement) ============
    @abstractmethod
    def invoke(self, input: Input, config: Optional[RunnableConfig] = None) -> Output:
        """Transform a single input into an output."""
        ...
    
    # ============ PROVIDED BY DEFAULT (override for optimization) ============
    def batch(self, inputs: List[Input], config: Optional[RunnableConfig] = None, **kwargs) -> List[Output]:
        """Default: calls invoke() in a thread pool for each input."""
        return [self.invoke(inp, config) for inp in inputs]
    
    def stream(self, input: Input, config: Optional[RunnableConfig] = None, **kwargs) -> Iterator[Output]:
        """Default: calls invoke() and yields the single result."""
        yield self.invoke(input, config)
    
    async def ainvoke(self, input: Input, config: Optional[RunnableConfig] = None, **kwargs) -> Output:
        """Default: runs invoke() in a thread executor."""
        return await asyncio.get_event_loop().run_in_executor(None, self.invoke, input, config)
    
    async def abatch(self, inputs: List[Input], config: Optional[RunnableConfig] = None, **kwargs) -> List[Output]:
        """Default: async version of batch()."""
        ...
    
    async def astream(self, input: Input, config: Optional[RunnableConfig] = None, **kwargs) -> AsyncIterator[Output]:
        """Default: async version of stream()."""
        ...
    
    # ============ COMPOSITION ============
    def __or__(self, other: "Runnable") -> "RunnableSequence":
        """Enable pipe operator: a | b"""
        from langchain_core.runnables import RunnableSequence
        return RunnableSequence(first=self, last=other)
    
    def __ror__(self, other: "Runnable") -> "RunnableSequence":
        return RunnableSequence(first=other, last=self)
```

### The Four Principles (LEGO Analogy)

| Principle | Runnable | LEGO |
|-----------|----------|------|
| **Unit of Work** | Each runnable has a specific task | Each block has a shape/purpose |
| **Common Interface** | All have `invoke()`, `batch()`, `stream()` | All have same connector studs |
| **Composable** | Connect R1→R2→R3, output→input | Snap blocks together |
| **Composite is Runnable** | R1|R2|R3 is itself a Runnable | Built structure is also a block |

---

## 5.3 Building Runnables from Scratch

Nitish builds the entire system from first principles. Here's the complete reconstruction:

### Step 1: The Abstract Base Class

```python
# runnable_base.py
from abc import ABC, abstractmethod
from typing import Any, Optional, List, Iterator, AsyncIterator

class Runnable(ABC):
    """Abstract base class - all components must inherit from this."""
    
    @abstractmethod
    def invoke(self, input: Any, config: Optional[dict] = None) -> Any:
        """Main entry point - process input, return output."""
        pass
    
    # Default implementations (can be overridden)
    def batch(self, inputs: List[Any], config: Optional[dict] = None) -> List[Any]:
        return [self.invoke(inp, config) for inp in inputs]
    
    def stream(self, input: Any, config: Optional[dict] = None) -> Iterator[Any]:
        yield self.invoke(input, config)
    
    def __or__(self, other: "Runnable") -> "RunnableSequence":
        return RunnableSequence([self, other])
    
    def __ror__(self, other: "Runnable") -> "RunnableSequence":
        return RunnableSequence([other, self])
```

### Step 2: FakeLLM — Standardized LLM Component

```python
# fake_llm.py
import random
from runnable_base import Runnable

class FakeLLM(Runnable):
    """Dummy LLM that implements the Runnable interface."""
    
    def __init__(self):
        print("LLM Created")
        self.responses = [
            "Delhi is the capital of India",
            "IPL is a cricket league", 
            "AI stands for Artificial Intelligence"
        ]
    
    def invoke(self, input: Any, config: Optional[dict] = None) -> Any:
        """Standardized method - replaces predict()"""
        # In real LLM: send prompt to API, return response
        return {"response": random.choice(self.responses)}
    
    # DEPRECATED but kept for backward compatibility
    def predict(self, prompt: str) -> dict:
        print("WARNING: predict() is deprecated. Use invoke() instead.")
        return self.invoke(prompt)
```

### Step 3: DummyPromptTemplate — Standardized Prompt Component

```python
# dummy_prompt.py
from runnable_base import Runnable

class DummyPromptTemplate(Runnable):
    """Dummy Prompt Template implementing Runnable."""
    
    def __init__(self, template: str, input_variables: list):
        self.template = template
        self.input_variables = input_variables
        print("Prompt Template Created")
    
    def invoke(self, input: dict, config: Optional[dict] = None) -> str:
        """Standardized method - replaces format()"""
        # Format template with input variables
        return self.template.format(**input)
    
    # DEPRECATED
    def format(self, **kwargs) -> str:
        print("WARNING: format() is deprecated. Use invoke() instead.")
        return self.invoke(kwargs)
```

### Step 4: RunnableConnector (RunnableSequence) — The Chain Builder

```python
# runnable_sequence.py
from runnable_base import Runnable
from typing import List, Any, Optional

class RunnableSequence(Runnable):
    """Chains multiple Runnables together. Output of each → Input of next."""
    
    def __init__(self, runnables: List[Runnable]):
        self.runnables = runnables
        print(f"RunnableSequence created with {len(runnables)} steps")
    
    def invoke(self, input: Any, config: Optional[dict] = None) -> Any:
        """Execute each runnable in sequence, passing output to next."""
        current_input = input
        
        for i, runnable in enumerate(self.runnables):
            print(f"Step {i+1}: {runnable.__class__.__name__}.invoke()")
            current_input = runnable.invoke(current_input, config)
        
        return current_input
    
    def stream(self, input: Any, config: Optional[dict] = None):
        # Real implementation streams through each step
        yield self.invoke(input, config)
```

### Step 5: Using the System

```python
# main.py
from fake_llm import FakeLLM
from dummy_prompt import DummyPromptTemplate
from runnable_sequence import RunnableSequence

# 1. Create components (both are Runnables)
llm = FakeLLM()
prompt = DummyPromptTemplate(
    template="Write a {length} poem about {topic}",
    input_variables=["length", "topic"]
)

# 2. Chain them using pipe operator (or explicit RunnableSequence)
chain = prompt | llm  # Creates RunnableSequence([prompt, llm])

# 3. Invoke
result = chain.invoke({"length": "short", "topic": "India"})
print(result)
# Output: {'response': 'Delhi is the capital of India'} (random)

# 4. Add more steps - String Output Parser
class StrOutputParser(Runnable):
    def invoke(self, input: dict, config: Optional[dict] = None) -> str:
        return input.get("response", "")

# 5. Extended chain: prompt → llm → parser → prompt2 → llm → parser
prompt2 = DummyPromptTemplate(
    template="Explain this poem: {text}",
    input_variables=["text"]
)

chain = prompt | llm | StrOutputParser() | prompt2 | llm | StrOutputParser()
result = chain.invoke({"length": "short", "topic": "AI"})
print(result)
```

---

## 5.4 The Five Core Runnable Primitives

LangChain provides **5 primitive Runnables** that compose into any workflow:

```
┌─────────────────────────────────────────────────────────────────┐
│                    RUNNABLE PRIMITIVES                          │
├──────────────────┬──────────────────┬───────────────────────────┤
│  RunnableSequence│ RunnableParallel │  RunnableBranch           │
│  (Sequential)    │ (Parallel)       │  (Conditional)            │
├──────────────────┼──────────────────┼───────────────────────────┤
│  RunnablePassthrough                    RunnableLambda          │
│  (Identity/Assign)                  │  (Custom Function)        │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4.1 RunnableSequence — Sequential Composition

Here's the GitHub implementation:

```python
# runnable_sequence.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv
from langchain.schema.runnable import RunnableSequence

load_dotenv()

prompt1 = PromptTemplate(template='Write a joke about {topic}', input_variables=['topic'])
model = ChatOpenAI()
parser = StrOutputParser()
prompt2 = PromptTemplate(template='Explain the following joke - {text}', input_variables=['text'])

# Explicit RunnableSequence construction
chain = RunnableSequence(prompt1, model, parser, prompt2, model, parser)

print(chain.invoke({'topic': 'AI'}))
# Step 1: prompt1 → "Write a joke about AI"
# Step 2: model → AIMessage("Why did the AI cross the road?...")
# Step 3: parser → "Why did the AI cross the road?..."
# Step 4: prompt2 → "Explain the following joke - Why did the AI..."
# Step 5: model → AIMessage("The joke plays on...")  
# Step 6: parser → "The joke plays on..."
```

**Key Insight:** `RunnableSequence` is what the `|` operator creates automatically.

---

### 5.4.2 RunnableParallel — Parallel Fan-Out/Fan-In

```python
# runnable_parallel.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv
from langchain.schema.runnable import RunnableSequence, RunnableParallel

load_dotenv()

prompt1 = PromptTemplate(template='Generate a tweet about {topic}', input_variables=['topic'])
prompt2 = PromptTemplate(template='Generate a LinkedIn post about {topic}', input_variables=['topic'])
model = ChatOpenAI()
parser = StrOutputParser()

# Parallel execution: same input → multiple branches simultaneously
parallel_chain = RunnableParallel({
    'tweet': RunnableSequence(prompt1, model, parser),
    'linkedin': RunnableSequence(prompt2, model, parser)
})

result = parallel_chain.invoke({'topic': 'AI'})
print(result['tweet'])
print(result['linkedin'])

# Output: Both generated concurrently (not sequentially!)
# {'tweet': '🚀 AI is transforming...', 'linkedin': 'Excited to share...'}
```

**Execution Timeline:**
```
Sequential (3 calls × 2s each):  6 seconds
Parallel (3 calls concurrent):   2 seconds  ← 3x faster!
```

**Dict Shorthand (Auto-converts to RunnableParallel):**
```python
# These are equivalent:
parallel_chain = RunnableParallel({'tweet': chain1, 'linkedin': chain2})
parallel_chain = {'tweet': chain1, 'linkedin': chain2}  # Dict shorthand
```

---

### 5.4.3 RunnablePassthrough — Identity & Assignment

```python
# runnable_passthrough.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv
from langchain.schema.runnable import RunnableSequence, RunnableParallel, RunnablePassthrough

load_dotenv()

prompt1 = PromptTemplate(template='Write a joke about {topic}', input_variables=['topic'])
model = ChatOpenAI()
parser = StrOutputParser()
prompt2 = PromptTemplate(template='Explain the following joke - {text}', input_variables=['text'])

joke_gen_chain = RunnableSequence(prompt1, model, parser)

# CRITICAL PATTERN: Preserve original input while transforming
parallel_chain = RunnableParallel({
    'joke': RunnablePassthrough(),           # Pass joke through unchanged
    'explanation': RunnableSequence(prompt2, model, parser)  # Transform
})

final_chain = RunnableSequence(joke_gen_chain, parallel_chain)

print(final_chain.invoke({'topic': 'cricket'}))
# Output: {'joke': 'Why did the cricketer...', 'explanation': 'The joke works because...'}
```

#### RunnablePassthrough.assign() — Add Keys While Preserving

```python
from langchain_core.runnables import RunnablePassthrough

# Add computed fields to input dict
chain = RunnablePassthrough.assign(
    word_count=lambda x: len(x["text"].split()),
    upper=lambda x: x["text"].upper(),
    char_count=lambda x: len(x["text"])
)

result = chain.invoke({"text": "Hello world from LangChain"})
# {'text': 'Hello world from LangChain', 'word_count': 5, 'upper': 'HELLO WORLD FROM LANGCHAIN', 'char_count': 30}

# THE RAG PATTERN (most important use case):
rag_chain = (
    {
        "context": retriever | RunnableLambda(format_docs),  # Transform
        "question": RunnablePassthrough()                     # Preserve
    }
    | rag_prompt
    | model
    | StrOutputParser()
)
```

---

### 5.4.4 RunnableLambda — Wrap Any Python Function

```python
# runnable_lambda.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv
from langchain.schema.runnable import RunnableSequence, RunnableLambda, RunnablePassthrough, RunnableParallel

load_dotenv()

def word_count(text: str) -> int:
    """Custom function: count words in text."""
    return len(text.split())

prompt = PromptTemplate(template='Write a joke about {topic}', input_variables=['topic'])
model = ChatOpenAI()
parser = StrOutputParser()

joke_gen_chain = RunnableSequence(prompt, model, parser)

# Wrap custom function as Runnable
parallel_chain = RunnableParallel({
    'joke': RunnablePassthrough(),
    'word_count': RunnableLambda(word_count)  # ← Custom logic in chain
})

final_chain = RunnableSequence(joke_gen_chain, parallel_chain)

result = final_chain.invoke({'topic': 'AI'})
final_result = f"{result['joke']}\nword count - {result['word_count']}"
print(final_result)
```

#### Auto-Wrapping (Implicit RunnableLambda)

```python
# LangChain automatically wraps plain functions in pipe chains!
chain = (
    prompt 
    | model 
    | StrOutputParser() 
    | word_count          # Auto-wrapped to RunnableLambda(word_count)
    | lambda x: f"Words: {x}"  # Also auto-wrapped
)

# For ASYNC streaming, define async functions:
async def async_word_count(text: str) -> int:
    return len(text.split())

chain = prompt | model | StrOutputParser() | async_word_count
result = await chain.ainvoke({"topic": "async"})  # Works with astream/abatch
```

---

### 5.4.5 RunnableBranch — Conditional Routing

```python
# runnable_branch.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv
from langchain.schema.runnable import RunnableSequence, RunnableParallel, RunnablePassthrough, RunnableBranch, RunnableLambda

load_dotenv()

prompt1 = PromptTemplate(template='Write a detailed report on {topic}', input_variables=['topic'])
prompt2 = PromptTemplate(template='Summarize the following text\n{text}', input_variables=['text'])

model = ChatOpenAI()
parser = StrOutputParser()

report_gen_chain = prompt1 | model | parser

# Conditional: if report > 300 words, summarize; else pass through
branch_chain = RunnableBranch(
    (lambda x: len(x.split()) > 300, prompt2 | model | parser),  # Condition, Runnable
    RunnablePassthrough()  # Default (no tuple = default)
)

final_chain = RunnableSequence(report_gen_chain, branch_chain)

print(final_chain.invoke({'topic': 'Russia vs Ukraine'}))
# Long report → gets summarized
# Short report → passes through unchanged
```

**RunnableBranch Signature:**
```python
RunnableBranch(
    (condition1, runnable1),   # Tuple: (predicate, runnable)
    (condition2, runnable2),   # Evaluated in order
    (condition3, runnable3),
    default_runnable           # Last arg without tuple = default
)
```

**Real-World Router Pattern:**
```python
# Classify → Route to specialized chain
classifier = (
    ChatPromptTemplate.from_template(
        "Classify: tech, science, or history? Question: {question}"
    )
    | model
    | StrOutputParser()
    | (lambda x: x.strip().lower())
)

tech_chain = ChatPromptTemplate.from_template("As engineer: {question}") | model
science_chain = ChatPromptTemplate.from_template("As scientist: {question}") | model
history_chain = ChatPromptTemplate.from_template("As historian: {question}") | model

router = RunnableBranch(
    (lambda x: "tech" in x["category"], tech_chain),
    (lambda x: "science" in x["category"], science_chain),
    history_chain  # default
)

full_chain = RunnableParallel(
    category=classifier,
    question=itemgetter("question")
) | router
```

---

## 5.5 RunnableBinding — Configuration Attachment

```python
# Not a primitive but critical pattern
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# Bind tools for function calling
model_with_tools = model.bind(tools=[my_tool_schema])

# Bind config (tags, metadata, callbacks)
model_tagged = model.with_config(tags=["production", "user-123"])

# Bind stop sequences
model_stopped = model.bind(stop=["\n\n", "END"])

# Chain with bound model
chain = prompt | model_with_tools | parser
```

**RunnableBinding wraps a Runnable with pre-configured kwargs/config.**

---

## 5.6 Composition Mechanics: How `|` Works

### The Pipe Operator

```python
# This:
chain = prompt | model | parser

# Is syntactic sugar for:
from langchain_core.runnables import RunnableSequence
chain = RunnableSequence(first=prompt, middle=[model], last=parser)

# Or equivalently:
chain = prompt.__or__(model).__or__(parser)
```

### Coercion Rules (What Can Be Piped)

```python
def coerce_to_runnable(thing):
    if isinstance(thing, Runnable):
        return thing
    elif callable(thing):
        return RunnableLambda(thing)      # Functions → RunnableLambda
    elif isinstance(thing, dict):
        return RunnableParallel(thing)     # Dicts → RunnableParallel
    else:
        raise TypeError(f"Cannot coerce {type(thing)} to Runnable")
```

**This is why you can write:**
```python
chain = (
    prompt                                    # Runnable
    | model                                   # Runnable  
    | StrOutputParser()                       # Runnable
    | word_count                              # function → RunnableLambda
    | {"a": lambda x: x*2, "b": lambda x: x*3}  # dict → RunnableParallel
    | lambda x: x["a"] + x["b"]               # function → RunnableLambda
)
```

---

## 5.7 The Computation Graph (DAG)

Every LCEL chain is a **Directed Acyclic Graph** internally:

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("Tell me about {topic}")
    | ChatOpenAI(model="gpt-4o")
    | StrOutputParser()
)

# Inspect the graph
graph = chain.get_graph()
print(graph.draw_ascii())
# 
# ┌───────────────┐     ┌─────────────┐     ┌──────────────────┐
# │ ChatPromptTem │────▶│ ChatOpenAI  │────▶│ StrOutputParser  │
# │ plate         │     │             │     │                  │
# └───────────────┘     └─────────────┘     └──────────────────┘

# Input/Output schemas (auto-generated Pydantic)
print(chain.input_schema.schema())
# {'properties': {'topic': {'type': 'string'}}, 'required': ['topic']}

print(chain.output_schema.schema())
# {'type': 'string'}
```

---

## 5.8 Streaming Architecture

### How Streaming Propagates Through Chain

```python
# Chain: prompt | model | StrOutputParser()
# When calling chain.stream(input):

# 1. prompt.stream(input) 
#    → yields PromptValue (usually single chunk)

# 2. model.stream(PromptValue) 
#    → yields AIMessageChunk (token by token) ← KEY: model overrides stream()

# 3. StrOutputParser.stream(AIMessageChunk) 
#    → yields string chunks (token text)

# 4. Final chain.stream() yields string chunks to caller
```

### Streaming Capabilities by Component

| Component | Streaming? | Behavior |
|-----------|------------|----------|
| `ChatOpenAI` / `ChatAnthropic` | ✅ Native | Yields tokens as generated |
| `StrOutputParser` | ✅ | Passes token chunks through |
| `JsonOutputParser` | ⚠️ | Buffers until valid JSON, yields complete |
| `PydanticOutputParser` | ⚠️ | Needs full response |
| `with_structured_output()` | ⚠️ | Model-native (function calling) |
| `RunnableLambda` | ❌ Default | Override `astream` for true streaming |
| `RunnableParallel` | ✅ | Streams each branch, merges dicts |
| `RunnableBranch` | ✅ | Streams selected branch |

---

## 5.9 RunnableConfig — The Invisible Thread

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.callbacks import StdOutCallbackHandler

config = RunnableConfig(
    tags=["production", "experiment-v2"],
    metadata={"user_id": "123", "session": "abc"},
    callbacks=[StdOutCallbackHandler()],
    max_concurrency=10,
    run_name="qa_chain",
    configurable={
        "model_name": "gpt-4o",
        "temperature": 0.0
    }
)

# Config threads through ENTIRE pipeline automatically
result = chain.invoke(input, config=config)
```

**Config merges down the chain:**
```python
# Parent config
chain.invoke(input, config={"tags": ["parent"]})
# Child automatically inherits
# Final tags: ["parent", "child"]
```

---

## 5.10 Advanced Patterns

### 5.10.1 Configurable Fields (Runtime Parameter Override)

```python
from langchain_core.runnables import ConfigurableField
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini", temperature=0).configurable_fields(
    temperature=ConfigurableField(id="temperature", name="Model Temperature"),
    model_name=ConfigurableField(id="model_name", name="Model Name")
)

chain = prompt | model | parser

# Override at call time
chain.invoke(
    {"topic": "LCEL"},
    config={"configurable": {"temperature": 0.9, "model_name": "gpt-4o"}}
)
```

### 5.10.2 Retry + Fallback (Production Resilience)

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

primary = ChatOpenAI(model="gpt-4o")
backup1 = ChatOpenAI(model="gpt-4o-mini")
backup2 = ChatAnthropic(model="claude-3-5-haiku-20241022")

# Retry primary 3x, then fallback chain
resilient_model = primary.with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True
).with_fallbacks([backup1, backup2])

chain = prompt | resilient_model | StrOutputParser()
```

### 5.10.3 Nested Chains (Chains of Chains)

```python
# Small reusable chains
summarize_chain = summarize_prompt | model | StrOutputParser()
translate_chain = translate_prompt | model | StrOutputParser()

# Compose into larger chain
full_pipeline = summarize_chain | translate_chain

# Also works with RunnableSequence explicitly
from langchain_core.runnables import RunnableSequence
full_pipeline = RunnableSequence(summarize_chain, translate_chain)
```

---

## 5.11 Type System & Generics

```python
# LangChain uses Python generics for type propagation
# Runnable[Input, Output]

prompt: Runnable[dict, ChatPromptValue]
model: Runnable[ChatPromptValue, AIMessage]
parser: Runnable[AIMessage, str]

chain = prompt | model | parser
# chain: Runnable[dict, str]  ← Inferred automatically!

# Custom Runnable with types
from langchain_core.runnables import Runnable
from typing import Generic, TypeVar

Input = TypeVar("Input")
Output = TypeVar("Output")

class MyRunnable(Runnable[dict[str, str], str]):
    def invoke(self, input: dict[str, str], config=None) -> str:
        return input["text"].upper()
```

---

## 5.12 Debugging & Inspection

```python
# Visualize chain
chain.get_graph().print_ascii()

# Input/Output schemas
print(chain.input_schema.schema())
print(chain.output_schema.schema())

# List all prompts in chain
chain.get_prompts()

# Get all runnables in sequence
for step in chain.steps:
    print(f"{step.__class__.__name__}: {step}")

# Stream events (fine-grained)
async for event in chain.astream_events(input, version="v2"):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
    elif event["event"] == "on_chain_start":
        print(f"Starting: {event['name']}")
```

---

## 5.13 When to Use Each Primitive

| Need | Primitive | Example |
|------|-----------|---------|
| Linear pipeline A→B→C | `RunnableSequence` (or `a \| b \| c`) | `prompt \| model \| parser` |
| Parallel independent ops | `RunnableParallel` (or `{...}`) | `{summary: chain1, sentiment: chain2}` |
| Preserve input through transform | `RunnablePassthrough()` | RAG: `{context: retriever, question: passthrough()}` |
| Add computed fields | `RunnablePassthrough.assign(...)` | Add `word_count`, `metadata` |
| Custom Python logic | `RunnableLambda(fn)` | `clean_text`, `format_docs`, `custom_transform` |
| Conditional routing | `RunnableBranch((cond, chain), default)` | Route by category, length, sentiment |
| Attach config/tools | `RunnableBinding` (`.bind()`, `.with_config()`) | `model.bind(tools=[...])` |

---

## 5.14 LCEL vs Runnables: The Relationship

```
LCEL (LangChain Expression Language)
    │
    ├─ Syntax: The `|` pipe operator
    ├─ Philosophy: Declarative composition
    └─ Result: Runnable objects

Runnables
    │
    ├─ Protocol: The `Runnable` interface (invoke/batch/stream)
    ├─ Primitives: Sequence, Parallel, Branch, Passthrough, Lambda
    ├─ Components: All prompts, models, parsers, retrievers implement it
    └─ Runtime: Execution engine (streaming, batching, async, config)
```

**LCEL is the syntax. Runnables are the implementation.**

---

## 5.15 Complete Decision Flowchart

```
Need to build a workflow?
│
├─► Simple linear chain?
│   └─► `prompt | model | parser` (auto RunnableSequence)
│
├─► Multiple independent operations on same input?
│   └─► `RunnableParallel` or dict shorthand `{...}`
│
├─► Need to preserve original input?
│   └─► `RunnablePassthrough()` in parallel branch
│
├─► Add computed fields to dict?
│   └─► `RunnablePassthrough.assign(...)`
│
├─► Custom Python transformation?
│   └─► `RunnableLambda(fn)` (auto-wrapped in `|`)
│
├─► Conditional routing (if/else)?
│   └─► `RunnableBranch((cond, chain), default)`
│
├─► Need cycles / state / human-in-loop?
│   └─► **STOP — Use LangGraph**
│
└─► Production hardening?
    └─► `.with_retry().with_fallbacks().with_config()`
```

---

## 5.16 Code Reference

| File | Primitive Demonstrated | Key Pattern |
|------|----------------------|-------------|
| `runnable_sequence.py` | `RunnableSequence` | Explicit sequential composition |
| `runnable_parallel.py` | `RunnableParallel` | Concurrent tweet + LinkedIn generation |
| `runnable_passthrough.py` | `RunnablePassthrough` | Preserve joke while explaining it |
| `runnable_lambda.py` | `RunnableLambda` | Word count as parallel branch |
| `runnable_branch.py` | `RunnableBranch` | Summarize long reports, pass short ones |

---

## 5.17 DeepLearning.AI Course Connection

### "Functions, Tools & Agents with LangChain" — LCEL Lesson

| Course Concept | Chapter 5 Coverage |
|----------------|-------------------|
| **Runnable Protocol** | invoke/batch/stream/async, RunnableConfig |
| **RunnableSequence** | Pipe operator, sequential composition |
| **RunnableParallel** | Parallel branches, dict shorthand |
| **RunnablePassthrough** | RAG pattern, assign() |
| **RunnableLambda** | Custom functions in chains |
| **RunnableBranch** | Routing, conditional logic |
| **Streaming** | Automatic propagation |
| **Fallbacks/Retry** | Production resilience |

---

## 5.18 Production Checklist

### Design
- [ ] **Single responsibility** — Each runnable does one thing
- [ ] **Composability** — Build small, reuse via `|`
- [ ] **Type annotations** — Use generics `Runnable[Input, Output]`
- [ ] **Config-driven** — `configurable_fields` for runtime params

### Reliability
- [ ] **Fallbacks** on every external call
- [ ] **Retries** with exponential backoff
- [ ] **Timeouts** configured on models
- [ ] **Rate limits** via `max_concurrency`

### Observability
- [ ] **LangSmith tags/metadata** via config
- [ ] **Custom callbacks** for logging
- [ ] **Graph inspection** in CI/CD (`get_graph().print_ascii()`)

### Performance
- [ ] **Parallel** independent operations
- [ ] **Batch** multiple inputs (`chain.batch()`)
- [ ] **Async** in web servers (`await chain.ainvoke()`)
- [ ] **Stream** for UX (`chain.astream()`)

---

## 5.19 Quick Reference: All Key Classes

| Category | Class | Package | Purpose |
|----------|-------|---------|---------|
| **Protocol** | `Runnable` | `langchain_core.runnables` | Base interface |
| **Composition** | `RunnableSequence` | `langchain_core.runnables` | Sequential (`a \| b`) |
| **Composition** | `RunnableParallel` | `langchain_core.runnables` | Parallel (`{...}`) |
| **Composition** | `RunnableBranch` | `langchain_core.runnables` | Conditional routing |
| **Primitive** | `RunnablePassthrough` | `langchain_core.runnables` | Identity + `assign()` |
| **Primitive** | `RunnableLambda` | `langchain_core.runnables` | Wrap function |
| **Primitive** | `RunnablePick` | `langchain_core.runnables` | Select dict keys |
| **Primitive** | `itemgetter` | `operator` | Extract field |
| **Modifier** | `RunnableBinding` | `langchain_core.runnables` | `.bind()`, `.with_config()` |
| **Modifier** | `RunnableWithFallbacks` | `langchain_core.runnables` | `.with_fallbacks()` |
| **Modifier** | `RunnableWithRetry` | `langchain_core.runnables` | `.with_retry()` |
| **Config** | `RunnableConfig` | `langchain_core.runnables` | Execution config |
| **Config** | `ConfigurableField` | `langchain_core.runnables` | Runtime field override |

---

## 5.20 Summary: The Runnable Revolution

| Before Runnables (v0.1) | After Runnables (v0.2+) |
|------------------------|------------------------|
| 50+ Chain classes | 5 Primitive Runnables |
| `predict()`, `format()`, `get_relevant_documents()`, `parse()` | **`invoke()` everywhere** |
| Manual streaming per chain | **Automatic streaming** |
| No batching | **`.batch()` free** |
| No async | **`.ainvoke()` / `.astream()` free** |
| Fragile composition | **Composable by default** |
| Opaque internals | **`get_graph()` inspection** |
| Steep learning curve | **One interface to rule them all** |

> **The Runnable protocol is the single most important abstraction in LangChain.** It turned a fragmented ecosystem of special-purpose chains into a unified, composable system where every component — prompts, models, parsers, retrievers, tools, custom functions — speaks the same language.

---

## 5.21 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **Runnables:** https://python.langchain.com/docs/concepts/runnables/
- **RunnableSequence:** https://python.langchain.com/docs/how_to/sequence/
- **RunnableParallel:** https://python.langchain.com/docs/how_to/parallel/
- **RunnableBranch:** https://python.langchain.com/docs/how_to/routing/
- **RunnablePassthrough:** https://python.langchain.com/docs/how_to/passthrough/
- **RunnableLambda:** https://python.langchain.com/docs/how_to/lambdas/

### Comprehensive Guides
- **DevOps LCEL Guide 2026:** https://devops.gheware.com/blog/posts/langchain-expression-language-lcel-guide-2026.html
- **TeachYou.ai Runnable Guide:** https://teachyou.ai/blog/langchain-lcel-guide
- **Official Runnable Interface:** https://mintlify.com/langchain-ai/langchain/concepts/runnables

---

*End of Chapter 5 — LangChain Runnables: The Universal Interface*