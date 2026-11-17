# LangGraph Lecture 3: LangChain vs LangGraph — When to Use Which

## 1. Goals & Learning Outcomes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             LECTURE 3 GOALS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GOAL 1: DEEP INTUITION — Why does LangGraph exist?                         │
│  ─────────────────────────────────────────────────                          │
│  What problem does LangChain NOT solve that requires LangGraph?             │
│                                                                             │
│  GOAL 2: TECHNICAL OVERVIEW — What exactly is LangGraph?                    │
│  ────────────────────────────────────────────────────                       │
│  Core concepts: control flow, state, events, fault tolerance, HITL          │
│                                                                             │
│  GOAL 3: DECISION FRAMEWORK — LangChain vs LangGraph                        │
│  ──────────────────────────────────────────────────                         │
│  Given ANY project, automatically know which to use (or both)               │
│                                                                             │
│  OUTCOME: Interview-ready understanding + practical project selection       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. LangChain Recap (The Foundation)

### 2.1 What is LangChain?

> **LangChain** is an open-source library designed to simplify building LLM-based applications by providing modular building blocks.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGCHAIN CORE BUILDING BLOCKS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. MODELS (Model Component)                                                │
│  ─────────────────────────                                                  │
│  • Unified interface for ANY LLM provider                                   │
│  • OpenAI, Anthropic, HuggingFace, Ollama, local models                     │
│  • Swap models with minimal code changes                                    │
│  • ChatOpenAI, ChatAnthropic, HuggingFaceHub, OllamaLLM                     │
│                                                                             │
│  2. PROMPTS (Prompt Component)                                              │
│  ────────────────────────                                                   │
│  • PromptTemplate, ChatPromptTemplate, FewShotPromptTemplate                │
│  • Partial formatting, variable injection                                   │
│  • MessagePromptTemplates (System, Human, AI)                               │
│                                                                             │
│  3. RETRIEVERS (Retriever Component)                                        │
│  ────────────────────────────                                               │
│  • Fetch relevant docs from vector stores / knowledge bases                 │
│  • Strategies: similarity, MMR, similarity_score_threshold                  │
│  • Enables RAG applications                                                 │
│                                                                             │
│  4. CHAINS (The Core Abstraction)  ◄── "CHAIN" IN LANGCHAIN                 │
│  ─────────────────────────────                                              │
│  • Compose components: Prompt → Model → Parser → Next...                    │
│  • Automatic output→input passing between steps                             │
│  • LCEL (LangChain Expression Language): `|` operator                       │
│  • RunnableSequence, RunnableParallel, RunnableBranch                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 What Can You Build with LangChain?

| Application Type | Example | LangChain Components Used |
|-----------------|---------|---------------------------|
| **Conversational** | Chatbots, Text Summarizers | Prompt + Model + Memory |
| **RAG Systems** | Document Q&A, Knowledge Base | Retriever + Prompt + Model |
| **Structured Output** | Data Extraction, Classification | Prompt + Model + OutputParser |
| **Tool Use** | Function Calling Agents | Model.bind_tools + AgentExecutor |
| **Chains** | Multi-step pipelines | LCEL: `prompt | model | parser | next_chain` |

### 2.3 LangChain Chain Example

```python
# Classic LangChain LCEL Chain
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_template("Summarize: {text}")
parser = StrOutputParser()

# Chain: input → prompt → model → parser → output
chain = prompt | llm | parser

result = chain.invoke({"text": "Long document..."})
# Output: "Summary..."
```

---

## 3. The Problem: Where LangChain Falls Short

### 3.1 Automated Workflows & Their Complexity

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AUTOMATED WORKFLOW COMPLEXITY                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WORKFLOW = Sequence of steps to achieve a goal                             │
│  ──────────────────────────────────────────────                             │
│                                                                             │
│  SIMPLE (Linear):         A → B → C → D                                     │
│  ─────────────────                                                          │
│  • Prompt → Model → Parser → Output                                         │
│  • LangChain CHAINS handle this perfectly                                   │
│                                                                             │
│  COMPLEX (Non-Linear):                                                      │
│  ────────────────────                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  CONDITIONAL BRANCHING                                              │    │
│  │  A → (if X) → B → D                                                 │    │
│  │    └→ (else) → C → D                                                │    │
│  │                                                                     │    │
│  │  LOOPS / ITERATION                                                  │    │
│  │  A → B → C → (check) → B → C → ... → D                              │    │
│  │                                                                     │    │
│  │  HUMAN-IN-THE-LOOP                                                  │    │
│  │  A → B → [HUMAN APPROVAL] → C → D                                   │    │
│  │                                                                     │    │
│  │  PARALLEL EXECUTION                                                 │    │
│  │  A → [B, C, D] parallel → E                                         │    │
│  │                                                                     │    │
│  │  EVENT-DRIVEN / ASYNC                                               │    │
│  │  A → wait for event → B → C                                         │    │
│  │                                                                     │    │
│  │  FAULT TOLERANCE / RETRY                                            │    │
│  │  A → B (fails) → retry B → C                                        │    │
│  │                                                                     │    │
│  │  NESTED WORKFLOWS                                                   │    │
│  │  Main: A → SubWorkflow(X→Y→Z) → B                                   │    │
│  │                                                                     │    │
│  │  MULTI-AGENT COORDINATION                                           │    │
│  │  Agent1 ↔ Agent2 ↔ Agent3 (collaborative)                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  LANGCHAIN CHAINS CANNOT NATIVELY HANDLE THESE!                             │
│  ────────────────────────────────────────────────                           │
│  • Chains are LINEAR (mostly)                                               │
│  • No built-in state management                                             │
│  • No persistence / checkpointing                                           │
│  • No human-in-the-loop primitives                                          │
│  • No conditional routing (RunnableBranch is limited)                       │
│  • No loops with state accumulation                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 The Control Flow Problem

```python
# LANGCHAIN: Trying to do conditional logic (awkward)
from langchain_core.runnables import RunnableBranch, RunnableLambda

chain = (
    first_step
    | RunnableBranch(
        (lambda x: x["condition"] == "A", chain_a),
        (lambda x: x["condition"] == "B", chain_b),
        default_chain
    )
)

# PROBLEMS:
# 1. Condition must be in the input/state already
# 2. No shared state between branches
# 3. Can't loop back with accumulated state
# 4. No persistence - if crash, restart from beginning
# 5. Can't pause for human input mid-chain
# 6. Debugging: no visibility into intermediate states
```

### 3.3 The State Management Problem

```
LANGCHAIN STATE HANDLING:
─────────────────────────────────────────────────────────────────────────────
• Chains are STATeless by default
• To share data: pass explicitly in input dict
• No automatic state merging between steps
• Memory = separate component (ConversationBufferMemory, etc.)
• Not designed for complex multi-turn stateful workflows
• Can't "rewind" or "time travel" to previous states
```

### 3.4 The Event-Driven / Async Problem

```
LANGCHAIN EXECUTION MODEL:
─────────────────────────────────────────────────────────────────────────────
• Synchronous by default (invoke)
• Async support (ainvoke) but no event-driven architecture
• Can't: "Wait for external webhook, then continue"
• Can't: "Stream partial results while background task runs"
• Can't: "Pause at checkpoint, resume days later"
```

---

## 4. Enter LangGraph: The Solution

### 4.1 What is LangGraph?

> **LangGraph** is a low-level orchestration framework for building stateful, long-running, multi-agent workflows. It solves the control flow, state, and persistence problems that LangChain chains cannot.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGGRAPH CORE PRIMITIVES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GRAPH = STATE + NODES + EDGES                                              │
│  ─────────────────────────────                                              │
│                                                                             │
│  STATE (Shared Context)                                                     │
│  ─────────────────                                                          │
│  • TypedDict / Pydantic model defining all data                             │
│  • Passed between ALL nodes automatically                                   │
│  • Immutable updates: nodes return partial → merged                         │
│  • Persisted via checkpointer                                               │
│                                                                             │
│  NODES (Processing Units)                                                   │
│  ──────────────────                                                         │
│  • Functions: (state) → partial_state_update                                │
│  • Can be: LLM calls, tool execution, logic, human interrupt                │
│  • Access full state, return only changes                                   │
│                                                                             │
│  EDGES (Flow Control)                                                       │
│  ──────────────────                                                         │
│  • Direct: A → B (always)                                                   │
│  • Conditional: A → router(state) → B or C                                  │
│  • START → entry, node → END                                                │
│  • Loops: conditional edge back to earlier node                             │
│                                                                             │
│  CHECKPOINTER (Persistence)                                                 │
│  ─────────────────────                                                      │
│  • Saves state after each node                                              │
│  • Enables: resume, retry, time travel, HITL                                │
│  • Backends: Memory, SQLite, PostgreSQL                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 LangGraph Capabilities Mapping

| Complex Workflow Need | LangChain | LangGraph |
|----------------------|-----------|-----------|
| **Conditional Branching** | `RunnableBranch` (limited) | Native `add_conditional_edges` |
| **Loops with State** | ❌ Very hard | Native (edge back to node) |
| **Human-in-the-Loop** | ❌ Manual | Native `interrupt()` + `Command(resume=...)` |
| **Persistence/Resume** | ❌ Manual | Native checkpointer |
| **Parallel Execution** | `RunnableParallel` | Native (fan-out/fan-in edges) |
| **Event-Driven/Async** | ❌ Limited | Native streaming + interrupts |
| **Fault Tolerance** | ❌ Try/catch | Native retry + checkpoint rollback |
| **Nested Workflows** | ❌ Manual | Native subgraphs |
| **Multi-Agent** | ❌ Manual | Native (multiple graphs + supervisor) |
| **Time Travel/Debug** | ❌ No | Native (checkpoint history) |

---

## 5. LangChain vs LangGraph: Detailed Comparison

### 5.1 Architectural Differences

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURAL COMPARISON                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LANGCHAIN (Component Library)                    LANGGRAPH (Runtime)       │
│  ─────────────────────────────                    ──────────────────        │
│                                                                             │
│  Purpose:                                         Purpose:                  │
│  • Provide LLM building blocks                   • Orchestrate workflows    │
│  • Unified interfaces                            • Manage state & flow      │
│  • Simplify common patterns                      • Enable complex patterns  │
│                                                                             │
│  Abstraction Level:                              Abstraction Level:         │
│  HIGH (opinionated)                              LOW (flexible)             │
│  • Chains, Agents, Memories                      • Graphs, Nodes, Edges     │
│  • Pre-built patterns                            • Build your own patterns  │
│                                                                             │
│  Control Flow:                                   Control Flow:              │
│  IMPLICIT (chain defines it)                     EXPLICIT (you define it)   │
│  • Linear by default                             • Graph defines it         │
│  • Branching via RunnableBranch                  • Conditional edges        │
│  • Loops: manual recursion                       • Loops: natural edges     │
│                                                                             │
│  State Management:                               State Management:          │
│  IMPLICIT (pass in dict)                         EXPLICIT (Typed State)     │
│  • Input dict → Output dict                      • Shared State object      │
│  • Memory = separate addon                       • State = first-class      │
│  • No auto-merging                               • Auto-merge updates       │
│                                                                             │
│  Persistence:                                    Persistence:               │
│  Manual (save/load yourself)                     Native (checkpointer)      │
│                                                                             │
│  Debugging:                                      Debugging:                 │
│  Logs, callbacks                                 Time travel, replay        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Code Comparison: Same Task

#### Task: *Research Agent - Search → Summarize → (if needed) Deep Dive → Report*

```python
# ═══════════════════════════════════════════════════════════════════════
# LANGCHAIN APPROACH (Linear Chain with Awkward Branching)
# ═══════════════════════════════════════════════════════════════════════

from langchain_core.runnables import RunnableBranch, RunnableLambda
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini")

# Step 1: Search
search_chain = (
    ChatPromptTemplate.from_template("Generate search queries for: {topic}")
    | llm
    | StrOutputParser()
)

# Step 2: Summarize (always runs)
summarize_chain = (
    ChatPromptTemplate.from_template("Summarize these results: {results}")
    | llm
    | StrOutputParser()
)

# Step 3: Conditional deep dive (CLUNKY)
def needs_deep_dive(inputs):
    # Must extract from previous outputs manually
    return "deep" if "complex" in inputs.get("summary", "").lower() else "shallow"

deep_dive_chain = (
    ChatPromptTemplate.from_template("Deep research on: {topic}")
    | llm
    | StrOutputParser()
)

final_chain = (
    {"topic": lambda x: x["topic"]}
    | RunnableLambda(lambda x: {"topic": x["topic"], "queries": search_chain.invoke(x)})
    | RunnableLambda(lambda x: {"topic": x["topic"], "results": search_tool(x["queries"])})
    | RunnableLambda(lambda x: {"topic": x["topic"], "summary": summarize_chain.invoke(x)})
    | RunnableBranch(
        (needs_deep_dive, 
         RunnableLambda(lambda x: {"topic": x["topic"], "deep_dive": deep_dive_chain.invoke(x)})
         | RunnableLambda(lambda x: {"report": x["summary"] + "\n\n" + x["deep_dive"]})
        ),
        RunnableLambda(lambda x: {"report": x["summary"]})
    )
)

# PROBLEMS:
# - State passing is manual and error-prone
# - Can't loop back to search with new queries
# - No persistence - crash = restart
# - Can't pause for human review
# - Debugging: no intermediate state visibility


# ═══════════════════════════════════════════════════════════════════════
# LANGGRAPH APPROACH (Natural Graph with Explicit State)
# ═══════════════════════════════════════════════════════════════════════

from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class ResearchState(TypedDict):
    topic: str
    queries: list[str]
    search_results: list[dict]
    summary: str
    deep_dive: str
    report: str
    needs_deep_dive: bool
    iteration: int

def generate_queries(state: ResearchState):
    # LLM generates search queries
    queries = llm.invoke(f"Generate 3 search queries for: {state['topic']}")
    return {"queries": parse_queries(queries), "iteration": state.get("iteration", 0) + 1}

def search(state: ResearchState):
    results = search_tool(state["queries"])
    return {"search_results": results}

def summarize(state: ResearchState):
    summary = llm.invoke(f"Summarize: {state['search_results']}")
    # Decide if deep dive needed
    needs_deep = "complex" in summary.lower() or len(state["search_results"]) > 10
    return {"summary": summary, "needs_deep_dive": needs_deep}

def deep_dive(state: ResearchState):
    deep = llm.invoke(f"Deep research on: {state['topic']}")
    return {"deep_dive": deep}

def compile_report(state: ResearchState):
    if state["needs_deep_dive"]:
        report = state["summary"] + "\n\nDEEP DIVE:\n" + state["deep_dive"]
    else:
        report = state["summary"]
    return {"report": report}

# BUILD GRAPH
graph = StateGraph(ResearchState)

graph.add_node("generate_queries", generate_queries)
graph.add_node("search", search)
graph.add_node("summarize", summarize)
graph.add_node("deep_dive", deep_dive)
graph.add_node("compile_report", compile_report)

graph.add_edge(START, "generate_queries")
graph.add_edge("generate_queries", "search")
graph.add_edge("search", "summarize")

# CONDITIONAL EDGE - Natural branching!
graph.add_conditional_edges(
    "summarize",
    lambda s: "deep_dive" if s["needs_deep_dive"] else "compile_report",
    {"deep_dive": "deep_dive", "compile_report": "compile_report"}
)

graph.add_edge("deep_dive", "compile_report")
graph.add_edge("compile_report", END)

# COMPILE WITH PERSISTENCE
app = graph.compile(checkpointer=MemorySaver())

# RUN
result = app.invoke({"topic": "LangGraph vs LangChain"}, config={"thread_id": "research-1"})

# BENEFITS:
# ✓ Natural conditional branching
# ✓ Easy to add loop: compile_report → generate_queries (for iterative research)
# ✓ Full persistence - resume anytime
# ✓ Human-in-the-loop: add interrupt before compile_report
# ✓ Debugging: inspect state at any node
# ✓ Streaming: watch progress in real-time
```

---

## 6. When to Use What: Decision Framework

### 6.1 The Golden Rule

> **USE LANGCHAIN FOR COMPONENTS, LANGGRAPH FOR ORCHESTRATION**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE HAND-IN-HAND RELATIONSHIP                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LANGCHAIN PROVIDES:                    LANGGRAPH ORCHESTRATES:             │
│  ──────────────────────────             ────────────────────────            │
│  • Chat Models (ChatOpenAI, etc.)       • Graph Structure (nodes/edges)     │
│  • Prompt Templates                     • State Management                  │
│  • Output Parsers                       • Conditional Routing               │
│  • Retrievers / Vector Stores           • Loops & Iteration                 │
│  • Document Loaders                     • Persistence (Checkpointer)        │
│  • Text Splitters                       • Human-in-the-Loop (Interrupt)     │
│  • Tools (Built-in + Custom)            • Streaming & Async                 │
│  • Agent Abstractions (ReAct, etc.)     • Subgraphs & Multi-Agent           │
│                                                                             │
│  YOU USE BOTH TOGETHER:                                                     │
│  ──────────────────────────                                                 │
│  LangGraph nodes CALL LangChain components                                  │
│                                                                             │
│  Example Node:                                                              │
│  ───────────                                                                │
│  def research_node(state):                                                  │
│      # Uses LangChain components!                                           │
│      prompt = ChatPromptTemplate.from_template(...)  # LangChain            │
│      model = ChatOpenAI(model="gpt-4o")                    # LangChain      │
│      chain = prompt | model | StrOutputParser()           # LangChain       │
│      result = chain.invoke({"topic": state["topic"]})      # LangChain      │
│      return {"findings": result}                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Decision Matrix

| Scenario | Use | Reason |
|----------|-----|--------|
| **Simple Chatbot** | LangChain | Linear: prompt → model → output |
| **Basic RAG** | LangChain | Retrieve → prompt → model |
| **Summarization Pipeline** | LangChain | Linear chain of transforms |
| **Structured Extraction** | LangChain | Prompt → model → parser |
| **Single Agent (ReAct)** | LangChain AgentExecutor | Pre-built pattern works |
| **Conditional Workflow** | **LangGraph** | Need `if/else` routing |
| **Iterative/Looping** | **LangGraph** | Need loops with state accumulation |
| **Human Approval Required** | **LangGraph** | Native `interrupt()` support |
| **Long-Running (Hours/Days)** | **LangGraph** | Persistence + resume |
| **Multi-Agent System** | **LangGraph** | Coordination, shared state |
| **Event-Driven/Async** | **LangGraph** | Streaming, interrupts, pauses |
| **Fault Tolerance Needed** | **LangGraph** | Checkpoint rollback |
| **Debugging/Time Travel** | **LangGraph** | Checkpoint history |
| **Nested Workflows** | **LangGraph** | Subgraphs |

### 6.3 Decision Flowchart

```
NEW PROJECT: Which framework?
│
├─► Is it a SIMPLE LINEAR PIPELINE?
│   │   (Input → Process → Output, no branching, no loops)
│   ├─► YES → LangChain (LCEL Chains)
│   └─► NO → Continue...
│
├─► Does it need CONDITIONAL BRANCHING?
│   │   (Different paths based on intermediate results)
│   ├─► YES → LangGraph
│   └─► NO → Continue...
│
├─► Does it need LOOPS / ITERATION?
│   │   (Repeat steps with accumulated state)
│   ├─► YES → LangGraph
│   └─► NO → Continue...
│
├─► Does it need HUMAN-IN-THE-LOOP?
│   │   (Pause for approval, then continue)
│   ├─► YES → LangGraph
│   └─► NO → Continue...
│
├─► Does it run LONG (hours/days) or need RESUME?
│   │   (Persist state, survive crashes)
│   ├─► YES → LangGraph
│   └─► NO → Continue...
│
├─► Does it need MULTI-AGENT COORDINATION?
│   ├─► YES → LangGraph
│   └─► NO → Continue...
│
├─► Is it a STANDARD AGENT PATTERN?
│   │   (ReAct, Plan-Execute, etc.)
│   ├─► YES → LangChain AgentExecutor (simpler)
│   └─► NO → LangGraph (custom control flow)
│
└─► DEFAULT: Start with LangChain, migrate to LangGraph when pain points appear
```

---

## 7. Advanced LangGraph Concepts

### 7.1 Control Flow Complexity

```python
# Complex routing patterns in LangGraph

# 1. MULTI-WAY BRANCHING
def router(state):
    if state["score"] > 0.9:
        return "auto_approve"
    elif state["score"] > 0.5:
        return "human_review"
    else:
        return "reject"

graph.add_conditional_edges("evaluate", router, {
    "auto_approve": "approve_node",
    "human_review": "hitl_node",
    "reject": "reject_node"
})

# 2. PARALLEL FAN-OUT / FAN-IN
graph.add_edge("start", "parallel_1")
graph.add_edge("start", "parallel_2")
graph.add_edge("start", "parallel_3")
graph.add_edge("parallel_1", "merge")
graph.add_edge("parallel_2", "merge")
graph.add_edge("parallel_3", "merge")

# 3. LOOPS WITH STATE ACCUMULATION
def should_continue(state):
    return "continue" if state["iteration"] < 5 and not state["done"] else "exit"

graph.add_conditional_edges("process", should_continue, {
    "continue": "process",
    "exit": "finalize"
})

# 4. DYNAMIC ROUTING BASED ON TOOL RESULTS
def route_after_tool(state):
    last_result = state["tool_results"][-1]
    if last_result.get("error"):
        return "handle_error"
    elif last_result.get("needs_clarification"):
        return "ask_clarification"
    else:
        return "continue_processing"
```

### 7.2 State Management Deep Dive

```python
# State design patterns

# PATTERN 1: Annotated Fields (Auto-merging)
from typing import Annotated
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # Auto-appends lists
    documents: Annotated[list, operator.add]  # Custom merge
    metadata: dict  # Replaced entirely

# PATTERN 2: Nested State (for subgraphs)
class ParentState(TypedDict):
    user_query: str
    subagent_state: "SubagentState"  # Passed to subgraph
    final_answer: str

# PATTERN 3: State with Defaults
from langgraph.graph import StateGraph

class State(TypedDict, total=False):  # total=False = all optional
    topic: str
    iteration: int
    max_iterations: int
    results: list

# Defaults handled in first node
def init_node(state):
    return {
        "iteration": state.get("iteration", 0),
        "max_iterations": state.get("max_iterations", 5),
        "results": state.get("results", [])
    }
```

### 7.3 Human-in-the-Loop (HITL)

```python
from langgraph.types import interrupt, Command

# HITL Node
def human_approval(state):
    # PAUSES EXECUTION - saves checkpoint
    decision = interrupt({
        "type": "approval_request",
        "data": {
            "action": state["proposed_action"],
            "risk_level": state["risk_level"],
            "details": state["details"]
        }
    })
    
    # RESUMES HERE when Command(resume=...) received
    return {"human_decision": decision}

# In graph:
graph.add_node("human_approval", human_approval)
graph.add_edge("propose_action", "human_approval")
graph.add_conditional_edges("human_approval", 
    lambda s: "execute" if s["human_decision"]["approved"] else "cancel")

# RESUMING FROM OUTSIDE:
# app.invoke(Command(resume={"approved": True, "notes": "Looks good"}), config)
```

### 7.4 Fault Tolerance & Retry

```python
# Retry policy at node level
from langgraph.graph import RetryPolicy

graph.add_node(
    "unreliable_tool",
    unreliable_tool_node,
    retry=RetryPolicy(
        max_attempts=3,
        backoff_factor=2.0,  # 1s, 2s, 4s
        retry_on=(ConnectionError, TimeoutError)
    )
)

# Checkpoint rollback on failure
try:
    result = app.invoke(input, config)
except Exception as e:
    # Rollback to previous checkpoint
    checkpointer = app.checkpointer
    previous = checkpointer.get(config)  # Get last good state
    # Resume from there with modified input
    result = app.invoke(modified_input, config)
```

### 7.5 Nested Workflows (Subgraphs)

```python
# Subgraph = Reusable workflow component
subgraph = StateGraph(SubState)
subgraph.add_node("step1", step1_node)
subgraph.add_node("step2", step2_node)
subgraph.add_edge(START, "step1")
subgraph.add_edge("step1", "step2")
subgraph.add_edge("step2", END)
subgraph_app = subgraph.compile()

# Use as node in parent graph
def call_subgraph(state):
    result = subgraph_app.invoke({"input": state["sub_input"]})
    return {"sub_output": result["output"]}

parent_graph.add_node("subworkflow", call_subgraph)

# OR: Native subgraph support (LangGraph 0.2+)
parent_graph.add_node("subworkflow", subgraph_app)  # Direct!
```

### 7.6 Observability (LangSmith Integration)

```python
# LangSmith tracing (automatic with LangGraph)
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_key"
os.environ["LANGCHAIN_PROJECT"] = "langgraph-demo"

# Traces show:
# • Graph structure visualization
# • Node execution timeline
# • State at each node
# • Token usage per node
# • Latency per node
# • Errors with full context

# LangSmith Engine: Auto-detects issues
# • "Node X takes 90% of time"
# • "State grows unbounded in loop"
# • "Repeated tool calls with same args"
```

---

## 8. Migration Path: LangChain → LangGraph

### 8.1 When to Migrate

```
MIGRATION TRIGGERS:
─────────────────────────────────────────────────────────────────────────────
✓ Chain becoming unwieldy (too many RunnableBranch)
✓ Need loops with state accumulation
✓ Need human approval mid-workflow
✓ Need persistence / resume capability
✓ Debugging becomes impossible (no state visibility)
✓ Multi-agent requirements emerge
✓ Fault tolerance requirements increase
✓ Team needs time-travel debugging
```

### 8.2 Migration Strategy

```python
# PHASE 1: Extract chain steps as independent functions
# LangChain: chain = step1 | step2 | step3 | step4

# Becomes:
def step1_fn(input): return step1.invoke(input)
def step2_fn(input): return step2.invoke(input)
def step3_fn(input): return step3.invoke(input)
def step4_fn(input): return step4.invoke(input)

# PHASE 2: Define unified state
class WorkflowState(TypedDict):
    step1_output: str
    step2_output: str
    step3_output: str
    final_output: str

# PHASE 3: Wrap as LangGraph nodes
def step1_node(state): return {"step1_output": step1_fn(state["input"])}
def step2_node(state): return {"step2_output": step2_fn(state["step1_output"])}
# ...

# PHASE 4: Build graph with edges
graph = StateGraph(WorkflowState)
graph.add_node("step1", step1_node)
graph.add_node("step2", step2_node)
graph.add_node("step3", step3_node)
graph.add_node("step4", step4_node)
graph.add_edge(START, "step1")
graph.add_edge("step1", "step2")
graph.add_edge("step2", "step3")
graph.add_edge("step3", "step4")
graph.add_edge("step4", END)

# PHASE 5: Add LangGraph superpowers incrementally
# - Add checkpointer
# - Add conditional edge where needed
# - Add interrupt for HITL
# - Add retry policy
```

---

## 9. Quick Reference: LangChain vs LangGraph Cheat Sheet

```python
# ============================================
# QUICK DECISION: LANGCHAIN OR LANGGRAPH?
# ============================================

# USE LANGCHAIN WHEN:
# ──────────────────
# ✓ Linear pipeline (A → B → C)
# ✓ Simple RAG / Chat / Summarization
# ✓ Standard ReAct Agent (use AgentExecutor)
# ✓ Need: Models, Prompts, Parsers, Retrievers, Tools
# ✓ Prototyping simple flows

# USE LANGGRAPH WHEN:
# ───────────────────
# ✓ Conditional branching (if/else/elif)
# ✓ Loops with state accumulation
# ✓ Human-in-the-loop (pause/resume)
# ✓ Long-running + persistence needed
# ✓ Multi-agent coordination
# ✓ Event-driven / async execution
# ✓ Fault tolerance + retry + rollback
# ✓ Time-travel debugging
# ✓ Nested workflows (subgraphs)

# THEY WORK TOGETHER:
# ───────────────────
# LangGraph nodes USE LangChain components
# 
# def my_node(state):
#     # LangChain inside LangGraph node
#     chain = prompt | model | parser
#     result = chain.invoke({"data": state["data"]})
#     return {"result": result}

# MIGRATION PATH:
# ───────────────
# 1. Break chain into functions
# 2. Define unified State
# 3. Wrap functions as nodes
# 4. Build graph with edges
# 5. Add LangGraph features incrementally

# MEMORY:
# ───────
# LangChain: ConversationBufferMemory (separate)
# LangGraph: checkpointer (built-in) + store (long-term)

# DEBUGGING:
# ──────────
# LangChain: callbacks, verbose logging
# LangGraph: checkpoint history, time travel, LangSmith visual traces
```

---

## 10. Key Takeaways

| Concept | Essence |
|---------|---------|
| **LangChain** | Component library: Models, Prompts, Parsers, Retrievers, Tools, Chains |
| **LangGraph** | Orchestration runtime: Graphs, State, Nodes, Edges, Checkpointers |
| **Core Problem** | LangChain chains are linear; can't do loops, branching, HITL, persistence |
| **Solution** | LangGraph: Explicit control flow via graph edges + shared state |
| **Relationship** | **NOT competitors** — LangGraph orchestrates LangChain components |
| **Decision Rule** | Linear → LangChain; Complex/Stateful/Long-running → LangGraph |
| **Migration** | Extract chain steps → Define State → Wrap as nodes → Build graph → Add features |

---

## 12. Resources & References

### Videos
- **Playlist:** https://www.youtube.com/watch?v=yC36gN-rqjo&list=PLKnIA16_RmvYsvB8qkUQuJmJNuiCUJFPL

### Official Documentation
- **LangGraph Overview:** https://docs.langchain.com/oss/python/langgraph/overview
- **LangGraph Quickstart:** https://docs.langchain.com/oss/python/langgraph/quickstart
- **Graph API:** https://docs.langchain.com/oss/python/langgraph/graph-api
- **Persistence:** https://docs.langchain.com/oss/python/langgraph/persistence
- **Human-in-the-Loop:** https://docs.langchain.com/oss/python/langgraph/interrupts
- **Streaming:** https://docs.langchain.com/oss/python/langgraph/streaming
