# Chapter 4: Chains & LCEL — The Core of LangChain

## 4.1 Why Chains Are the Heart of LangChain

> **"LangChain" = Language + Chain**

The framework's name comes from this concept. Chains are **composable pipelines** that transform input through a series of steps — each step's output becomes the next step's input.

### The Problem Before LCEL (v0.1 Legacy)

```python
# ❌ LEGACY (v0.1) — Don't use
from langchain.chains import LLMChain, SimpleSequentialChain

llm = OpenAI()
prompt1 = PromptTemplate(...)
prompt2 = PromptTemplate(...)

chain1 = LLMChain(llm=llm, prompt=prompt1)
chain2 = LLMChain(llm=llm, prompt=prompt2)

# Manual composition, no streaming, no async, no batching
overall_chain = SimpleSequentialChain(chains=[chain1, chain2])
result = overall_chain.run("topic")
```

**Legacy Chain Problems:**
| Issue | Impact |
|-------|--------|
| Each chain type has different API | `LLMChain.run()` vs `SequentialChain.run()` vs `ConversationalRetrievalChain()` |
| No unified streaming | Must implement per-chain |
| No batching | Manual loops |
| No async | Blocking calls only |
| Fragile composition | Glue code between chains |
| Hard to debug | Opaque internals |

---

## 4.2 LCEL — LangChain Expression Language

### The Core Insight

> **Every component implements the `Runnable` protocol. Composition via `|` operator creates a new `Runnable` with ALL capabilities (stream, batch, async, fallback, retry) automatically.**

### The Runnable Protocol (Universal Interface)

```python
# Simplified from langchain_core.runnables.base.Runnable
class Runnable(Generic[Input, Output]):
    # REQUIRED - implement this
    def invoke(self, input: Input, config: RunnableConfig = None) -> Output: ...

    # PROVIDED BY DEFAULT (subclasses can override for optimization)
    def batch(self, inputs: list[Input], config: RunnableConfig = None) -> list[Output]: ...
    def stream(self, input: Input, config: RunnableConfig = None) -> Iterator[Output]: ...
    async def ainvoke(self, input: Input, config: RunnableConfig = None) -> Output: ...
    async def abatch(self, inputs: list[Input], config: RunnableConfig = None) -> list[Output]: ...
    async def astream(self, input: Input, config: RunnableConfig = None) -> AsyncIterator[Output]: ...
```

**Every LCEL component implements this:**
| Component | Package | Input | Output |
|-----------|---------|-------|--------|
| `ChatPromptTemplate` | `langchain_core.prompts` | `dict` | `PromptValue` |
| `ChatOpenAI` / `ChatAnthropic` | `langchain_openai` / `langchain_anthropic` | `PromptValue` / `list[BaseMessage]` | `AIMessage` |
| `StrOutputParser` | `langchain_core.output_parsers` | `AIMessage` | `str` |
| `JsonOutputParser` | `langchain_core.output_parsers` | `AIMessage` | `dict` |
| `RunnableLambda` | `langchain_core.runnables` | `Any` | `Any` |
| `RunnableParallel` | `langchain_core.runnables` | `Any` | `dict` |
| `RunnablePassthrough` | `langchain_core.runnables` | `Any` | `Any` |
| `RunnableBranch` | `langchain_core.runnables` | `Any` | `Any` |
| Retrievers | `langchain_core.retrievers` | `str` | `list[Document]` |

---

## 4.3 The Pipe Operator (`|`) — How It Works

```python
# This:
chain = prompt | model | parser

# Is equivalent to:
from langchain_core.runnables import RunnableSequence
chain = RunnableSequence(first=prompt, middle=[model], last=parser)

# Which is a Runnable itself!
print(type(chain))  # <class 'langchain_core.runnables.base.RunnableSequence'>
```

**Execution Flow:**
```python
chain.invoke({"topic": "cricket"})
# 1. prompt.invoke({"topic": "cricket"}) → PromptValue
# 2. model.invoke(PromptValue) → AIMessage
# 3. parser.invoke(AIMessage) → str
# 4. Returns final string
```

**Chain Inspection (Built-in Debugging):**
```python
# Visualize the chain graph
chain.get_graph().print_ascii()

# Input/Output schemas (auto-generated Pydantic)
print(chain.input_schema.schema())
# {'properties': {'topic': {'type': 'string'}}, 'required': ['topic']}
print(chain.output_schema.schema())
# {'type': 'string'}
```

---

## 4.4 Five Invocation Modes — One Chain, Zero Extra Code

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Explain {concept} in one sentence.")
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()
chain = prompt | model | parser

# 1. SYNC — Single call
result = chain.invoke({"concept": "recursion"})
print(result)

# 2. STREAMING — Token by token (for UX)
for chunk in chain.stream({"concept": "recursion"}):
    print(chunk, end="", flush=True)
print()

# 3. BATCH — Parallel multiple inputs
results = chain.batch([
    {"concept": "recursion"},
    {"concept": "memoization"},
    {"concept": "dynamic programming"},
], config={"max_concurrency": 5})  # Rate limit control

# 4. ASYNC — Non-blocking (FastAPI, etc.)
async def main():
    result = await chain.ainvoke({"concept": "async LCEL"})
    print(result)

# 5. ASYNC STREAMING — Async token streaming
async def stream_main():
    async for chunk in chain.astream({"concept": "async streaming"}):
        print(chunk, end="", flush=True)
```

---

## 4.5 Fundamental Chain Patterns

### Pattern 1: Basic Chain (Prompt → Model → Parser)
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_messages([
        ("system", "You are a {role} expert. Be concise."),
        ("human", "{question}")
    ])
    | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    | StrOutputParser()
)

result = chain.invoke({"role": "Kubernetes", "question": "What is a DaemonSet?"})
```

### Pattern 2: Sequential Chain (Multi-step Pipeline)
```python
# sequential_chain.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt1 = PromptTemplate(
    template="Generate a detailed report on {topic}",
    input_variables=["topic"]
)
prompt2 = PromptTemplate(
    template="Generate a 5 pointer summary from the following text\n{text}",
    input_variables=["text"]
)

model = ChatOpenAI()
parser = StrOutputParser()

# Chain: prompt1 → model → parser → prompt2 → model → parser
chain = prompt1 | model | parser | prompt2 | model | parser

result = chain.invoke({"topic": "Unemployment in India"})
print(result)
```

**Key Insight:** The output of `parser` (string) automatically becomes input to `prompt2` (expects `{"text": str}`).

### Pattern 3: Parallel Chain (RunnableParallel)
```python
# parallel_chain.py
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel

model1 = ChatOpenAI()
model2 = ChatAnthropic(model_name="claude-3-5-sonnet-20241022")

prompt1 = PromptTemplate(
    template="Generate short and simple notes from the following text\n{text}",
    input_variables=["text"]
)
prompt2 = PromptTemplate(
    template="Generate 5 short question answers from the following text\n{text}",
    input_variables=["text"]
)
prompt3 = PromptTemplate(
    template="Merge the provided notes and quiz into a single document\nnotes -> {notes} and quiz -> {quiz}",
    input_variables=["notes", "quiz"]
)

parser = StrOutputParser()

# Fan out: same input → multiple branches in PARALLEL
parallel_chain = RunnableParallel({
    "notes": prompt1 | model1 | parser,
    "quiz": prompt2 | model2 | parser
})

# Fan in: merge results
merge_chain = prompt3 | model1 | parser

# Full chain: parallel → merge
chain = parallel_chain | merge_chain

text = """Support vector machines (SVMs) are a set of supervised learning methods..."""
result = chain.invoke({"text": text})
```

**Execution:**
```
Input: {"text": "..."}
         │
         ▼
    ┌────┴────┐
    ▼         ▼
 notes      quiz        ← PARALLEL (concurrent)
    │         │
    └────┬────┘
         ▼
      merge       ← SEQUENTIAL
         │
         ▼
     Output
```

**Performance Gain:** 3 sequential LLM calls = ~9s → Parallel = ~3s (slowest branch)

### Pattern 4: Conditional Chain (RunnableBranch)
```python
# conditional_chain.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser, PydanticOutputParser
from langchain_core.runnables import RunnableBranch, RunnableLambda
from pydantic import BaseModel, Field
from typing import Literal

model = ChatOpenAI()
parser = StrOutputParser()

# Step 1: Classify sentiment
class Feedback(BaseModel):
    sentiment: Literal["positive", "negative"] = Field(description="Sentiment of feedback")

parser2 = PydanticOutputParser(pydantic_object=Feedback)

prompt1 = PromptTemplate(
    template="Classify the sentiment of the following feedback text into positive or negative\n{feedback}\n{format_instruction}",
    input_variables=["feedback"],
    partial_variables={"format_instruction": parser2.get_format_instructions()}
)

classifier_chain = prompt1 | model | parser2

# Step 2: Different responses based on classification
prompt2 = PromptTemplate(
    template="Write an appropriate response to this positive feedback\n{feedback}",
    input_variables=["feedback"]
)
prompt3 = PromptTemplate(
    template="Write an appropriate response to this negative feedback\n{feedback}",
    input_variables=["feedback"]
)

branch_chain = RunnableBranch(
    (lambda x: x.sentiment == "positive", prompt2 | model | parser),
    (lambda x: x.sentiment == "negative", prompt3 | model | parser),
    RunnableLambda(lambda x: "could not find sentiment")  # default
)

# Full chain: classify → branch
chain = classifier_chain | branch_chain

print(chain.invoke({"feedback": "This is a beautiful phone"}))
print(chain.invoke({"feedback": "This phone is terrible and slow"}))
```

**RunnableBranch Signature:**
```python
RunnableBranch(
    (condition1, runnable1),
    (condition2, runnable2),
    default_runnable  # no tuple = default
)
# Conditions evaluated in order; first match wins
```

---

## 4.6 Core Runnable Primitives — Building Blocks

### 4.6.1 RunnablePassthrough — Pass Through / Add Keys

```python
from langchain_core.runnables import RunnablePassthrough
from operator import itemgetter

# 1. Pass through unchanged
passthrough = RunnablePassthrough()
result = passthrough.invoke("hello")  # "hello"

# 2. assign() — Add computed keys while preserving input
chain = RunnablePassthrough.assign(
    word_count=lambda x: len(x["text"].split()),
    upper=lambda x: x["text"].upper()
)
result = chain.invoke({"text": "Hello world from LangChain"})
# {"text": "Hello world from LangChain", "word_count": 5, "upper": "HELLO WORLD FROM LANGCHAIN"}

# 3. CRITICAL PATTERN: RAG — Preserve question while retrieving context
rag_chain = (
    {
        "context": retriever | RunnableLambda(format_docs),
        "question": RunnablePassthrough()  # ← Pass original question through
    }
    | rag_prompt
    | model
    | StrOutputParser()
)
```

### 4.6.2 RunnableLambda — Wrap Any Python Function

```python
from langchain_core.runnables import RunnableLambda

def format_docs(docs) -> str:
    """Format retrieved Document objects for prompt."""
    return "\n\n---\n\n".join(
        f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
        for doc in docs
    )

def clean_text(text: str) -> str:
    import re
    return re.sub(r"\s+", " ", text).strip()

# Explicit wrapping
format_docs_runnable = RunnableLambda(format_docs)
clean_runnable = RunnableLambda(clean_text)

# AUTO-WRAPPING: Just use function directly in pipe chain!
chain = (
    prompt
    | model
    | StrOutputParser()
    | clean_text          # Auto-wrapped to RunnableLambda
    | format_docs         # Auto-wrapped
)
```

**Async Functions (for astream/abatch):**
```python
async def add_disclaimer_async(text: str) -> str:
    return f"{text}\n\n(This is AI-generated content.)"

chain = prompt | model | StrOutputParser() | add_disclaimer_async
result = await chain.ainvoke({"topic": "RunnableLambda"})
```

### 4.6.3 RunnableParallel — Explicit Parallel Execution

```python
from langchain_core.runnables import RunnableParallel

# Explicit form
analysis_chain = RunnableParallel(
    summary=summary_chain,
    sentiment=sentiment_chain,
    keywords=keywords_chain
)

# Implicit form (dict shorthand) — auto-converted to RunnableParallel
chain = {
    "summary": summary_chain,
    "sentiment": sentiment_chain,
    "keywords": keywords_chain
}

result = chain.invoke({"text": "Long article..."})
# {"summary": "...", "sentiment": "...", "keywords": "..."}
```

**When to Use Which:**
| Form | Use Case |
|------|----------|
| Dict shorthand | Simple, readable, 2-3 branches |
| `RunnableParallel()` | Complex, many branches, need explicit config |

### 4.6.4 RunnablePick — Select Keys from Dict

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda

chain = RunnableParallel(
    a=RunnableLambda(lambda x: x * 2),
    b=RunnableLambda(lambda x: x * 3),
    c=RunnableLambda(lambda x: x * 4)
)

# Pick only needed keys
chain_filtered = chain.pick(["a", "c"])
result = chain_filtered.invoke(5)  # {"a": 10, "c": 20}
```

### 4.6.5 RunnableWithFallbacks — Resilience

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

primary = ChatOpenAI(model="gpt-4o")
backup = ChatOpenAI(model="gpt-4o-mini")
# Or different provider
backup2 = ChatAnthropic(model="claude-3-5-haiku-20241022")

# Retry primary 3 times, then fallback
resilient_model = primary.with_retry(stop_after_attempt=3).with_fallbacks([backup, backup2])

chain = prompt | resilient_model | StrOutputParser()
```

### 4.6.6 RunnableBinding — Bind Config/Kwargs

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# Bind tools for function calling
model_with_tools = model.bind(tools=[my_tool_schema])

# Bind config (tags, metadata, callbacks)
model_tagged = model.with_config(tags=["production", "user-123"])

# Bind stop sequences
model_stopped = model.bind(stop=["\n\n", "END"])

# Use in chain
chain = prompt | model_with_tools | parser
```

---

## 4.7 Advanced Chain Patterns

### 4.7.1 Router Chain (Classification + Routing)

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch, RunnableLambda, RunnableParallel

model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Step 1: Classify
classifier = (
    ChatPromptTemplate.from_template(
        "Classify this question into exactly one category: tech, science, history.\n"
        "Question: {question}\nCategory:"
    )
    | model
    | StrOutputParser()
    | (lambda x: x.strip().lower())
)

# Step 2: Specialized chains
tech_chain = ChatPromptTemplate.from_template("As a senior engineer, answer: {question}") | model | StrOutputParser()
science_chain = ChatPromptTemplate.from_template("As a PhD scientist, answer: {question}") | model | StrOutputParser()
history_chain = ChatPromptTemplate.from_template("As a historian, answer: {question}") | model | StrOutputParser()

# Step 3: Route
def route(info: dict) -> str:
    category = info["category"]
    question = info["question"]
    if "tech" in category:
        return tech_chain.invoke({"question": question})
    elif "science" in category:
        return science_chain.invoke({"question": question})
    else:
        return history_chain.invoke({"question": question})

# Full router chain
router_chain = (
    RunnableParallel(
        category=classifier,
        question=itemgetter("question")
    )
    | RunnableLambda(route)
)

result = router_chain.invoke({"question": "How does TCP/IP work?"})
```

### 4.7.2 RAG Chain (Standard Pattern)

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

RAG_PROMPT = ChatPromptTemplate.from_template("""
Answer the question using only the context below.
Cite sources using [1], [2] notation.

Context: {context}

Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini")

# Primary + fallback model
model = llm.with_fallbacks([ChatOpenAI(model="gpt-4o-mini")])

rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | RAG_PROMPT
    | model
    | StrOutputParser()
)

# Streaming RAG
for chunk in rag_chain.stream("How does LCEL propagate streaming?"):
    print(chunk, end="", flush=True)
```

### 4.7.3 Chain with Metadata Enrichment

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from operator import itemgetter

# Enrich data as it flows
chain = (
    # Start with question
    RunnablePassthrough.assign(
        # Add context from retriever
        context=itemgetter("question") | retriever,
    )
    | RunnablePassthrough.assign(
        # Add answer using context
        answer=(
            ChatPromptTemplate.from_template(
                "Answer based on context:\n{context}\n\nQuestion: {question}"
            )
            | model
            | StrOutputParser()
        )
    )
)

result = chain.invoke({"question": "What is LangChain?"})
# {"question": "...", "context": [Document...], "answer": "LangChain is..."}
```

### 4.7.4 Multi-Model Chain (Cost Optimization)

```python
from langchain_openai import ChatOpenAI
from langchain_core.runnables import RunnableBranch

# Use cheap model for simple tasks, expensive for complex
cheap_model = ChatOpenAI(model="gpt-4o-mini")
expensive_model = ChatOpenAI(model="gpt-4o")

# Classify complexity
complexity_chain = (
    ChatPromptTemplate.from_template(
        "Rate complexity of this question 1-10:\n{question}\nScore:"
    )
    | cheap_model
    | StrOutputParser()
    | (lambda x: int(x.strip()))
)

simple_chain = ChatPromptTemplate.from_template("Answer simply: {question}") | cheap_model
complex_chain = ChatPromptTemplate.from_template("Answer thoroughly: {question}") | expensive_model

chain = RunnableBranch(
    (lambda x: x["complexity"] <= 5, simple_chain),
    complex_chain
)

full_chain = RunnableParallel(
    complexity=complexity_chain,
    question=itemgetter("question")
) | chain
```

---

## 4.8 Configuration & Context (RunnableConfig)

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.callbacks import StdOutCallbackHandler

config = RunnableConfig(
    tags=["production", "experiment-v2"],
    metadata={"user_id": "123", "session": "abc"},
    callbacks=[StdOutCallbackHandler()],
    max_concurrency=10,
    run_name="question_answering_chain",
    configurable={
        "model_name": "gpt-4o",
        "temperature": 0.0
    }
)

# Config merges down the chain automatically
result = chain.invoke(input, config=config)

# Configurable fields — change at runtime
from langchain_core.runnables import ConfigurableField

model = ChatOpenAI(model="gpt-4o-mini", temperature=0).configurable_fields(
    temperature=ConfigurableField(id="temperature", name="Model Temperature")
)

chain = prompt | model | parser

# Override temperature per call
chain.invoke(
    {"topic": "LCEL"},
    config={"configurable": {"temperature": 0.9}}
)
```

---

## 4.9 Streaming Deep Dive

### How Streaming Propagates

```python
# Chain: prompt | model | StrOutputParser()
# When you call chain.stream(input):
# 1. prompt.stream() → yields PromptValue (usually single chunk)
# 2. model.stream(PromptValue) → yields AIMessageChunk (token by token)
# 3. StrOutputParser.stream(AIMessageChunk) → yields string chunks
# 4. Final chain.stream() yields string chunks to caller
```

### Streaming Types

| Parser | Streaming Behavior |
|--------|-------------------|
| `StrOutputParser` | ✅ Streams token chunks incrementally |
| `JsonOutputParser` | ⚠️ Buffers until valid JSON, then yields complete object |
| `PydanticOutputParser` | ⚠️ Same as JSON — needs full response |
| `with_structured_output()` | ⚠️ Model-native streaming (function calling) — varies by provider |

### Streaming with Callbacks (LangSmith / Custom)

```python
from langchain_core.callbacks import BaseCallbackHandler

class StreamingHandler(BaseCallbackHandler):
    def on_llm_new_token(self, token: str, **kwargs):
        print(f"Token: {token}", end="", flush=True)

handler = StreamingHandler()
result = chain.invoke(
    {"topic": "streaming"},
    config={"callbacks": [handler]}
)
```

---

## 4.10 Legacy Chain Migration Guide

| Legacy (v0.1) | Modern LCEL (v0.2+) |
|---------------|---------------------|
| `LLMChain(llm, prompt)` | `prompt \| llm \| StrOutputParser()` |
| `SimpleSequentialChain([chain1, chain2])` | `chain1 \| chain2` |
| `SequentialChain(chains=[...], input_variables=[...])` | `RunnableParallel` + explicit mapping |
| `ConversationalRetrievalChain` | `RunnableWithMessageHistory` + RAG chain |
| `RouterChain` | `RunnableBranch` |
| `MapReduceDocumentsChain` | `RunnableParallel` + `RunnableLambda` |
| `StuffDocumentsChain` | `RunnableLambda(format_docs)` in RAG pattern |
| `LLMMathChain` | Custom `RunnableLambda` with Python eval |

### Migration Example

```python
# ❌ LEGACY
from langchain.chains import LLMChain, SimpleSequentialChain
chain1 = LLMChain(llm=llm, prompt=prompt1)
chain2 = LLMChain(llm=llm, prompt=prompt2)
overall = SimpleSequentialChain(chains=[chain1, chain2])
result = overall.run("topic")

# ✅ MODERN LCEL
chain = prompt1 | llm | StrOutputParser() | prompt2 | llm | StrOutputParser()
result = chain.invoke({"topic": "topic"})
```

---

## 4.11 When LCEL Isn't Enough → LangGraph

> **LCEL = DAG (Directed Acyclic Graph) — Straight line or tree**
> **LangGraph = General Graph — Cycles, state, persistence, human-in-the-loop**

### Use LCEL When:
- ✅ Linear or tree-shaped pipelines
- ✅ Fixed sequence of steps
- ✅ Fan-out/fan-in parallelism
- ✅ Simple conditional routing
- ✅ Streaming, batching, async needed automatically

### Use LangGraph When:
- 🔄 **Cycles** — Agent loops (model → tool → model → tool...)
- 🔄 **Complex state** — Multiple evolving variables across steps
- 🔄 **Persistence** — Checkpoint/resume long-running jobs
- 🔄 **Human-in-the-loop** — Pause for approval, resume
- 🔄 **Dynamic routing** — Graph edges determined at runtime

### Signal You've Outgrown LCEL:
```python
# ❌ Don't do this in LCEL
while True:
    result = agent_chain.invoke(state)
    if result.is_final:
        break
    state = update_state(state, result)

# ✅ Move to LangGraph
from langgraph.graph import StateGraph
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge("agent", "tools")
graph.add_edge("tools", "agent")
graph.set_entry_point("agent")
app = graph.compile()
result = app.invoke(initial_state)
```

---

## 4.12 Production Best Practices

### Chain Design
- [ ] **Single responsibility** — Each chain does one thing well
- [ ] **Type annotations** — Use Pydantic for input/output schemas
- [ ] **Composability** — Build small reusable chains, compose with `|`
- [ ] **Config-driven** — Use `configurable_fields` for runtime params

### Reliability
- [ ] **Fallbacks** — `with_fallbacks([backup_model])` on every external call
- [ ] **Retries** — `with_retry(stop_after_attempt=3)` for transient errors
- [ ] **Timeouts** — Configure in model (`request_timeout=30`)
- [ ] **Rate limiting** — `max_concurrency` in batch config

### Observability
- [ ] **LangSmith tracing** — Auto-enabled, add tags/metadata via config
- [ ] **Custom callbacks** — Log inputs/outputs, latency, token counts
- [ ] **Chain inspection** — `get_graph().print_ascii()` in CI/CD

### Performance
- [ ] **Parallel where possible** — `RunnableParallel` for independent steps
- [ ] **Batch processing** — `chain.batch(inputs)` instead of loops
- [ ] **Async in web servers** — `await chain.ainvoke()` / `astream()`
- [ ] **Model selection** — `gpt-4o-mini` for simple, `gpt-4o` for complex

---

## 4.13 Complete LCEL Reference Card

### Composition
| Syntax | Creates | Use For |
|--------|---------|---------|
| `a \| b` | `RunnableSequence` | Sequential pipeline |
| `{"k": a, "k2": b}` | `RunnableParallel` | Parallel fan-out |
| `RunnableParallel(a=..., b=...)` | `RunnableParallel` | Explicit parallel |
| `RunnableBranch((cond, run), default)` | `RunnableBranch` | Conditional routing |

### Core Primitives
| Primitive | Purpose |
|-----------|---------|
| `RunnablePassthrough()` | Identity; `assign()` adds keys |
| `RunnableLambda(func)` | Wrap any function |
| `RunnablePick(keys)` | Select dict keys |
| `itemgetter("key")` | Extract dict field (operator module) |

### Execution Methods (on ANY chain)
| Method | Async | Use Case |
|--------|-------|----------|
| `invoke(input)` | `ainvoke(input)` | Single request |
| `stream(input)` | `astream(input)` | Token streaming |
| `batch(inputs)` | `abatch(inputs)` | Parallel multiple |

### Modifiers (Return New Runnable)
| Modifier | Purpose |
|----------|---------|
| `with_fallbacks([fallbacks])` | Failover on error |
| `with_retry(stop_after_attempt=3)` | Retry transient errors |
| `with_config(config)` | Attach config |
| `configurable_fields(...)` | Runtime parameter override |
| `bind(kwargs)` | Bind model kwargs (tools, stop) |

### Debugging
```python
chain.get_graph().print_ascii()           # Visual graph
print(chain.input_schema.schema())        # Input schema
print(chain.output_schema.schema())       # Output schema
chain.get_prompts()                       # List all prompts in chain
```

---

## 4.14 Quick Reference: Key Classes

| Category | Class | Package | Purpose |
|----------|-------|---------|---------|
| **Composition** | `RunnableSequence` | `langchain_core.runnables` | Sequential (`a \| b`) |
| **Composition** | `RunnableParallel` | `langchain_core.runnables` | Parallel (`{...}`) |
| **Composition** | `RunnableBranch` | `langchain_core.runnables` | Conditional routing |
| **Primitive** | `RunnablePassthrough` | `langchain_core.runnables` | Pass-through, `assign()` |
| **Primitive** | `RunnableLambda` | `langchain_core.runnables` | Wrap Python function |
| **Primitive** | `RunnablePick` | `langchain_core.runnables` | Select dict keys |
| **Primitive** | `itemgetter` | `operator` | Extract field |
| **Modifier** | `RunnableBinding` | `langchain_core.runnables` | `.bind()`, `.with_config()` |
| **Modifier** | `RunnableWithFallbacks` | `langchain_core.runnables` | `.with_fallbacks()` |
| **Config** | `RunnableConfig` | `langchain_core.runnables` | Execution config |
| **Config** | `ConfigurableField` | `langchain_core.runnables` | Runtime field override |

---

## 4.15 Summary: Chain Type Decision Tree

```
Need a chain?
│
├─► Linear sequence? (A → B → C)
│   │
│   ├─ Simple? (prompt → model → parser)
│   │   └─► `prompt | model | parser`
│   │
│   ├─ Multi-step with intermediate processing?
│   │   └─► `step1 | step2 | step3 | ...`
│   │
│   └─ Need to preserve original input?
│       └─► Use `RunnablePassthrough()` in parallel branch
│
├─► Parallel independent operations?
│   │
│   └─► `RunnableParallel({...})` or `{...}` dict shorthand
│
├─► Conditional routing?
│   │
│   ├─ Simple (2-3 branches)?
│   │   └─► `RunnableBranch((cond, chain), default)`
│   │
│   └─ Complex (depends on model output)?
│       └─► Classifier chain → `RunnableBranch` (Router pattern)
│
├─► Need cycles / state / human-in-loop?
│   │
│   └─► STOP — Use **LangGraph** instead
│
└─► Need resilience?
    └─► `.with_retry().with_fallbacks()` on external calls
```

---

## 4.16 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **LCEL:** https://python.langchain.com/docs/concepts/lcel/
- **Runnables:** https://python.langchain.com/docs/concepts/runnables/
- **RunnableParallel:** https://python.langchain.com/docs/how_to/parallel/
- **RunnableBranch:** https://python.langchain.com/docs/how_to/routing/
- **RunnablePassthrough:** https://python.langchain.com/docs/how_to/passthrough/
- **Migration Guide:** https://python.langchain.com/docs/versions/migrating_chains/

### Comprehensive Guides
- **DevOps LCEL Guide 2026:** https://devops.gheware.com/blog/posts/langchain-expression-language-lcel-guide-2026.html
- **TeachYou.ai LCEL Guide:** https://teachyou.ai/blog/langchain-lcel-guide
- **Mintlify Official:** https://mintlify.wiki/langchain-ai/langchain/concepts/chains

---

*End of Chapter 4 — Chains & LCEL: The Core of LangChain*