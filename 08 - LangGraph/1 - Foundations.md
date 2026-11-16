# LangGraph Lecture 1: Foundations of Agentic AI & LangGraph Overview

## 1. Why LangGraph? Why Now?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY AGENTIC AI NOW?                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  REASON 1: TIMING                                                           │
│  ─────────────────                                                          │
│  • 2022: ChatGPT arrives → Generative AI explosion                          │
│  • 2023-2024: GenAI matures (RAG, tools, structured output)                 │
│  • 2025: Agentic AI emerges as NEXT BIG THING                               │
│  • All top CEOs/CTOs hyping: "Agents will change everything"                │
│  • Market demand: Every 3rd comment = "Please teach LangGraph!"             │
│                                                                             │
│  REASON 2: BUILD-UP                                                         │
│  ────────────────                                                           │
│  ML → Deep Learning → LangChain (GenAI) → LangGraph (Agentic AI)            │
│  Sequential curriculum prepares you for this moment                         │
│                                                                             │
│  REASON 3: INDUSTRY DEMAND                                                  │
│  ────────────────────────                                                   │
│  • Companies actively hiring for agentic AI skills                          │
│  • LangGraph becoming THE orchestration standard                            │
│  • Early adopters gain massive career advantage                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
---

## 2. Generative AI vs Agentic AI: The Fundamental Distinction

### 2.1 Generative AI Recap

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GENERATIVE AI (GenAI)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DEFINITION:                                                                │
│  ────────────                                                               │
│  "A transformative technology (3 years old) that generates content:         │
│   text, code, images, audio, video — from patterns in training data"        │
│                                                                             │
│  TWO EMOTIONS PEOPLE HAVE:                                                  │
│  ────────────────────────────                                               │
│  1. EXCITEMENT: "It's incredibly capable!"                                  │
│     • Writes code, essays, emails                                           │
│     • Summarizes, translates, creates                                       │
│     • Passes exams, assists daily work                                      │
│                                                                             │
│  2. FRUSTRATION: "But it can't DO things for me"                            │
│     • Can't send emails, can't book meetings                                │
│     • Can't execute code, can't access my files                             │
│     • Can't remember context across sessions                                │
│     • Hallucinates, needs constant supervision                              │
│                                                                             │
│  CORE LIMITATION:                                                           │
│  ────────────────                                                           │
│  GenAI = PASSIVE GENERATOR                                                  │
│  Input → Model → Output (one-shot, stateless)                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 The Practical Scenario: Evolution from GenAI to Agentic AI

**Scenario:** *Building a Travel Planning Assistant*

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVOLUTION: GenAI → Agentic AI                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  VERSION 1: PURE GENERATIVE AI                                              │
│  ──────────────────────────────────                                         │
│  User: "Plan a 5-day Tokyo trip"                                            │
│  GenAI: "Here's an itinerary: Day 1: Shibuya... Day 2: Asakusa..."          │
│                                                                             │
│  PROBLEMS:                                                                  │
│  ✗ No real-time data (flights, hotels, weather)                             │
│  ✗ Can't book anything                                                      │
│  ✗ Can't adapt if flight delayed                                            │
│  ✗ Forgets preferences next session                                         │
│  ✗ Hallucinates restaurant names                                            │
│                                                                             │
│  ────────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  VERSION 2: GenAI + TOOLS (RAG + Function Calling)                          │
│  ─────────────────────────────────────────────────                          │
│  Tools: search_flights, search_hotels, get_weather, send_email              │
│                                                                             │
│  User: "Plan Tokyo trip"                                                    │
│  Agent: [search_flights] → [search_hotels] → [get_weather] → [itinerary]    │
│                                                                             │
│  BETTER BUT STILL:                                                          │
│  ✗ Fixed workflow (always same tool order)                                  │
│  ✗ No recovery if tool fails                                                │
│  ✗ No human approval for bookings                                           │
│  ✗ Context lost between turns                                               │
│  ✗ Can't handle "Actually, change hotel to Kyoto"                           │
│                                                                             │
│  ────────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  VERSION 3: AGENTIC AI (LangGraph)                                          │
│  ──────────────────────────────────                                         │
│  • DYNAMIC PLANNING: LLM decides which tools, in what order                 │
│  • STATE MANAGEMENT: Remembers everything (flights, prefs, decisions)       │
│  • HUMAN-IN-THE-LOOP: Pauses for approval before booking                    │
│  • RECOVERY: Retries, alternatives, graceful degradation                    │
│  • PERSISTENCE: Resumes after crash, continues next session                 │
│  • BRANCHING: "If flight > $1000, suggest alternatives"                     │
│  • MEMORY: Learns preferences across trips                                  │
│                                                                             │
│  RESULT: Autonomous assistant that ACTS, not just suggests                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Core Distinction Summary

| Aspect | Generative AI | Agentic AI |
|--------|--------------|------------|
| **Paradigm** | Generate content | Execute goals |
| **Flow** | Input → Output (one-shot) | Loop: Plan → Act → Observe → Repeat |
| **State** | Stateless | Stateful (persistent memory) |
| **Tools** | None or fixed | Dynamic selection & chaining |
| **Control** | User drives | Agent drives (with oversight) |
| **Recovery** | None | Retry, fallback, escalate |
| **Human Role** | Supervise every output | Approve high-stakes actions |
| **Analogy** | Smart intern | Autonomous employee |

---

## 3. What is LangGraph?

### 3.1 Core Definition

> **LangGraph** is a **low-level orchestration framework and runtime** for **long-running, stateful agents**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGGRAPH POSITIONING                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LANGCHAIN ECOSYSTEM LAYERS:                                                │
│  ───────────────────────────                                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  DEEP AGENTS (Highest)                                              │    │
│  │  Agent harness: planning, subagents, filesystem, context mgmt       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ▲                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  LANGCHAIN (Framework)                                              │    │
│  │  Abstractions for: models, tools, agent loops, integrations         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ▲                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  LANGGRAPH (Runtime) ◄── YOU ARE HERE                               │    │
│  │  Orchestration: durable execution, streaming, HITL, persistence     │    │
│  │  Graph = State + Nodes + Edges                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ▲                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  LANGSMITH (Platform)                                               │    │
│  │  Observability, evaluation, deployment, tracing                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  KEY INSIGHT:                                                               │
│  ───────────                                                                │
│  LangGraph does NOT prescribe prompts or agent architecture.                │
│  It gives you EXPLICIT CONTROL over state and control flow.                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 LangGraph Capabilities

| Capability | Description |
|------------|-------------|
| **Deterministic + Agentic Steps** | Mix hand-coded nodes with LLM-driven nodes in same graph |
| **Persistence** | Checkpoint state, resume after failures/pauses |
| **Human-in-the-Loop (HITL)** | Pause execution, inspect/edit state, continue |
| **Memory** | Short-term (thread) + Long-term (cross-session) |
| **Streaming** | Tokens, state updates, events in real-time |

---

## 4. Core LangGraph Concepts

### 4.1 The Graph Abstraction

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGGRAPH GRAPH = STATE + NODES + EDGES                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STATE (The "Brain")                                                        │
│  ────────────────                                                           │
│  • Shared data structure passed between all nodes                           │
│  • Can be: TypedDict, Pydantic model, dataclass, MessagesState              │
│  • Contains: conversation history, user prefs, tool results, plans          │
│  • Immutable updates: nodes return partial state → merged automatically     │
│                                                                             │
│  NODES (The "Actions")                                                      │
│  ───────────────                                                            │
│  • Functions that process state → return state updates                      │
│  • Types:                                                                   │
│    - LLM nodes: Call model, return response                                 │
│    - Tool nodes: Execute functions, return results                          │
│    - Logic nodes: Deterministic code (validation, routing, transforms)      │
│    - Human nodes: interrupt() for HITL                                      │
│  • Signature: (state) → Partial[State]                                      │
│                                                                             │
│  EDGES (The "Flow")                                                         │
│  ──────────────                                                             │
│  • Define execution order between nodes                                     │
│  • Types:                                                                   │
│    - Direct: A → B (always)                                                 │
│    - Conditional: A → B if condition else C                                 │
│    - START → Entry node                                                     │
│    - Node → END (termination)                                               │
│  • Router functions: state → next_node_name                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Minimal Working Example

```python
from langgraph.graph import StateGraph, MessagesState, START, END

# 1. Define node (processes state)
def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

# 2. Build graph
graph = StateGraph(MessagesState)
graph.add_node("llm", mock_llm)
graph.add_edge(START, "llm")
graph.add_edge("llm", END)

# 3. Compile (creates runnable)
app = graph.compile()

# 4. Invoke
result = app.invoke({"messages": [{"role": "user", "content": "hi!"}]})
# result = {"messages": [HumanMessage("hi!"), AIMessage("hello world")]}
```

---

## 5. Advanced Concepts Preview

### 5.1 Persistence & Checkpointing

```python
# Enable persistence with checkpointer
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# In-memory (development)
checkpointer = MemorySaver()

# SQLite (production)
checkpointer = SqliteSaver.from_conn_string("sqlite:///checkpoints.db")

# Compile with persistence
app = graph.compile(checkpointer=checkpointer)

# Run with thread_id (enables resume)
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke(input, config=config)

# Resume later - same thread_id continues from checkpoint
result2 = app.invoke({"messages": [{"role": "user", "content": "continue"}]}, config=config)
```

### 5.2 Human-in-the-Loop (HITL)

```python
from langgraph.types import interrupt, Command

def human_approval_node(state):
    # Pause execution, wait for human
    decision = interrupt({
        "question": "Approve this booking?",
        "details": state["proposed_booking"]
    })
    
    # Resume with human decision
    if decision["approved"]:
        return {"booking_confirmed": True}
    else:
        return {"booking_confirmed": False, "reason": decision["reason"]}

# In graph:
graph.add_node("human_approval", human_approval_node)
graph.add_edge("propose_booking", "human_approval")
graph.add_conditional_edges(
    "human_approval",
    lambda s: "book" if s["booking_confirmed"] else "cancel"
)
```

### 5.3 Memory (Short-term + Long-term)

```python
# Short-term: Thread state (automatic with checkpointer)
# Long-term: Cross-session memory
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# In node:
def save_preference(state, store):
    user_id = state["user_id"]
    prefs = store.get(("preferences", user_id)) or {}
    prefs["preferred_airline"] = state["airline_choice"]
    store.put(("preferences", user_id), prefs)
    return {}

# Retrieve in later session:
prefs = store.get(("preferences", "user-123"))
```

### 5.4 Streaming

```python
# Stream tokens
async for chunk in app.astream(input, config, stream_mode="messages"):
    print(chunk)  # Real-time token output

# Stream state updates
async for chunk in app.astream(input, config, stream_mode="values"):
    print(f"State: {chunk}")  # Full state after each node

# Stream events (node start/end, tool calls)
async for chunk in app.astream(input, config, stream_mode="updates"):
    print(f"Event: {chunk}")
```

---

## 6. LangGraph vs Other Frameworks

### 6.1 When to Use LangGraph

| Use LangGraph When... | Use Alternatives When... |
|----------------------|-------------------------|
| Need explicit state control | Want high-level agent loop (use Deep Agents) |
| Building custom workflows | Standard ReAct agent suffices (use LangChain agents) |
| Need durability/persistence | Simple one-shot tasks |
| Human-in-the-loop required | No oversight needed |
| Multi-agent coordination | Single agent |
| Long-running (hours/days) | Short tasks |

### 6.2 LangGraph vs LangChain Agents

```python
# LANGCHAIN AGENT (High-level, opinionated)
from langchain.agents import create_react_agent, AgentExecutor

agent = create_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)
result = executor.invoke({"input": "Plan Tokyo trip"})

# LANGGRAPH (Low-level, flexible)
from langgraph.graph import StateGraph

# You define EVERYTHING: state, nodes, edges, routing
graph = StateGraph(MyState)
graph.add_node("plan", plan_node)
graph.add_node("execute", execute_node)
graph.add_node("observe", observe_node)
# ... custom routing logic
app = graph.compile()
result = app.invoke({"input": "Plan Tokyo trip"})
```

---

## 7. Prerequisites Checklist

Before proceeding, ensure you're comfortable with:

```python
# 1. PYTHON ADVANCED
# ──────────────────
✅ Classes, inheritance, OOP
✅ Type hints, typing module (Generic, TypeVar, Union, Optional)
✅ Pydantic v2 (BaseModel, Field, validators, computed fields)
✅ AsyncIO (async/await, asyncio.gather, async context managers)
✅ Dataclasses, protocols, abstract base classes

# 2. LLM BASICS
# ─────────────
✅ How to call LLMs (OpenAI, Anthropic, local)
✅ Prompt engineering basics
✅ Token limits, context windows
✅ Temperature, top-p, streaming

# 3. LANGCHAIN FUNDAMENTALS (Ch 1-10 of your guide)
# ────────────────────────────────────────────────
✅ Components: Models, Prompts, Parsers, Runnables
✅ Chains & LCEL (| operator, RunnableSequence)
✅ Document Loaders, Text Splitters, Vector Stores
✅ Tools & Tool Calling (bind_tools, structured output)
✅ RAG pipeline basics
✅ Agents: ReAct, tool calling agents
```

---

## 8. Module Roadmap

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LANGGRAPH LEARNING ROADMAP                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MODULE 1: FOUNDATIONS OF AGENTIC AI (Lectures 1-5)                         │
│  ─────────────────────────────────────────────────────                      │
│  Lecture 1: GenAI vs Agentic AI (THIS LECTURE)                              │
│  Lecture 2: What is Agentic AI? Deep Dive                                   │
│  Lecture 3: Agentic AI vs Traditional RAG/Workflows                         │
│  Lecture 4: Top Frameworks Comparison (LangGraph, CrewAI, AutoGen, etc.)    │
│  Lecture 5: When to Use Agents vs Workflows                                 │
│                                                                             │
│  MODULE 2: LANGGRAPH FUNDAMENTALS (Lectures 6-15)                           │
│  ─────────────────────────────────────────────────                          │
│  • Graph anatomy: State, Nodes, Edges                                       │
│  • StateGraph API, compilation, invocation                                  │
│  • Conditional edges, routing, loops                                        │
│  • Building workflows: Sequential, Parallel, Branching                      │
│  • Popular patterns: ReAct, Plan-Execute, Reflection                        │
│                                                                             │
│  MODULE 3: ADVANCED LANGGRAPH (Lectures 16-28)                              │
│  ─────────────────────────────────────────────                              │
│  • Persistence & Checkpointing                                              │
│  • Human-in-the-Loop (interrupt, Command)                                   │
│  • Memory: Short-term + Long-term                                           │
│  • Streaming (tokens, values, updates)                                      │
│  • Breakpoints, Time Travel, Debugging                                      │
│  • Subgraphs, Multi-agent patterns                                          │
│                                                                             │
│  MODULE 4: BUILDING AI AGENTS (Lectures 29-50)                              │
│  ──────────────────────────────────────────────                             │
│  • Agent design patterns (ReAct, Plan-Execute, ReWOO, etc.)                 │
│  • ReAct Agent from scratch                                                 │
│  • Multi-agent systems                                                      │
│  • Production agents: observability, eval, guardrails                       │
│  • Real projects: Research agent, Coding agent, Support agent               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Installation & Setup

```bash
# Core installation
pip install -U langgraph

# With LangChain (recommended for integrations)
pip install -U langgraph langchain langchain-openai langchain-anthropic

# Optional: Checkpointers
pip install -U langgraph-checkpoint-sqlite  # SQLite persistence
pip install -U langgraph-checkpoint-postgres  # PostgreSQL

# Optional: LangSmith (observability)
pip install -U langsmith
# Set env vars:
# LANGCHAIN_TRACING_V2=true
# LANGCHAIN_API_KEY=your_key
# LANGCHAIN_PROJECT=langgraph-demo
```

---

## 10. Quick Reference: LangGraph Cheat Sheet

```python
# ============================================
# LANGGRAPH ESSENTIALS CHEAT SHEET
# ============================================

# 1. STATE DEFINITION
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # Auto-merges message lists
    user_id: str
    preferences: dict
    current_plan: list

# 2. NODE DEFINITION
def my_node(state: State) -> State:
    # Process state, return UPDATES only (merged automatically)
    return {"messages": [AIMessage("response")], "current_plan": ["step1", "step2"]}

# 3. GRAPH BUILDING
from langgraph.graph import StateGraph, START, END

graph = StateGraph(State)
graph.add_node("node_name", my_node)
graph.add_edge(START, "node_name")
graph.add_edge("node_name", END)

# Conditional edge
def router(state: State) -> str:
    return "path_a" if state["condition"] else "path_b"

graph.add_conditional_edges("node_name", router, {"path_a": "node_a", "path_b": "node_b"})

# 4. COMPILE & RUN
app = graph.compile(checkpointer=MemorySaver())  # With persistence

# Invoke
result = app.invoke({"messages": [HumanMessage("hi")]}, config={"thread_id": "1"})

# Stream
async for chunk in app.astream(input, config, stream_mode="values"):
    print(chunk)

# 5. HUMAN IN THE LOOP
from langgraph.types import interrupt, Command

def hitl_node(state):
    decision = interrupt({"question": "Approve?"})
    return {"approved": decision["yes"]}

# Resume after interrupt
app.invoke(Command(resume={"yes": True}), config)

# 6. MEMORY (Cross-session)
from langgraph.store.memory import InMemoryStore
store = InMemoryStore()
store.put(("prefs", "user1"), {"theme": "dark"})
prefs = store.get(("prefs", "user1"))
```

---

## 11. Key Takeaways

| Concept | Essence |
|---------|---------|
| **Agentic AI** | Autonomous goal execution, not passive generation |
| **LangGraph** | Low-level orchestration runtime for stateful agents |
| **Graph = State + Nodes + Edges** | Core abstraction — explicit control over flow |
| **Persistence** | Checkpointing enables durability, resume, HITL |
| **HITL** | `interrupt()` pauses graph, `Command(resume=...)` continues |
| **Memory** | Thread state (short-term) + Store (long-term) |
| **Streaming** | Real-time tokens, state updates, events |
| **Prerequisites** | Python OOP, Pydantic, AsyncIO, LangChain basics |

---

## 12. Resources & References

### Official
- **LangGraph Overview:** https://docs.langchain.com/oss/python/langgraph/overview
- **LangGraph Quickstart:** https://docs.langchain.com/oss/python/langgraph/quickstart
- **LangGraph API Reference:** https://docs.langchain.com/oss/python/langgraph/api-reference
- **LangGraph Concepts:** https://docs.langchain.com/oss/python/langgraph/concepts

### Videos
- **Channel:** https://www.youtube.com/@campusx-official

---