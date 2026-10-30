# Chapter 1: Introduction to LangChain & Core Components

## 1.1 What is LangChain?

**One-sentence definition:**
> **LangChain is an open-source framework for developing applications powered by LLMs.**

If you want to build *any* LLM-based application, LangChain is the framework that helps you build it.

---

## 1.2 Why Do We Need LangChain? (The "Why Before What")

Before understanding *what* LangChain is, we must understand *why* it was needed. The best way to grasp this is through a real-world system design problem.

### The Motivating Example: "Chat with Your PDF"

Imagine building an application where:
1. Users upload a PDF (e.g., a Machine Learning textbook)
2. Users can *chat* with the PDF — ask questions like:
   - "Explain page 5 like I'm 5 years old"
   - "Summarize Chapter 3 in 200 words"
   - "What are the key assumptions in the proof on page 42?"

#### System Design from Scratch (Without LangChain)

To build this, you'd need to orchestrate multiple components:

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐
│  PDF Upload │───▶│ Text Extract │───▶│ Text Split  │───▶│ Embedding Gen  │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘
                                                                    │
                                                                    ▼
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐
│  Response   │◀───│  LLM Call    │◀───│ Context     │◀───│ Vector Search  │
│  Generation │    │  (Answer)    │    │ Assembly    │    │ (Retrieval)    │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘
```

**Challenges you'd face:**
- **Component Integration:** Each box above has different APIs, data formats, error modes
- **Pipeline Orchestration:** Passing outputs between components manually is brittle
- **Model Lock-in:** OpenAI API ≠ Anthropic API ≠ Google API — different signatures, response formats
- **State Management:** Conversation history, memory, context windows
- **Scalability:** Chunking strategies, embedding models, vector DB choices
- **Production Concerns:** Streaming, async, retries, observability, cost tracking

### How LangChain Solves This

LangChain provides **standardized interfaces** and **composable primitives** for every box above:

| Problem | LangChain Solution |
|---------|-------------------|
| Model API differences | **Model Component** — unified interface for 50+ LLM providers |
| Prompt engineering complexity | **Prompt Component** — templates, few-shot, partial variables, output parsers |
| Pipeline orchestration | **Chain Component (LCEL)** — declarative composition with `|` operator |
| External knowledge | **Index/Retriever Component** — loaders, splitters, embeddings, vector stores |
| Conversation state | **Memory Component** — buffer, window, summary, custom |
| Autonomous action | **Agent Component** — reasoning + tool use (ReAct, tool-calling, plan-and-execute) |

**Result:** What took 500+ lines of custom orchestration code becomes ~20 lines of declarative LangChain code.

---

## 1.3 What Can You Build with LangChain?

| Category | Examples |
|----------|----------|
| **Conversational Chatbots** | Customer support, personal assistants, roleplay bots |
| **Knowledge Assistants (RAG)** | "Chat with PDF/docs/Notion/Slack/SQL", enterprise search |
| **AI Agents** | Web research agents, coding agents, travel planners, email triage |
| **Structured Extraction** | Invoice parsing, resume screening, contract analysis |
| **Workflow Automation** | Multi-step pipelines: summarize → translate → extract → notify |
| **Evaluation & Testing** | LLM-as-judge, benchmarking, regression testing |

---

## 1.4 Popular Alternatives to LangChain

| Framework | Focus | Best For |
|-----------|-------|----------|
| **LlamaIndex** | Data indexing & retrieval | RAG-heavy applications, document processing |
| **Haystack** | Production NLP pipelines | Enterprise search, QA systems |
| **Semantic Kernel** | Microsoft ecosystem | .NET/C# integration, planners |
| **AutoGen** | Multi-agent conversations | Agent societies, group chat |
| **CrewAI** | Role-based agent teams | Business process automation |
| **LangGraph** (by LangChain) | Stateful, cyclic agent graphs | Complex agent workflows, human-in-the-loop |

> **Note:** LangGraph is now the **recommended way to build agents** in the LangChain ecosystem (replaces `AgentExecutor` for production use).

---

## 1.5 LangChain's Six Core Components

The entire framework organizes around **six fundamental components**. Master these, and you understand 90% of LangChain.

```
┌─────────────────────────────────────────────────────────────────┐
│                        LANGCHAIN ARCHITECTURE                   │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  MODELS  │ PROMPTS  │  CHAINS  │ INDEXES  │ MEMORY   │ AGENTS   │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Talk to  │ Structure│ Compose  │ Retrieve │ Remember │ Reason & │
│ any LLM  │ inputs   │ pipelines│ knowledge│ context  │ act      │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 1.6 Component Deep-Dives

### 1.6.1 Models — The Universal LLM Interface

**Core Problem:** Every LLM provider has a different API signature.

```python
# OpenAI
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}]
)

# Anthropic (Claude)
import anthropic
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
```

**LangChain Solution — Unified Interface:**

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Same interface, different provider
llm_openai = ChatOpenAI(model="gpt-4o", temperature=0)
llm_claude = ChatAnthropic(model="claude-3-5-sonnet-20241022", temperature=0)

# Identical invocation
response = llm_openai.invoke("Hello")
response = llm_claude.invoke("Hello")  # Swap one line!
```

**Two Model Types in LangChain:**

| Type | Input | Output | Use Cases |
|------|-------|--------|-----------|
| **Chat Models (LLMs)** | Text/Messages | Text/Messages | Chatbots, agents, generation, reasoning |
| **Embedding Models** | Text | Vector (List[float]) | Semantic search, clustering, classification, RAG retrieval |

**Key Abstractions:**
- `BaseChatModel` — common interface: `invoke()`, `stream()`, `batch()`, `abatch()`
- `BaseEmbeddings` — `embed_query()`, `embed_documents()`
- **Model-agnostic code** — swap providers in 1-2 lines
- **Standardized output** — `AIMessage`, `HumanMessage`, `SystemMessage` classes

---

### 1.6.2 Prompts — Structuring Inputs to LLMs

**Core Problem:** Raw string concatenation is fragile, unreadable, and non-reusable.

**LangChain Prompt Components:**

| Feature | Description | Example |
|---------|-------------|---------|
| **PromptTemplate** | Variable interpolation | `"Translate {text} to {language}"` |
| **ChatPromptTemplate** | Message-based (system/human/ai) | Role-aware conversations |
| **FewShotPromptTemplate** | Dynamic few-shot examples | Classification, style transfer |
| **Partial Variables** | Pre-fill some variables | `prompt.partial(language="Hindi")` |
| **Output Parsers** | Structure LLM output | Pydantic, JSON, CSV, Enum |
| **MessagePlaceholder** | Insert chat history/memory | `MessagesPlaceholder("history")` |

**Prompting Techniques Supported:**
- Zero-shot / Few-shot
- Chain-of-Thought (CoT)
- Tree-of-Thought
- ReAct (Reasoning + Acting)
- Structured output (function calling / JSON mode)
- Prompt chaining (output of one → input of next)

**Example: Few-Shot Classification**
```python
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate

examples = [
    {"ticket": "My bill is wrong!", "category": "Billing"},
    {"ticket": "App crashes on login", "category": "Technical"},
    {"ticket": "What are your hours?", "category": "General"},
]

example_prompt = PromptTemplate(
    input_variables=["ticket", "category"],
    template="Ticket: {ticket}\nCategory: {category}"
)

prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="Classify the support ticket:",
    suffix="Ticket: {input}\nCategory:",
    input_variables=["input"]
)
```

---

### 1.6.3 Chains (LCEL) — Composing Pipelines

**Core Concept:** *Chain = Pipeline.* The name "LangChain" comes from this component.

**LangChain Expression Language (LCEL)** — Declarative composition using the **pipe operator (`|`)**:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "Translate {text} to {language}:"
)
model = ChatOpenAI(model="gpt-4o-mini")
parser = StrOutputParser()

# Chain: prompt → model → parser
chain = prompt | model | parser

# Invoke
result = chain.invoke({"text": "Hello world", "language": "Hindi"})
# Output: "नमस्ते दुनिया"
```

**Why LCEL is Powerful:**

| Feature | Description |
|---------|-------------|
| **Streaming** | `chain.stream(...)` — token-by-token output |
| **Async** | `await chain.ainvoke(...)` — non-blocking |
| **Batch** | `chain.batch([...])` — parallel execution |
| **Fallbacks** | `chain.with_fallbacks([backup_chain])` |
| **Retry/Retry Policies** | Built-in resilience |
| **Observability** | LangSmith tracing automatic |
| **Runnable Protocol** | Everything is a `Runnable` — uniform interface |

**Chain Patterns:**

| Pattern | Code | Use Case |
|---------|------|----------|
| **Sequential** | `prompt \| model \| parser` | Linear pipelines |
| **Parallel** | `RunnableParallel({"summary": chain1, "keywords": chain2})` | Multi-output |
| **Branching** | `RunnableBranch((condition, chain_a), chain_b)` | Conditional logic |
| **With History** | `RunnableWithMessageHistory(chain, get_session_history)` | Conversational |

**Under the Hood:** LCEL compiles to **Runnable Sequences** — a DAG of `Runnable` objects (RunnableLambda, RunnableSequence, RunnableParallel, RunnableBranch, RunnableMap).

---

### 1.6.4 Indexes / Retrievers — External Knowledge (RAG)

**Core Problem:** LLMs only know training data. They hallucinate on private/recent data.

**RAG Pipeline in LangChain:**

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐
│  Document   │───▶│  Text        │───▶│  Embedding  │───▶│  Vector Store  │
│  Loaders    │    │  Splitters   │    │  Model      │    │  (Index)       │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘
                                                                    │
                                                                    ▼
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐
│  Answer     │◀───│  LLM +       │◀───│  Retriever  │◀───│  Similarity    │
│  Generation │    │  Prompt      │    │  (Top-K)    │    │  Search        │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘
```

**Key Abstractions:**

| Component | Options | Purpose |
|-----------|---------|---------|
| **Document Loaders** | `PyPDFLoader`, `TextLoader`, `WebBaseLoader`, `NotionDBLoader`, `CSVLoader`, `UnstructuredFileLoader` | Ingest data from any source |
| **Text Splitters** | `RecursiveCharacterTextSplitter`, `TokenTextSplitter`, `MarkdownHeaderTextSplitter`, `PythonCodeTextSplitter` | Chunk documents semantically |
| **Embedding Models** | `OpenAIEmbeddings`, `HuggingFaceEmbeddings`, `CohereEmbeddings`, `OllamaEmbeddings` | Text → Vector |
| **Vector Stores** | `Chroma`, `FAISS`, `Pinecone`, `Weaviate`, `Qdrant`, `Milvus`, `PGVector` | Store & search vectors |
| **Retrievers** | `VectorStoreRetriever`, `MultiQueryRetriever`, `ContextualCompressionRetriever`, `ParentDocumentRetriever` | Fetch relevant chunks |

**Advanced RAG Patterns:**
- **Hybrid Search:** Dense + Sparse (BM25) combination
- **Reranking:** Cross-encoder reranker after retrieval
- **Query Transformation:** Multi-query, HyDE, step-back prompting
- **Modular RAG:** Separate index per document type / tenant

---

### 1.6.5 Memory — Conversation State

**Core Problem:** LLMs are stateless. Each API call is independent.

**Memory Types in LangChain:**

| Memory Type | Description | Trade-off |
|-------------|-------------|-----------|
| **ConversationBufferMemory** | Store full history | Simple, but grows unbounded → token cost |
| **ConversationBufferWindowMemory** | Keep last *k* messages | Fixed window, loses long-term context |
| **ConversationSummaryMemory** | Summarize history periodically | Compresses history, loses detail |
| **ConversationSummaryBufferMemory** | Buffer + summary hybrid | Best of both worlds |
| **Custom Memory** | Implement `BaseChatMemory` | Full control (e.g., entity memory, KG memory) |

**Usage Pattern:**
```python
from langchain.memory import ConversationBufferWindowMemory
from langchain.chains import ConversationChain

memory = ConversationBufferWindowMemory(k=5, return_messages=True)
chain = ConversationChain(llm=llm, memory=memory)

chain.invoke({"input": "Hi, I'm Nitish"})
chain.invoke({"input": "What's my name?"})  # Remembers!
```

**Modern Approach (LangGraph):** Memory is now **explicit state** in the graph — more controllable, inspectable, and persistent.

---

### 1.6.6 Agents — Reasoning + Tools

**Core Distinction:**
> **Chatbot** = LLM + Prompt (passive responder)
> **Agent** = LLM + Prompt + **Reasoning** + **Tools** (active actor)

**Agent Capabilities:**
1. **Reasoning** — Break down complex tasks (Chain-of-Thought, ReAct, Plan-and-Execute)
2. **Tool Use** — Call external functions (search, calculator, code exec, API, DB)
3. **Decision Making** — Choose *which* tool, *when*, with *what arguments*

**Agent Example: "Multiply Delhi's temperature by 3"**

```
User Query: "Multiply today's Delhi temperature by 3"
                │
                ▼
        ┌───────────────┐
        │  REASONING    │  (Chain-of-Thought)
        │  1. Need temp │
        │  2. Need calc │
        └───────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
   ┌─────────┐    ┌──────────┐
   │ Weather │    │Calculator│
   │  Tool   │    │  Tool    │
   └─────────┘    └──────────┘
        │               │
        ▼               ▼
   "25°C"          "25 * 3 = 75"
        │               │
        └───────┬───────┘
                ▼
         "Answer: 75"
```

**LangChain Agent Components:**

| Component | Role |
|-----------|------|
| **Tools** | `@tool` decorated functions — search, calc, Python REPL, SQL, custom APIs |
| **Agent Prompt** | System prompt with tool descriptions + ReAct format |
| **Agent Executor** | Runtime loop: LLM → parse action → execute tool → observe → repeat |
| **Agent Types** | `create_tool_calling_agent` (modern), `create_react_agent`, `create_openai_functions_agent` |

**Modern Agent Stack (2024+):**
- **LangGraph** — Stateful, cyclic graphs for agents (replaces `AgentExecutor`)
- **Prebuilt Agents** — `create_react_agent` (LangGraph version)
- **Human-in-the-loop** — `interrupt()` / `Command(resume=...)` for approval

---

## 1.7 Component Interaction Map

```
┌────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION FLOW                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  USER INPUT                                                            │
│       │                                                                │
│       ▼                                                                │
│  ┌─────────────┐                                                       │
│  │   PROMPT    │  ◀── Formats input, injects history, few-shots        │
│  │  TEMPLATE   │                                                       │
│  └──────┬──────┘                                                       │
│         │                                                              │
│         ▼                                                              │
│  ┌─────────────┐     ┌──────────────┐     ┌──────────────────────┐     │
│  │    MODEL    │────▶│   CHAIN/LCEL │────▶│  OUTPUT PARSER       │     │
│  │  (LLM/Chat) │     │  (Pipeline)  │     │  (Structured/Str)    │     │
│  └──────┬──────┘     └──────┬───────┘     └──────────────────────┘     │
│         │                   │                                          │
│         │          ┌────────┴────────┐                                 │
│         │          ▼                 ▼                                 │
│         │   ┌──────────┐       ┌─────────┐                             │
│         │   │ RETRIEVER│       │  TOOLS  │                             │
│         │   │ (RAG)    │       │ (Agent) │                             │
│         │   └────┬─────┘       └────┬────┘                             │
│         │        │                  │                                  │
│         │        ▼                  ▼                                  │
│         │   ┌─────────────────────────────┐                            │
│         │   │        MEMORY               │                            │
│         │   │  (History / State / Facts)  │                            │
│         │   └─────────────────────────────┘                            │
│         │                    │                                         │
│         ▼                    ▼                                         │
│  ┌─────────────────────────────────────┐                               │
│  │            FINAL OUTPUT             │                               │
│  └─────────────────────────────────────┘                               │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 1.8 Mental Model: How to Think in LangChain

| Don't Think... | Think... |
|----------------|----------|
| "How do I call OpenAI API?" | "Which **Model** implementation do I need?" |
| "How do I format this prompt?" | "Which **PromptTemplate** fits this pattern?" |
| "How do I chain these calls?" | "What **Runnable** sequence expresses this pipeline?" |
| "How do I search my documents?" | "What **Retriever** + **VectorStore** combo?" |
| "How do I remember conversation?" | "What **Memory** type fits my context window?" |
| "How do I make the LLM use tools?" | "What **Agent** + **Tools** architecture?" |

---

## 1.9 Version Note: LangChain v0.2+ / v0.3

The reference **LangChain v0.2** (mid-2024). Key modern patterns:

| Legacy (v0.1) | Modern (v0.2+) |
|---------------|----------------|
| `LLMChain`, `ConversationChain` | **LCEL** (`prompt \| model \| parser`) |
| `AgentExecutor` | **LangGraph** agents (`create_react_agent`) |
| `VectorStoreRetriever` | Same, but with **Runnable** interface |
| `memory=ConversationBufferMemory()` | `RunnableWithMessageHistory` or **LangGraph state** |
| `OpenAI()` / `ChatOpenAI()` | Same, but **`bind_tools()`**, **`with_structured_output()`** |

> **Recommendation:** Write all new code using **LCEL + LangGraph**. Avoid legacy chain classes.

---

## 1.10 Chapter Summary

| Component | One-Liner | Key Class/Function |
|-----------|-----------|-------------------|
| **Models** | Universal interface to any LLM/embedding provider | `ChatOpenAI`, `ChatAnthropic`, `OpenAIEmbeddings` |
| **Prompts** | Templated, reusable, structured input construction | `ChatPromptTemplate`, `FewShotPromptTemplate` |
| **Chains (LCEL)** | Declarative pipeline composition via `|` operator | `RunnableSequence`, `RunnableParallel`, `RunnableBranch` |
| **Indexes/Retrievers** | Load → Split → Embed → Store → Retrieve for RAG | `RecursiveCharacterTextSplitter`, `Chroma`, `VectorStoreRetriever` |
| **Memory** | Persist conversation state across turns | `ConversationBufferWindowMemory`, `RunnableWithMessageHistory` |
| **Agents** | LLM that reasons, decides, and acts via tools | `create_tool_calling_agent`, **LangGraph** `StateGraph` |

---

## 1.11 What's Next

This chapter established the **conceptual foundation**. The next chapters will go **deep on each component** with:

1. **Models** — Chat models, embeddings, multimodal, custom wrappers, streaming, async
2. **Prompts** — Advanced templates, output parsers, few-shot optimization, prompt versioning
3. **Chains/LCEL** — Streaming, batch, fallbacks, retries, observability, custom Runnables
4. **RAG/Indexes** — Loaders, splitters, embedding models, vector stores, hybrid search, reranking, modular RAG
5. **Memory** — Types, LangGraph state, LangMem (long-term), persistence
6. **Agents** — Tool design, ReAct vs tool-calling, LangGraph agents, multi-agent, human-in-the-loop

Each chapter will include **runnable code patterns**, **architecture diagrams**, and **production considerations**.

---

## 1.12 References

1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0
2. **LangChain Docs:** https://python.langchain.com/docs/
3. **LangGraph Docs:** https://langchain-ai.github.io/langgraph/
4. **LangSmith (Observability):** https://docs.smith.langchain.com/

---

*End of Chapter 1 — Introduction & Core Components*