# Chapter 7: AI Agents in LangChain — Complete Guide to Autonomous Systems

## 7.1 What is an AI Agent? The Fundamental Shift

### 7.1.1 From Chains to Agents: The Autonomy Spectrum

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    AUTONOMY SPECTRUM                                      │
├──────────────────┬──────────────────┬──────────────────┬──────────────────┤
│   CHAIN          │  TOOL-ENABLED    │   AGENT          │  MULTI-AGENT     │
│  (Deterministic) │  (Semi-Auto)     │  (Autonomous)    │  (Collaborative) │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│  Fixed sequence  │  LLM decides     │  LLM plans,      │  Multiple agents │
│  of steps        │  which tool      │  acts, observes, │  coordinate,     │
│  No decision     │  to call         │  loops until     │  delegate,       │
│  making          │  Human defines   │  goal achieved   │  negotiate       │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│  prompt \| model │  bind_tools() +  │  create_react_   │  LangGraph       │
│  \| parser       │  manual invoke   │  agent()         │  StateGraph      │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

### 7.1.2 What Makes an Agent "Agentic"?

> **Definition:** "An agent is a software application that is autonomous. You give it a goal, and it pursues that goal by reasoning, taking actions, observing results, and looping until the goal is achieved."

| Characteristic | Chain | Agent |
|----------------|-------|-------|
| **Control Flow** | Fixed (programmer-defined) | Dynamic (LLM-decided) |
| **Reasoning** | Single-shot | Multi-step (ReAct, CoT) |
| **Tool Use** | Predefined sequence | Dynamic selection |
| **Error Handling** | Fixed retry logic | Adaptive (retry, switch tools, replan) |
| **Goal Pursuit** | Single execution | Loop until success/failure |
| **State Management** | Stateless (per call) | Stateful (conversation + scratchpad) |

---

## 7.2 The ReAct Pattern: Foundation of Modern Agents

### 7.2.1 ReAct = Reasoning + Acting

> **Origin:** "ReAct: Synergizing Reasoning and Acting in Language Models" (Yao et al., 2022)

```
┌─────────────────────────────────────────────────────────────────┐
│                      ReAct LOOP                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐                                               │
│   │   THOUGHT   │  "I need to find the current weather..."      │
│   └──────┬──────┘                                               │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                               │
│   │   ACTION    │  search_weather("Tokyo")                      │
│   └──────┬──────┘                                               │
│          │                                                      │
│          ▼                                                      │
│   ┌────────────────┐                                            │
│   │  OBSERVATION   │  "Tokyo: 22°C, Sunny, 60% humidity"        │
│   └──────┬─────────┘                                            │
│          │                                                      │
│          ▼ (loop until final answer)                            │
│   ┌─────────────┐                                               │
│   │  THOUGHT    │  "I have the weather info, can answer now"    │
│   └──────┬──────┘                                               │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                               │
│   │ FINAL ANSWER│  "Tokyo is currently 22°C and sunny..."       │
│   └─────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2.2 ReAct Prompt Structure (Internal)

```
You are an agent that can use tools. You have access to:
- search_weather(city: str) -> str
- calculate(expression: str) -> float

Format your response as:
Thought: <your reasoning>
Action: <tool_name>
Action Input: <arguments>
Observation: <result>
... (repeat)
Thought: <final reasoning>
Final Answer: <answer>
```

---

## 7.3 LangChain Agent Architecture: Complete Taxonomy

### 7.3.1 Agent Types in LangChain v0.2+

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGCHAIN AGENT TAXONOMY                                 │
├─────────────────────┬───────────────────────────────────────────────────────┤
│  LEGACY (v0.1)      │  AgentExecutor + ZeroShotAgent / ChatAgent            │
│                     │  ❌ DEPRECATED - Don't use                            │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  MODERN v0.2+       │                                                       │
│  ─────────────────  │                                                       │
│  • ReAct Agent      │  create_react_agent() → LangGraph                     │
│    (Standard)       │  Reasoning + Acting loop                              │
│                     │                                                       │
│  • OpenAI Functions │  create_openai_functions_agent()                      │
│    Agent            │  Native function calling (structured)                 │
│                     │                                                       │
│  • Plan-and-Execute │  create_plan_and_execute_agent()                      │
│    Agent            │  Planner → Executor → Replanner                       │
│                     │                                                       │
│  • Self-Ask /       │  Specialized patterns                                 │
│    Self-Reflection  │                                                       │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  LANGGRAPH          │  create_react_agent() returns LangGraph app           │
│  (Recommended)      │  Stateful, checkpointer, human-in-the-loop            │
│                     │                                                       │
│  MULTI-AGENT        │  LangGraph: Supervisor + Workers                      │
│                     │  Swarm, Hierarchical, Peer-to-Peer                    │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

---

## 7.4 Building ReAct Agents: Complete Implementation

### 7.4.1 Modern ReAct Agent (LangGraph) — RECOMMENDED

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage

# 1. Define Tools
@tool
def search_web(query: str) -> str:
    """Search the web for current information."""
    # Implementation: requests + BeautifulSoup / SerpAPI
    return f"Search results for: {query}"

@tool
def calculate(expression: str) -> float:
    """Evaluate a mathematical expression safely."""
    # Safe eval implementation
    allowed = {"__builtins__": {}}
    return eval(expression, {"__builtins__": {}}, {})

@tool
def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    return {"city": city, "temp": 22, "condition": "Sunny"}

# 2. Create Agent (One-liner!)
tools = [search_web, calculate, get_weather]
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

agent = create_react_agent(
    model=llm,
    tools=tools,
    prompt="You are a helpful assistant with access to tools. Use them when needed."
)

# 3. Invoke (Autonomous Loop!)
result = agent.invoke({"messages": [HumanMessage("What's 25 * 42 + weather in Tokyo?")]})
print(result["messages"][-1].content)
# Agent autonomously: multiplies → gets weather → combines answer
```

### 7.4.2 Legacy AgentExecutor (v0.1 — For Migration Reference)

```python
# ⚠️ LEGACY — For migration only
from langchain.agents import AgentExecutor, create_react_agent
from langchain import hub

# Legacy prompt from LangChain Hub
prompt = hub.pull("hwchase17/react")

agent = create_react_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = agent_executor.invoke({"input": "What's 25 * 42?"})
```

### 7.4.3 ReAct Agent with Memory (Conversational)

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent

# Add memory for conversation continuity
memory = MemorySaver()

agent = create_react_agent(
    model=llm,
    tools=tools,
    checkpointer=memory  # Enables conversation memory
)

# Config with session ID
config = {"configurable": {"thread_id": "user_123"}}

# First interaction
result = agent.invoke({"messages": [HumanMessage("My name is Alice")]}, config=config)

# Second interaction — remembers name!
result = agent.invoke({"messages": [HumanMessage("What's my name?")]}, config=config)
```

---

## 7.5 Advanced Agent Patterns

### 7.5.1 OpenAI Functions Agent (Structured Tool Calling)

```python
from langgraph.prebuilt import create_openai_functions_agent

# Uses OpenAI's native function calling (more reliable than ReAct parsing)
agent = create_openai_functions_agent(
    model=llm,
    tools=tools,
    prompt="You are a helpful assistant with tools."
)

# Better for: structured outputs, parallel tool calls, reliability
result = agent.invoke({"messages": [HumanMessage("Calculate 25*42 AND get Tokyo weather")]})
# Can make PARALLEL tool calls in single LLM call!
```

### 7.5.2 Plan-and-Execute Agent (Complex Multi-Step Tasks)

```python
from langgraph.prebuilt import create_plan_and_execute_agent

# For complex tasks requiring planning
agent = create_plan_and_execute_agent(
    model=llm,
    tools=tools,
    planner_prompt="You are a planner. Break down the task into steps.",
    executor_prompt="You are an executor. Execute each step with tools."
)

# Flow: Planner creates plan → Executor runs steps → Replanner adjusts
result = agent.invoke({"messages": [HumanMessage("Research solar energy trends and create a summary report")]})
```

**When to Use Plan-and-Execute:**
- ✅ Complex multi-step tasks (research, report generation)
- ✅ Tasks needing decomposition before execution
- ✅ When you want visibility into the plan

### 7.5.3 Self-Ask / Self-Reflection Agents

```python
from langgraph.prebuilt import create_self_ask_with_search_agent

# Agent that asks itself sub-questions
agent = create_self_ask_with_search_agent(
    model=llm,
    tools=[search_web],
    prompt="You are a research assistant. Ask yourself sub-questions to answer complex queries."
)

# Automatically decomposes: "Who won the 2022 World Cup?" 
# → "Who played in the 2022 World Cup final?" → "Argentina vs France" → "Argentina won"
```

---

## 7.6 Agent State & Architecture (LangGraph Deep Dive)

### 7.6.1 Agent State Schema

```python
from typing import TypedDict, List, Annotated
from langgraph.graph import StateGraph, MessagesState
from langchain_core.messages import BaseMessage
from operator import add

# Default MessagesState (used by create_react_agent)
class AgentState(MessagesState):
    """Default state: messages + any custom fields"""
    pass

# Custom state for complex agents
class CustomAgentState(TypedDict):
    messages: Annotated[List[BaseMessage], add]
    user_id: str
    session_data: dict
    iteration_count: int
    last_tool_result: str
```

### 7.6.2 Custom Agent Graph (Full Control)

```python
from langgraph.graph import StateGraph, END
from langchain_core.messages import ToolMessage

# Build custom agent graph
workflow = StateGraph(AgentState)

# Nodes
def agent_node(state):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def tool_node(state):
    # Execute tool calls from last AIMessage
    last_message = state["messages"][-1]
    tool_messages = []
    for tc in last_message.tool_calls:
        tool_fn = tool_map[tc["name"]]
        result = tool_fn.invoke(tc["args"])
        tool_messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))
    return {"messages": tool_messages}

# Conditional edge
def should_continue(state):
    last = state["messages"][-1]
    return "tools" if last.tool_calls else "end"

workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
workflow.add_edge("tools", "agent")

app = workflow.compile(checkpointer=MemorySaver())
```

---

## 7.7 Multi-Agent Systems (LangGraph)

### 7.7.1 Supervisor Pattern (Hierarchical)

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import create_react_agent

# Specialist agents
researcher = create_react_agent(llm, [search_web, search_academic])
writer = create_react_agent(llm, [write_document, format_report])
critic = create_react_agent(llm, [review_document, fact_check])

# Supervisor routes to specialists
supervisor_prompt = """You are a supervisor. Route tasks:
- research → researcher
- write → writer
- review → critic
"""

supervisor = create_react_agent(
    llm,
    tools=[],  # Supervisor only routes
    prompt=supervisor_prompt
)

# Build graph
workflow = StateGraph(MessagesState)
workflow.add_node("supervisor", supervisor)
workflow.add_node("researcher", researcher)
workflow.add_node("writer", writer)
workflow.add_node("critic", critic)

# Supervisor routes
def route(state):
    last = state["messages"][-1]
    if "research" in last.content.lower():
        return "researcher"
    elif "write" in last.content.lower():
        return "writer"
    elif "review" in last.content.lower():
        return "critic"
    return END

workflow.add_conditional_edges("supervisor", route)
workflow.add_edge("researcher", "supervisor")
workflow.add_edge("writer", "supervisor")
workflow.add_edge("critic", "supervisor")
workflow.set_entry_point("supervisor")

app = workflow.compile()
```

### 7.7.2 Swarm / Peer-to-Peer

```python
# Agents communicate directly, no central supervisor
# Useful for: collaborative problem solving, debate, consensus
```

---

## 7.8 Agent Tools: Design Patterns

### 7.8.1 Tool Design Best Practices

```python
from pydantic import BaseModel, Field
from typing import Optional
from langchain_core.tools import tool

# ✅ GOOD: Clear schema, descriptive names, docstrings
class WeatherInput(BaseModel):
    city: str = Field(description="City name, e.g., 'Tokyo'")
    units: str = Field(default="celsius", description="celsius or fahrenheit")

@tool(args_schema=WeatherInput)
def get_weather(city: str, units: str = "celsius") -> dict:
    """Get current weather for a city. Returns temperature, condition, humidity."""
    # Implementation
    return {"city": city, "temp": 22, "unit": units, "condition": "Sunny"}

# ✅ Structured output for downstream use
class WeatherResult(BaseModel):
    city: str
    temperature: float
    condition: str
    humidity: int

@tool
def get_weather_structured(city: str) -> WeatherResult:
    """Returns structured weather data."""
    return WeatherResult(city=city, temperature=22.5, condition="Sunny", humidity=65)
```

### 7.8.2 Tool Categories for Agents

| Category | Tools | Use Case |
|----------|-------|----------|
| **Search** | `DuckDuckGoSearchRun`, `TavilySearch`, `GoogleSearch` | Information retrieval |
| **Code** | `PythonREPLTool`, `ShellTool` | Computation, data analysis |
| **API** | `RequestsGetTool`, `RequestsPostTool` | External integrations |
| **Database** | `SQLDatabaseToolkit` | Structured data queries |
| **File** | `FileManagementToolkit` | Read/write/list files |
| **Math** | `LLMMathChain` / custom `calculate` | Computations |
| **Custom** | Domain-specific | Business logic |

---

## 7.9 Agent Execution Patterns

### 7.9.1 Streaming Agent Output

```python
# Stream agent steps in real-time
for chunk in agent.stream({"messages": [HumanMessage("Research AI trends")]}):
    # chunk contains incremental state updates
    if "agent" in chunk:
        print(chunk["agent"]["messages"][-1].content, end="")
    elif "tools" in chunk:
        print(f"\n[Tool: {chunk['tools']}]")
```

### 7.9.2 Async Execution

```python
# Async for web servers
result = await agent.ainvoke({"messages": [HumanMessage("Research topic")]})
```

### 7.9.3 Human-in-the-Loop (Approval)

```python
from langgraph.types import interrupt, Command

def human_approval_node(state):
    # Pause for human review
    action = interrupt({"pending_action": state["messages"][-1].tool_calls[0]})
    # action = {"approve": True, "modified_args": {...}} or {"approve": False}
    return Command(resume=action)

# In graph:
workflow.add_node("human_approval", human_approval_node)
workflow.add_conditional_edges("agent", lambda s: "human_approval" if needs_approval(s) else "tools")
```

---

## 7.10 Debugging & Observability (LangSmith Integration)

### 7.10.1 Automatic Tracing (Zero Config)

```python
# Just set env vars - agents auto-trace!
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "my-agent"

# Every agent step appears in LangSmith:
# - Agent reasoning (Thought)
# - Tool calls (Action + Input)
# - Tool results (Observation)
# - Loop iterations
```

### 7.10.2 What You See in Traces

```
Trace: "Agent Execution"
├── Iteration 1:
│   ├── Agent Node (LLM)
│   │   Input: Messages
│   │   Output: AIMessage with tool_calls
│   │   Tokens: 45 in / 120 out
│   │
│   ├── Tool Node: search_web
│   │   Input: {"query": "solar energy trends 2024"}
│   │   Output: "Top trends: perovskite cells, AI optimization..."
│   │   Latency: 1.2s
│
├── Iteration 2:
│   ├── Agent Node (LLM with observation)
│   │   Output: Tool call: summarize_findings
│
├── Iteration 3:
│   ├── Agent Node
│   │   Output: Final Answer
│
Total: 3 iterations | 4.2s | 3 LLM calls | 2 tool calls
```

---

## 7.10 Production Patterns

### 7.11.1 Error Handling & Retry

```python
from langchain_core.runnables import RunnableWithFallbacks

# Tool with automatic retry + fallback
@tool
def unreliable_api(query: str) -> str:
    if "fail" in query:
        raise ValueError("API error")
    return "Success"

# Wrap tool with resilience
from langchain_core.runnables import RunnableLambda

safe_tool = RunnableLambda(unreliable_api.invoke).with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True
).with_fallbacks([RunnableLambda(lambda x: "Service unavailable, using cached data")])

# Use in agent
agent = create_react_agent(llm, [safe_tool, other_tools])
```

### 7.12 Rate Limiting & Cost Control

```python
# Track costs per agent run
from langchain_core.callbacks import BaseCallbackHandler

class CostTracker(BaseCallbackHandler):
    def on_llm_end(self, response, **kwargs):
        tokens = response.llm_output.get("token_usage", {})
        cost = (tokens.get("prompt_tokens", 0) * 0.00015 + 
                tokens.get("completion_tokens", 0) * 0.0006) / 1000
        print(f"Run cost: ${cost:.4f}")

agent = create_react_agent(llm, tools, callbacks=[CostTracker()])
```

---

## 7.13 Agent Evaluation & Testing

### 7.13.1 Testing Agent Behavior

```python
import pytest
from langsmith.evaluation import evaluate

def test_agent_weather():
    result = agent.invoke({"messages": [HumanMessage("Weather in Tokyo?")]})
    assert "Tokyo" in result["messages"][-1].content
    assert "22" in result["messages"][-1].content or "°C" in result["messages"][-1].content

# LangSmith evaluation
def agent_predict(example):
    return agent.invoke({"messages": [HumanMessage(example["question"])]})

results = evaluate(
    agent_predict,
    data="agent-qa-dataset",
    evaluators=["correctness", "tool_usage_correctness"],
)
```

---

## 7.14 Decision Framework: Which Agent to Use?

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    AGENT SELECTION GUIDE                                         │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Need simple tool use?          → create_openai_functions_agent()                │
│  Need reasoning + acting?       → create_react_agent() (DEFAULT)                 │
│  Need complex planning?         → create_plan_and_execute_agent()                │
│  Need self-decomposition?       → create_self_ask_with_search_agent()            │
│  Need full control/custom?      → Build custom StateGraph                        │
│  Need multi-agent?              → LangGraph StateGraph + multiple agents         │
│  Need human approval?           → LangGraph interrupt() + Command                │
│  Need memory?                   → create_react_agent(checkpointer=MemorySaver()) │
│                                                                                  │
│  LEGACY: AgentExecutor + create_react_agent()  →  MIGRATE TO LANGGRAPH           │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.15 Complete Agent Cheat Sheet

```python
# ============================================
# QUICK START: PRODUCTION-READY REACT AGENT
# ============================================
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import MemorySaver

# 1. Tools with clear schemas
@tool
def search(query: str) -> str:
    """Search web for information."""
    return f"Results for: {query}"

@tool
def calculate(expr: str) -> float:
    """Safe math evaluation."""
    return eval(expr, {"__builtins__": {}}, {})

# 2. LLM + Tools
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
tools = [search, calculate]

# 3. Agent with Memory
memory = MemorySaver()
agent = create_react_agent(
    model=llm,
    tools=tools,
    checkpointer=memory,
    prompt="You are a helpful assistant. Use tools when needed."
)

# 4. Run with Session
config = {"configurable": {"thread_id": "user_123"}}
result = agent.invoke({"messages": [HumanMessage("What's 15*42?")]}, config=config)

# 5. Streaming
for chunk in agent.stream({"messages": [HumanMessage("Research topic")]}, config=config):
    # Process chunks
    pass

# 6. LangSmith Tracing (auto with env vars)
# LANGCHAIN_TRACING_V2=true
# LANGCHAIN_PROJECT=my-agent
```

---

## 7.16 Complete Agent Decision Tree

```
START: Need an autonomous system?
│
├─► Simple Q&A with tools? 
│   └─► create_openai_functions_agent()
│
├─► Need reasoning + tool use? (DEFAULT)
│   └─► create_react_agent() + MemorySaver()
│
├─► Complex multi-step task?
│   └─► create_plan_and_execute_agent()
│
├─► Self-decomposition needed?
│   └─► create_self_ask_with_search_agent()
│
├─► Full custom control?
│   └─► Custom StateGraph (LangGraph)
│
├─► Multiple specialists?
│   └─► Supervisor + Worker agents (LangGraph)
│
├─► Need human approval?
│   └─► interrupt() + Command in LangGraph
│
└─► Production deployment?
    └─► LangGraph + LangSmith + MemorySaver + Error handling
```

---

## 7.17 Resources & Next Steps

### Official Resources
- **LangGraph Agents:** https://langchain-ai.github.io/langgraph/
- **Agent Guide:** https://python.langchain.com/docs/concepts/agents/
- **Migration Guide:** https://python.langchain.com/docs/versions/migrating_agents/

### Advanced Topics to Explore
| Topic | Description |
|-------|-------------|
| **LangGraph Platform** | Hosted agent runtime with persistence |
| **Agent Evaluation** | LangSmith agent benchmarks |
| **Multi-Agent Orchestration** | Supervisor, Swarm, Hierarchical |
| **Tool Calling Optimization** | Parallel calls, structured outputs |
| **Agent Memory** | Long-term, semantic, episodic, procedural |

---

## 7.18 Summary: From Chains to Agents

| Aspect | Chain | Agent |
|--------|-------|-------|
| **Paradigm** | Declarative pipeline | Autonomous loop |
| **Control** | Programmer | LLM |
| **Flexibility** | Fixed | Dynamic |
| **Debugging** | Easy (linear) | Complex (requires tracing) |
| **Use Case** | Deterministic workflows | Open-ended problems |

> **Key Insight:** Agents are not "better" than chains — they solve **different problems**. Use chains for deterministic workflows; agents for open-ended, reasoning-heavy tasks requiring autonomy.
