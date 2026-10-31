# Chapter 6: Tools & Tool Calling — Giving LLMs Hands and Feet

## 6.1 The Fundamental Problem: LLMs Can't Act

> **Analogy:** LLMs are like humans who can think and speak but have **no hands or feet**.

```
LLM Capabilities:
├── ✅ Reasoning (break down questions, understand intent)
├── ✅ Output Generation (generate text, code, explanations)
└── ❌ Action Execution (cannot call APIs, query DBs, send emails, browse web)

The Missing Piece: TOOLS
```

**Tools = Functions with schemas that LLMs can invoke to perform actions.**

---

## 6.2 Tool Anatomy

Every tool in LangChain has three required components:

```python
@tool
def multiply(a: int, b: int) -> int:
    """Given two numbers A and B, this tool returns their product."""
    return a * b
```

| Component | Purpose | Example |
|-----------|---------|---------|
| **Name** | Unique identifier | `"multiply"` |
| **Description** | Tells LLM *what* the tool does | `"Given two numbers A and B, this tool returns their product."` |
| **Input Schema** | Tells LLM *how* to call it (JSON Schema) | `{"a": {"type": "integer"}, "b": {"type": "integer"}}` |

---

## 6.3 Creating Tools

### 6.3.1 The `@tool` Decorator (Modern, Recommended)

```python
from langchain_core.tools import tool
from typing import Optional

# Simple tool
@tool
def multiply(a: int, b: int) -> int:
    """Given two numbers A and B, this tool returns their product."""
    return a * b

# Tool with optional parameters
@tool
def get_weather(city: str, units: Optional[str] = "celsius") -> dict:
    """Get current weather for a city."""
    # Real implementation would call weather API
    return {"city": city, "temperature": 22, "units": units}

# Tool with complex input (Pydantic model)
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    max_results: int = Field(default=5, description="Max results to return")

@tool(args_schema=SearchInput)
def web_search(query: str, max_results: int = 5) -> list:
    """Search the web for information."""
    # Implementation here
    return [{"title": "Result 1", "url": "..."}]
```

### 6.3.2 Tool Attributes (All Tools Are Runnables!)

```python
print(multiply.name)        # "multiply"
print(multiply.description) # "Given two numbers A and B..."
print(multiply.args_schema) # Pydantic model with schema

# Invoke directly (tools are Runnables!)
result = multiply.invoke({"a": 3, "b": 4})
print(result)  # 12
```

---

## 6.4 Built-in Tools & Toolkits

LangChain provides 50+ built-in tools:

```python
from langchain_community.tools import (
    DuckDuckGoSearchRun,
    WikipediaQueryRun,
    ShellTool,
    PythonREPLTool,
    RequestsGetTool,
    BraveSearch,
    GoogleSearchAPIWrapper,
)

# Quick usage
search = DuckDuckGoSearchRun()
result = search.invoke("LangChain tool calling")

wiki = WikipediaQueryRun()
result = wiki.invoke("LangChain")

shell = ShellTool()
result = shell.invoke({"commands": ["ls", "-la"]})

python_repl = PythonREPLTool()
result = python_repl.invoke({"code": "print(2+2)"})
```

### Toolkits (Pre-configured Tool Collections)

```python
from langchain_community.agent_toolkits import (
    SQLDatabaseToolkit,
    VectorStoreToolkit,
    FileManagementToolkit,
    GitHubToolkit,
    PandasToolkit,
)

# SQL Toolkit
from langchain_community.utilities import SQLDatabase
db = SQLDatabase.from_uri("sqlite:///example.db")
toolkit = SQLDatabaseToolkit(db=db, llm=llm)
tools = toolkit.get_tools()  # [query_sql, list_tables, describe_table, ...]

# File Management
from langchain_community.agent_toolkits.file_management import FileManagementToolkit
file_toolkit = FileManagementToolkit(root_dir="./workspace")
tools = file_toolkit.get_tools()  # [read_file, write_file, list_directory, ...]
```

---

## 8.5 Custom Tools — Real-World Examples

### Currency Converter

```python
import requests
from langchain_core.tools import tool
from pydantic import BaseModel, Field

class ConversionInput(BaseModel):
    from_currency: str = Field(description="Source currency code (e.g., USD)")
    to_currency: str = Field(description="Target currency code (e.g., INR)")
    amount: float = Field(description="Amount to convert")

@tool(args_schema=ConversionInput)
def convert_currency(from_currency: str, to_currency: str, amount: float) -> str:
    """Convert amount from one currency to another using real-time rates."""
    url = f"https://api.exchangerate-api.com/v4/latest/{from_currency}"
    response = requests.get(url)
    data = response.json()
    rate = data["rates"][to_currency]
    converted = amount * rate
    return f"{amount} {from_currency} = {converted:.2f} {to_currency} (Rate: {rate:.4f})"

class RateInput(BaseModel):
    from_currency: str = Field(description="Source currency")
    to_currency: str = Field(description="Target currency")

@tool(args_schema=RateInput)
def get_conversion_factor(from_currency: str, to_currency: str) -> str:
    """Get the current conversion rate between two currencies."""
    url = f"https://api.exchangerate-api.com/v4/latest/{from_currency}"
    response = requests.get(url)
    data = response.json()
    rate = data["rates"][to_currency]
    return f"1 {from_currency} = {rate:.4f} {to_currency}"
```

---

## 6.6 Tool Binding — Connecting Tools to LLMs

> **Tool Binding** = Registering tools with an LLM so it knows:
> 1. What tools are available
> 2. What each tool does (description)
> 3. How to call each tool (input schema)

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Bind tools (returns NEW LLM with tools attached)
llm_with_tools = llm.bind_tools([multiply, get_weather, convert_currency])

# Now the LLM knows about these tools!
print(llm_with_tools.kwargs)  # Shows bound tools
```

### Which LLMs Support Tool Binding?

| Provider | Models Supporting Tools |
|----------|------------------------|
| **OpenAI** | GPT-4o, GPT-4o-mini, GPT-4-turbo, GPT-3.5-turbo (1106+) |
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Haiku/Opus, Claude 3.7+ |
| **Google** | Gemini 1.5 Pro, Gemini 1.5 Flash, Gemini 1.0 Pro |
| **Open Source** | Via `ChatHuggingFace` (limited), Ollama (with `format="json"`) |

---

## 6.7 Tool Calling — The LLM Decides When to Use Tools

> **Tool Calling** = Process where LLM **autonomously decides** to invoke a tool during conversation.

### 6.7.1 How It Works (Automatic)

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, ToolMessage

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

llm = ChatOpenAI(model="gpt-4o-mini")
llm_with_tools = llm.bind_tools([multiply])

# LLM automatically decides to call tool!
result = llm_with_tools.invoke("What is 25 * 83?")
print(result.tool_calls)
# [{'name': 'multiply', 'args': {'a': 25, 'b': 83}, 'id': 'call_abc123'}]

# Execute the tool call
tool_result = multiply.invoke(result.tool_calls[0]["args"])
print(tool_result)  # 2075

# Feed back to LLM
messages = [
    HumanMessage("What is 25 * 83?"),
    result,  # AIMessage with tool_calls
    ToolMessage(content=str(tool_result), tool_call_id=result.tool_calls[0]["id"])
]
final = llm_with_tools.invoke(messages)
print(final.content)  # "25 * 83 = 2075"
```

---

## 6.8 Tool Execution Patterns

### 6.8.1 Manual Execution (Full Control)

```python
# Step 1: LLM decides to call tool
response = llm_with_tools.invoke("What's 15 * 42?")

# Step 2: Execute each tool call
tool_messages = []
for tool_call in response.tool_calls:
    # Find the tool function
    tool_fn = {"multiply": multiply}[tool_call["name"]]
    
    # Execute
    output = tool_fn.invoke(tool_call["args"])
    
    # Create ToolMessage
    tool_messages.append(ToolMessage(
        content=str(output),
        tool_call_id=tool_call["id"]
    ))

# Step 3: Send results back to LLM
final = llm_with_tools.invoke([
    HumanMessage("What's 15 * 42?"),
    response,
    *tool_messages
])
```

---

### 6.8.2 Automatic Execution with `ToolNode` (LangGraph)

```python
from langgraph.prebuilt import ToolNode
from langgraph.graph import StateGraph, MessagesState

# ToolNode automatically executes tool calls!
tools = [multiply, get_weather, convert_currency]
tool_node = ToolNode(tools)

# In a graph:
graph = StateGraph(MessagesState)
graph.add_node("llm", lambda state: llm_with_tools.invoke(state["messages"]))
graph.add_node("tools", tool_node)
graph.add_edge("llm", "tools")
graph.add_edge("tools", "llm")
graph.set_entry_point("llm")

app = graph.compile()
result = app.invoke({"messages": [HumanMessage("What's 25 * 83?")]})
```

---

### 6.8.3 Agent Pattern (Full Autonomy)

> **Key Distinction:** Manual execution = **Chain**. Automatic loop = **Agent**.

```python
from langgraph.prebuilt import create_react_agent

# create_react_agent handles the entire loop automatically!
agent = create_react_agent(
    model=llm,
    tools=[multiply, get_weather, convert_currency],
    prompt="You are a helpful assistant with access to tools."
)

# Agent autonomously:
# 1. Thinks about query
# 2. Calls tools as needed
# 3. Gets results
# 4. Decides next step
# 5. Repeats until done
result = agent.invoke({"messages": [HumanMessage("Convert 100 USD to INR")]})
print(result["messages"][-1].content)
```

---

## 6.9 Complete Tool Calling Flow (Currency Converter Example)

```python
# Full manual flow
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage
from pydantic import BaseModel, Field
import requests

# Tools
class RateInput(BaseModel):
    from_currency: str = Field(description="Source currency")
    to_currency: str = Field(description="Target currency")

@tool(args_schema=RateInput)
def get_conversion_factor(from_currency: str, to_currency: str) -> str:
    """Get current conversion rate between two currencies."""
    url = f"https://api.exchangerate-api.com/v4/latest/{from_currency}"
    response = requests.get(url)
    data = response.json()
    rate = data["rates"][to_currency]
    return f"1 {from_currency} = {rate:.4f} {to_currency}"

class ConversionInput(BaseModel):
    from_currency: str = Field(description="Source currency")
    to_currency: str = Field(description="Target currency")
    amount: float = Field(description="Amount to convert")

@tool(args_schema=ConversionInput)
def convert_currency(from_currency: str, to_currency: str, amount: float) -> str:
    """Convert amount using real-time rates."""
    url = f"https://api.exchangerate-api.com/v4/latest/{from_currency}"
    response = requests.get(url)
    data = response.json()
    rate = data["rates"][to_currency]
    converted = amount * rate
    return f"{amount} {from_currency} = {converted:.2f} {to_currency} (Rate: {rate:.4f})"

# Setup
llm = ChatOpenAI(model="gpt-4o-mini")
llm_with_tools = llm.bind_tools([get_conversion_factor, convert_currency])

# User query
user_query = "Convert 10 USD to INR"
messages = [HumanMessage(user_query)]

# === STEP 1: LLM calls get_conversion_factor ===
response1 = llm_with_tools.invoke(messages)
messages.append(response1)

# Execute tool
for tc in response1.tool_calls:
    result = get_conversion_factor.invoke(tc["args"])
    messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))

# === STEP 2: LLM calls convert_currency with rate ===
response2 = llm_with_tools.invoke(messages)
messages.append(response2)

# Execute tool
for tc in response2.tool_calls:
    result = convert_currency.invoke(tc["args"])
    messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))

# === STEP 3: LLM gives final answer ===
final = llm_with_tools.invoke(messages)
print(final.content)
# "10 USD = 830.45 INR (Rate: 83.0450)"
```

---

## 6.10 Tools vs Agents — Critical Distinction

| Aspect | Tool-Enabled Chain | Agent (LangGraph) |
|--------|-------------------|-------------------|
| **Control Flow** | Programmer defines steps | LLM decides steps |
| **Tool Calling** | Manual (you invoke) | Automatic (loop) |
| **Reasoning** | Single shot | Multi-step (ReAct) |
| **Autonomy** | Low | High |
| **Use Case** | Deterministic workflows | Open-ended problems |

```python
# TOOL-ENABLED CHAIN (Programmer controls flow)
def convert_currency_chain(query: str):
    # 1. Programmer decides: first get rate
    rate = get_conversion_factor.invoke(parse_query(query))
    # 2. Programmer decides: then convert
    result = convert_currency.invoke({"amount": 10, "rate": rate})
    return result

# AGENT (LLM controls flow)
agent = create_react_agent(llm, tools=[get_rate, convert])
result = agent.invoke({"messages": [HumanMessage("Convert 10 USD to INR")]})
# Agent decides: need rate → call get_rate → have rate → call convert → done
```

---

## 6.11 Advanced Tool Patterns

### 6.11.1 Structured Tool Output (Pydantic)

```python
from pydantic import BaseModel, Field

class WeatherResult(BaseModel):
    city: str
    temperature: float
    condition: str
    humidity: int

@tool
def get_weather_structured(city: str) -> WeatherResult:
    """Get structured weather data."""
    # API call...
    return WeatherResult(
        city=city,
        temperature=22.5,
        condition="Sunny",
        humidity=65
    )

# LLM gets structured output automatically
llm_with_tools = llm.bind_tools([get_weather_structured])
result = llm_with_tools.invoke("Weather in Tokyo?")
# result.tool_calls[0]["args"] validated against WeatherResult schema
```

---

### 6.11.2 Tool Choice Control

```python
# Force tool use
llm_forced = llm.bind_tools(tools, tool_choice="multiply")

# Force specific tool
llm_specific = llm.bind_tools(tools, tool_choice={"name": "get_weather"})

# Auto (default) - LLM decides
llm_auto = llm.bind_tools(tools, tool_choice="auto")

# None - no tools
llm_none = llm.bind_tools(tools, tool_choice="none")
```

---

### 6.11.3 Parallel Tool Calls

```python
# LLM can call MULTIPLE tools at once!
result = llm_with_tools.invoke("What's 5*3 AND weather in Tokyo?")
# result.tool_calls = [
#     {'name': 'multiply', 'args': {'a': 5, 'b': 3}, 'id': 'call_1'},
#     {'name': 'get_weather', 'args': {'city': 'Tokyo'}, 'id': 'call_2'}
# ]

# Execute in parallel
import asyncio
async def execute_parallel(tool_calls):
    tasks = [tool_map[tc["name"]].ainvoke(tc["args"]) for tc in tool_calls]
    return await asyncio.gather(*tasks)
```

---

## 6.12 Error Handling & Retry

```python
from langchain_core.runnables import RunnableWithFallbacks

# Tool with fallback
@tool
def unreliable_tool(x: int) -> int:
    if x < 0:
        raise ValueError("Negative not allowed")
    return x * 2

# Wrap with retry
from langchain_core.runnables import RunnableLambda

safe_tool = RunnableLambda(unreliable_tool.invoke).with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True
).with_fallbacks([RunnableLambda(lambda x: {"error": "Tool failed"})])

llm_with_safe = llm.bind_tools([safe_tool])
```

---

## 6.13 Testing Tools

```python
import pytest
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

def test_add_tool():
    # Direct invocation
    assert add.invoke({"a": 2, "b": 3}) == 5
    
    # Test schema
    assert add.args_schema.model_fields["a"].annotation == int
    
    # Test description
    assert "Add two numbers" in add.description

def test_add_tool_with_llm():
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(model="gpt-4o-mini")
    llm_with_tools = llm.bind_tools([add])
    
    result = llm_with_tools.invoke("What is 2 + 3?")
    assert len(result.tool_calls) == 1
    assert result.tool_calls[0]["name"] == "add"
    assert result.tool_calls[0]["args"] == {"a": 2, "b": 3}
```

---

## 6.14 Production Best Practices

### Tool Design
- [ ] **Clear descriptions** — LLM reads these to decide when to use
- [ ] **Precise schemas** — Use Pydantic for validation
- [ ] **Idempotent when possible** — Safe to retry
- [ ] **Timeout handling** — Set request timeouts
- [ ] **Error messages** — Return actionable errors, not stack traces

### Security
- [ ] **Validate inputs** — Pydantic does this automatically
- [ ] **Limit tool permissions** — Principle of least privilege
- [ ] **Sanitize outputs** — Prevent injection attacks
- [ ] **Audit tool calls** — Log for compliance

### Performance
- [ ] **Parallel execution** — Use `asyncio.gather` for multiple calls
- [ ] **Cache results** — For idempotent tools (weather, rates)
- [ ] **Batch similar calls** — Reduce API overhead
- [ ] **Monitor latency** — Tool calls add to total response time

---

## 6.15 Complete Decision Flowchart

```
Need to extend LLM capabilities?
│
├─► Simple function call?
│   └─► @tool decorator + bind_tools()
│
├─► Need LLM to decide WHEN to use tool?
│   └─► bind_tools() + automatic tool calling
│
├─► Multi-step autonomous problem solving?
│   └─► create_react_agent() (LangGraph)
│
├─► Deterministic workflow (programmer controls)?
│   └─► Manual tool invocation in chain
│
├─► Multiple tools, LLM chooses?
│   └─► bind_tools([tool1, tool2, ...]) — LLM picks
│
├─► Force specific tool?
│   └─► bind_tools(tools, tool_choice="tool_name")
│
├─► Parallel tool calls needed?
│   └─► LLM auto-generates multiple tool_calls → asyncio.gather()
│
├─► Structured output from tool?
│   └─► Return Pydantic model from @tool
│
└─► Tool fails often?
    └─► .with_retry().with_fallbacks() on tool Runnable
```

---

## 6.16 Complete Tool Calling Cheat Sheet

```python
# 1. CREATE TOOLS
from langchain_core.tools import tool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    max_results: int = Field(default=5)

@tool(args_schema=SearchInput)
def web_search(query: str, max_results: int = 5) -> list:
    """Search the web for information."""
    # Implementation
    return results

# 2. BIND TO LLM
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")
llm_with_tools = llm.bind_tools([web_search, get_weather, calculate])

# 3. AUTOMATIC TOOL CALLING
result = llm_with_tools.invoke("Search for LangChain tutorials")
# result.tool_calls = [{'name': 'web_search', 'args': {...}, 'id': '...'}]

# 4. EXECUTE TOOLS (Manual)
tool_messages = []
for tc in result.tool_calls:
    output = globals()[tc["name"]].invoke(tc["args"])
    tool_messages.append(ToolMessage(content=str(output), tool_call_id=tc["id"]))

# 5. FEED BACK TO LLM
final = llm_with_tools.invoke([
    HumanMessage("Search for LangChain tutorials"),
    result,
    *tool_messages
])

# 6. OR USE AGENT (Full Autonomy)
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, tools=[web_search, get_weather])
result = agent.invoke({"messages": [HumanMessage("Search for LangChain and summarize")]})
```

---

## 6.16 Summary

### Tools — Key Topics
| Topic | Covered |
|-------|---------|
| **What is a Tool?** | Function with schema for LLM interaction |
| **Built-in Tools** | DuckDuckGo, Wikipedia, Shell, Python REPL, Requests |
| **Custom Tools** | `@tool` decorator, schemas, descriptions |
| **Toolkits** | SQL, File Management, GitHub, Pandas, VectorStore |
| **Creating Tools** | Step-by-step decorator usage |

### Tool Calling — Key Topics
| Topic | Covered |
|-------|---------|
| **Tool Binding** | `llm.bind_tools([tools])` — registers tools with LLM |
| **Tool Calling** | LLM autonomously decides to invoke tools |
| **Tool Execution** | Manual loop: LLM → tool_calls → execute → ToolMessage → LLM |
| **Currency Converter** | Real-world app: get_rate → convert → answer |
| **Agent vs Chain** | Chain = programmer controls; Agent = LLM controls |

---

## 6.17 Quick Reference: Key Classes

| Category | Class/Function | Package | Purpose |
|----------|----------------|---------|---------|
| **Tool Creation** | `@tool` | `langchain_core.tools` | Decorator for creating tools |
| **Tool Schema** | `BaseModel` | `pydantic` | Input validation |
| **Tool Binding** | `bind_tools()` | `BaseChatModel` | Attach tools to LLM |
| **Tool Message** | `ToolMessage` | `langchain_core.messages` | Tool result in conversation |
| **Auto Execution** | `ToolNode` | `langgraph.prebuilt` | LangGraph auto-executor |
| **Agent** | `create_react_agent` | `langgraph.prebuilt` | Full autonomous agent |
| **Built-in Tools** | `DuckDuckGoSearchRun`, etc. | `langchain_community.tools` | Ready-to-use tools |
| **Toolkits** | `SQLDatabaseToolkit`, etc. | `langchain_community.agent_toolkits` | Tool collections |

---

## 6.18 Final References

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **Tools:** https://python.langchain.com/docs/concepts/tools/
- **Tool Calling:** https://python.langchain.com/docs/how_to/tool_calling/
- **Built-in Tools:** https://python.langchain.com/docs/integrations/tools/
- **Toolkits:** https://python.langchain.com/docs/integrations/toolkits/

### Advanced
- **LangGraph Agents:** https://langchain-ai.github.io/langgraph/
- **MCP (Model Context Protocol):** https://modelcontextprotocol.io/

---

*End of Chapter 6 — Tools & Tool Calling: Giving LLMs Hands and Feet*
