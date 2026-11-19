# LangGraph Lecture 4: LangGraph Core Concepts

## 1. LangGraph Recap: The Orchestration Framework

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGGRAPH = ORCHESTRATION FRAMEWORK                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DEFINITION:                                                                │
│  ────────────                                                               │
│  LangGraph is an orchestration framework for building intelligent,          │
│  stateful, multi-step LLM workflows.                                        │
│                                                                             │
│  CORE IDEA:                                                                 │
│  ─────────                                                                  │
│  1. Represent ANY LLM workflow as a GRAPH                                   │
│  2. Each NODE = one sub-task (LLM call, tool, decision, memory)             │
│ 3. EDGES define execution order (next task after current)                   │
│ 4. Provide input to first node → graph executes automatically               │
│                                                                             │
│  CORE FEATURES (from Lecture 3):                                            │
│  ─────────────────────────────                                              │
│  ✓ Conditional Branching    ✓ Loops/Cycles    ✓ Parallel Execution          │
│  ✓ Memory/Persistence       ✓ Resumability    ✓ Human-in-the-Loop           │
│  ✓ Multi-Agent              ✓ Streaming       ✓ Time Travel Debug           │
│                                                                             │
│  IDEAL FOR: Agentic & Production-Grade AI Applications                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. LLM Workflows: Foundation

### 2.1 Definition

> **Workflow:** A series of tasks executed in order to achieve a goal.
> **LLM Workflow:** A workflow where multiple tasks depend on LLMs (prompting, reasoning, tool calling, memory access, decision making).

### 2.2 Workflow Topologies

| Topology | Description | Example |
|----------|-------------|---------|
| **Linear** | A → B → C → D | Simple RAG: Retrieve → Prompt → Generate |
| **Parallel** | A → [B, C, D] → E | Multi-source research |
| **Branching** | A → (if X) B else C | Router: classify → specialized handler |
| **Looped** | A → B → C → (check) → B | Iterative refinement |
| **Nested** | Main: A → SubWorkflow → B | Main agent delegates to sub-agent |

> **Key Insight:** Real applications mix these topologies. LangGraph handles ALL of them.

---

## 3. Five Common LLM Workflow Patterns

### 3.1 Pattern 1: Prompt Chaining

```
PROMPT CHAINING:
─────────────────────────────────────────────────────────────────────────────
Linear sequence where each step's output feeds the next step's prompt.

A → B → C → D
│   │   │   │
│   │   │   └─ Final Output
│   │   └─ Transform/Summarize
│   └─ Process/Extract
└─ Initial Input

USE CASES:
• Document Processing: Extract → Summarize → Translate → Format
• Content Pipeline: Research → Outline → Write → Edit → Polish
• Data Cleaning: Parse → Validate → Transform → Enrich → Store

LANGRAPH IMPLEMENTATION:
─────────────────────────────────────────────────────────────────────────────
graph.add_node("extract", extract_node)
graph.add_node("summarize", summarize_node)
graph.add_node("translate", translate_node)
graph.add_node("format", format_node)

graph.add_edge(START, "extract")
graph.add_edge("extract", "summarize")
graph.add_edge("summarize", "translate")
graph.add_edge("translate", "format")
graph.add_edge("format", END)
```

### 3.2 Pattern 2: Routing

```
ROUTING:
─────────────────────────────────────────────────────────────────────────────
Classify input → Route to specialized handler.

        ┌──→ Handler A (Technical)
Input ──→ Router ──→ Handler B (Billing)
        └──→ Handler C (General)

USE CASES:
• Customer Support: Classify ticket → Specialized agent
• Content Moderation: Route by category (spam, toxic, legal, safe)
• Code Review: Route by language (Python, JS, Go, Rust)
• Medical Triage: Route by specialty (cardio, neuro, ortho)

LANGRAPH IMPLEMENTATION:
─────────────────────────────────────────────────────────────────────────────
def router(state):
    category = classify(state["input"])
    return category  # Returns: "technical", "billing", "general"

graph.add_node("classify", classify_node)
graph.add_node("technical", technical_handler)
graph.add_node("billing", billing_handler)
graph.add_node("general", general_handler)

graph.add_edge(START, "classify")
graph.add_conditional_edges("classify", router, {
    "technical": "technical",
    "billing": "billing",
    "general": "general"
})
graph.add_edge("technical", END)
graph.add_edge("billing", END)
graph.add_edge("general", END)
```

### 3.3 Pattern 3: Parallelization

```
PARALLELIZATION:
─────────────────────────────────────────────────────────────────────────────
Fan-out: One node triggers multiple parallel nodes.
Fan-in: Results merged before next step.

        ┌──→ Task A
Start ──→ ┤    Task B   ──→ Merge ──→ Next
        └──→ Task C

USE CASES:
• Multi-source Research: Search Google + Bing + DuckDuckGo simultaneously
• Code Analysis: Lint + Type-check + Security-scan in parallel
• Data Enrichment: Enrich from API A + API B + Database simultaneously
• Ensemble: Run same task with different models/prompts, vote on result

LANGRAPH IMPLEMENTATION:
─────────────────────────────────────────────────────────────────────────────
graph.add_node("search_google", google_search)
graph.add_node("search_bing", bing_search)
graph.add_node("search_ddg", duckduckgo_search)
graph.add_node("merge_results", merge_node)

graph.add_edge(START, "search_google")
graph.add_edge(START, "search_bing")
graph.add_edge(START, "search_ddg")

graph.add_edge("search_google", "merge_results")
graph.add_edge("search_bing", "merge_results")
graph.add_edge("search_ddg", "merge_results")

graph.add_edge("merge_results", "next_step")
```

### 3.4 Pattern 4: Orchestrator-Workers

```
ORCHESTRATOR-WORKERS:
─────────────────────────────────────────────────────────────────────────────
Orchestrator plans & delegates → Workers execute in parallel → Orchestrator aggregates.

                    ┌──→ Worker 1 (Research)
Orchestrator ──────┤    Worker 2 (Analyze)
  (Planner)        └──→ Worker 3 (Write)
       │
       ▼
  Aggregate Results
       │
       ▼
  Final Output

USE CASES:
• Complex Research: Planner creates sub-tasks, workers execute, planner synthesizes
• Code Generation: Architect designs, specialists implement modules, architect integrates
• Report Generation: Planner outlines sections, writers draft each, editor compiles
• Multi-step Planning: Break down goal, delegate, coordinate

LANGRAPH IMPLEMENTATION:
─────────────────────────────────────────────────────────────────────────────
def orchestrator_plan(state):
    # LLM creates execution plan
    plan = llm.invoke(f"Create plan for: {state['goal']}")
    tasks = parse_plan(plan)
    return {"tasks": tasks, "results": {}}

def worker_execute(state):
    # Each worker handles one task
    task = state["current_task"]
    result = execute_task(task)
    return {"results": {task.id: result}}

def orchestrator_aggregate(state):
    # Combine all worker results
    final = synthesize(state["results"])
    return {"final_output": final}

# Graph structure:
graph.add_node("plan", orchestrator_plan)
graph.add_node("worker", worker_execute)  # Called in parallel for each task
graph.add_node("aggregate", orchestrator_aggregate)

graph.add_edge(START, "plan")
# Fan-out to workers (dynamic based on plan)
graph.add_conditional_edges("plan", lambda s: "worker" if s["tasks"] else "aggregate")
graph.add_edge("worker", "aggregate")  # After all workers done
graph.add_edge("aggregate", END)
```

### 3.5 Pattern 5: Evaluator-Optimizer

```
EVALUATOR-OPTIMIZER:
─────────────────────────────────────────────────────────────────────────────
Generator creates → Evaluator critiques → Loop until quality threshold.

Generate ──→ Evaluate ──→ (Pass?) ──→ YES ──→ Output
  ▲           │           │
  │           └── NO ─────┘
  └───────────────────────┘  (Iterate with feedback)

USE CASES:
• Code Generation: Write → Test/Review → Fix → Repeat
• Content Creation: Draft → Critique → Improve → Repeat
• Translation: Translate → Quality Check → Refine → Repeat
• SQL Generation: Generate → Validate → Execute → Fix → Repeat

LANGRAPH IMPLEMENTATION:
─────────────────────────────────────────────────────────────────────────────
def generator(state):
    draft = llm.invoke(f"Create {state['task']}. Feedback: {state.get('feedback', '')}")
    return {"draft": draft, "iteration": state.get("iteration", 0) + 1}

def evaluator(state):
    score, feedback = evaluate_quality(state["draft"], state["criteria"])
    return {"score": score, "feedback": feedback, "passed": score >= THRESHOLD}

def should_continue(state):
    return "generate" if not state["passed"] and state["iteration"] < MAX_ITER else "end"

graph.add_node("generate", generator)
graph.add_node("evaluate", evaluator)

graph.add_edge(START, "generate")
graph.add_edge("generate", "evaluate")
graph.add_conditional_edges("evaluate", should_continue, {
    "generate": "generate",
    "end": END
})
```

---

## 4. Graph Anatomy: Nodes & Edges

### 4.1 Nodes: The Processing Units

```
NODE TYPES IN LANGGRAPH:
─────────────────────────────────────────────────────────────────────────────
┌─────────────────┬─────────────────────────────────────────────────────────┐
│ Type            │ Description                                             │
├─────────────────┼─────────────────────────────────────────────────────────┤
│ LLM Node        │ Calls LLM with prompt, returns response                 │
│ Tool Node       │ Executes function/tool, returns result                  │
│ Logic Node      │ Deterministic code (validation, routing, transform)     │
│ Human Node      │ interrupt() - pauses for human input                    │
│ Subgraph Node   │ Invokes another compiled graph                          │
│ State Node      │ Updates state without external calls                    │
└─────────────────┴─────────────────────────────────────────────────────────┘

NODE SIGNATURE:
─────────────────────────────────────────────────────────────────────────────
def my_node(state: State) -> Partial[State]:
    # Process state
    # Return ONLY the fields to UPDATE (auto-merged)
    return {"field1": new_value, "field2": new_value}

# Can also return Command for advanced control:
def my_node(state):
    return Command(
        update={"field": value},
        goto="next_node"  # Override edge routing
    )
```

### 4.2 Edges: The Flow Control

```
EDGE TYPES:
─────────────────────────────────────────────────────────────────────────────
┌─────────────────┬─────────────────────────────────────────────────────────┐
│ Type            │ Syntax                                                  │
├─────────────────┼─────────────────────────────────────────────────────────┤
│ Direct          │ graph.add_edge("A", "B")                                │
│ START           │ graph.add_edge(START, "A")                              │
│ END             │ graph.add_edge("A", END)                                │
│ Conditional     │ graph.add_conditional_edges("A", router, mapping)       │
│ Self-loop       │ graph.add_edge("A", "A")  (with condition to exit)      │
└─────────────────┴─────────────────────────────────────────────────────────┘

CONDITIONAL EDGE ROUTER:
─────────────────────────────────────────────────────────────────────────────
def router(state: State) -> str:
    # Return node name to go to next
    if state["score"] > 0.8:
        return "approve"
    elif state["score"] > 0.5:
        return "review"
    return "reject"

# Mapping: router return value → node name
graph.add_conditional_edges("evaluate", router, {
    "approve": "approve_node",
    "review": "review_node",
    "reject": "reject_node"
})

# Can also use dictionary for complex routing:
def router(state):
    return {
        "next": "node_name",
        "update": {"extra_field": "value"}  # Optional state update
    }
```

### 4.3 Complete Graph Construction

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from langgraph.graph import add_messages
import operator

# 1. DEFINE STATE
class WorkflowState(TypedDict):
    messages: Annotated[list, add_messages]
    user_input: str
    classification: str
    research_results: list
    draft: str
    review_score: float
    final_output: str

# 2. DEFINE NODES
def classify_node(state):
    # Logic node - no LLM needed
    category = classify_intent(state["user_input"])
    return {"classification": category}

def research_node(state):
    # Tool node - parallel searches
    results = parallel_search(state["classification"])
    return {"research_results": results}

def draft_node(state):
    # LLM node
    draft = llm.invoke(f"Write draft for {state['classification']} using {state['research_results']}")
    return {"draft": draft}

def review_node(state):
    # Evaluator node
    score = evaluate_quality(state["draft"])
    return {"review_score": score}

def finalize_node(state):
    # Final processing
    return {"final_output": polish(state["draft"])}

# 3. BUILD GRAPH
graph = StateGraph(WorkflowState)

# Add nodes
graph.add_node("classify", classify_node)
graph.add_node("research", research_node)
graph.add_node("draft", draft_node)
graph.add_node("review", review_node)
graph.add_node("finalize", finalize_node)

# Add edges
graph.add_edge(START, "classify")
graph.add_edge("classify", "research")
graph.add_edge("research", "draft")
graph.add_edge("draft", "review")

# Conditional: review → finalize OR back to draft
graph.add_conditional_edges("review", lambda s: "finalize" if s["review_score"] > 0.8 else "draft", {
    "finalize": "finalize",
    "draft": "draft"
})

graph.add_edge("finalize", END)

# 4. COMPILE
app = graph.compile(checkpointer=MemorySaver())

# 5. RUN
result = app.invoke(
    {"user_input": "Write a blog post about LangGraph"},
    config={"configurable": {"thread_id": "blog-1"}}
)
```

---

## 5. State: The Shared Context

### 5.1 State Definition Patterns

```
STATE DESIGN PRINCIPLES:
─────────────────────────────────────────────────────────────────────────────
1. State = Single source of truth for entire workflow
2. Passed to EVERY node automatically
3. Nodes return PARTIAL updates (merged automatically)
4. Persisted via checkpointer (enables resume, HITL, time travel)
5. Should be serializable (JSON-compatible for persistence)
```

### 5.2 TypedDict with Annotated Fields

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages
import operator

class State(TypedDict):
    # Auto-merging lists (append)
    messages: Annotated[list, add_messages]
    documents: Annotated[list, operator.add]
    search_queries: Annotated[list, operator.add]
    
    # Auto-merging dicts (update)
    metadata: Annotated[dict, operator.or_]
    
    # Replaced entirely (default)
    user_query: str
    current_plan: list
    final_answer: str
    iteration: int
    max_iterations: int
```

### 5.3 Pydantic State (Alternative)

```python
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any

class State(BaseModel):
    messages: List[Any] = Field(default_factory=list)
    documents: List[Document] = Field(default_factory=list)
    user_query: str = ""
    current_plan: List[str] = Field(default_factory=list)
    final_answer: str = ""
    iteration: int = 0
    max_iterations: int = 5
    metadata: Dict[str, Any] = Field(default_factory=dict)
    
    # Custom merge logic
    def merge(self, update: dict) -> "State":
        # Implement custom merging if needed
        return self.model_copy(update=update)
```

### 5.4 State Best Practices

| Practice | Why |
|----------|-----|
| **Minimal state** | Only include what nodes need |
| **Use Annotated for lists** | Auto-append avoids manual concatenation |
| **Separate concerns** | Different fields for different node types |
| **Plan for persistence** | All fields must be JSON-serializable |
| **Version state** | Add `schema_version` for migrations |

---

## 6. Reducers: Merging State Updates

### 6.1 What Are Reducers?

> **Reducers** are functions that define how to merge multiple state updates into a single state. They handle concurrent updates from parallel nodes.

```
REDUCER CONCEPT:
─────────────────────────────────────────────────────────────────────────────
When multiple nodes run in PARALLEL (superstep), they each return state updates.
Reducers determine how to COMBINE these updates into the new state.

DEFAULT REDUCERS:
─────────────────
• dict: Shallow merge (later wins)
• list: Concatenate (extend)
• Annotated[list, add_messages]: Smart message merging
• Custom: Your function
```

### 6.2 Built-in Reducers

```python
from typing import Annotated
from langgraph.graph import add_messages
import operator

class State(TypedDict):
    # 1. MESSAGES: Smart merging (handles AIMessage, HumanMessage, ToolMessage)
    messages: Annotated[list, add_messages]
    
    # 2. LISTS: Simple concatenation
    documents: Annotated[list, operator.add]
    search_results: Annotated[list, operator.add]
    errors: Annotated[list, operator.add]
    
    # 3. DICTS: Shallow merge
    metadata: Annotated[dict, operator.or_]
    config: Annotated[dict, operator.or_]
    
    # 4. PRIMITIVES: Replace (default behavior)
    user_query: str
    iteration: int
    status: str
```

### 6.3 Custom Reducers

```python
from typing import Annotated, Callable

# Custom reducer for deduplicated list merge
def dedup_merge(left: list, right: list) -> list:
    seen = set()
    result = []
    for item in left + right:
        key = item.get("id") if isinstance(item, dict) else item
        if key not in seen:
            seen.add(key)
            result.append(item)
    return result

class State(TypedDict):
    unique_results: Annotated[list, dedup_merge]
    # Usage: parallel nodes return {"unique_results": [{"id": 1, "data": "x"}]}
    # Merged automatically with deduplication

# Custom reducer for max value
def max_reducer(left: int, right: int) -> int:
    return max(left, right)

class State(TypedDict):
    best_score: Annotated[int, max_reducer]
    # Parallel evaluators return {"best_score": 0.9}, merged takes max

# Custom reducer for accumulating costs
def cost_reducer(left: float, right: float) -> float:
    return left + right

class State(TypedDict):
    total_cost: Annotated[float, cost_reducer]
```

### 6.4 Reducers in Supersteps (Parallel Execution)

```
SUPERSTEP EXECUTION WITH REDUCERS:
─────────────────────────────────────────────────────────────────────────────

STEP: 3 parallel nodes execute simultaneously
      │
      ▼
Node A returns: {"documents": [doc1], "cost": 0.05}
Node B returns: {"documents": [doc2], "cost": 0.03}
Node C returns: {"documents": [doc3], "cost": 0.02}
      │
      ▼
REDUCERS APPLY:
─────────────────────────────────────────────────────────────────────────────
documents: operator.add      → [doc1, doc2, doc3]
cost:      cost_reducer      → 0.10 (sum)
      │
      ▼
MERGED STATE passed to next node
```

---

## 7. LangGraph Execution Model

### 7.1 Supersteps: The Atomic Unit

```
SUPERSTEP DEFINITION:
─────────────────────────────────────────────────────────────────────────────
A "superstep" = One wave of parallel node executions.

WHY "SUPER" STEP?
─────────────────
• Single node = 1 step
• Multiple parallel nodes = 1 superstep (they execute simultaneously)
• Graph execution = Sequence of supersteps

EXECUTION FLOW:
─────────────────────────────────────────────────────────────────────────────
Superstep 1:  [Node A]                    → State updated
Superstep 2:  [Node B, Node C, Node D]    → All run in parallel, reducers merge
Superstep 3:  [Node E]                    → State updated
Superstep 4:  [Node F, Node G]            → Parallel again
...
Until: No active nodes AND no messages in edges
```

### 7.2 Message Passing & State Flow

```
MESSAGE PASSING MECHANICS:
─────────────────────────────────────────────────────────────────────────────

1. NODE EXECUTION:
   - Node receives current state
   - Executes logic (LLM, tool, etc.)
   - Returns partial state update

2. REDUCER MERGE:
   - If single node: direct update
   - If parallel (superstep): reducers combine all updates

3. EDGE TRAVERSAL:
   - Updated state sent along outgoing edges
   - Conditional edges evaluate router function
   - Next node(s) receive state

4. REPEAT:
   - Continue until no active nodes

TERMINATION CONDITIONS:
─────────────────────────────────────────────────────────────────────────────
Graph stops when BOTH are true:
1. No active nodes (all current nodes completed)
2. No messages in transit on edges (no pending state transfers)
```

### 7.3 Execution Modes

```python
# 1. SYNCHRONOUS (blocking)
result = app.invoke(input, config)

# 2. ASYNCHRONOUS (non-blocking)
result = await app.ainvoke(input, config)

# 3. STREAMING - Values (state after each superstep)
async for state in app.astream(input, config, stream_mode="values"):
    print(f"State: {state}")

# 4. STREAMING - Updates (node-level events)
async for update in app.astream(input, config, stream_mode="updates"):
    print(f"Node finished: {update}")

# 5. STREAMING - Messages (LLM tokens)
async for msg in app.astream(input, config, stream_mode="messages"):
    print(f"Token: {msg}")

# 6. STREAMING - Custom (debug)
async for chunk in app.astream(input, config, stream_mode="custom"):
    print(f"Debug: {chunk}")
```

### 7.4 Configuration Options

```python
config = {
    "configurable": {
        "thread_id": "user-123",           # Required for persistence
        "checkpoint_ns": "namespace",      # Optional: namespace checkpoints
        "recursion_limit": 50,             # Max supersteps (default: 25)
    },
    "tags": ["production", "v1.2"],        # For LangSmith filtering
    "metadata": {"user_tier": "premium"},  # Custom metadata
    "run_name": "blog_generation",         # For tracing
}

# Recursion limit prevents infinite loops
# Increase for iterative workflows:
config["configurable"]["recursion_limit"] = 100
```

### 7.5 Checkpointing & Resumability

```python
# Checkpointer saves state AFTER EACH SUPERSTEP
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# In-memory (dev)
checkpointer = MemorySaver()

# SQLite (production)
checkpointer = SqliteSaver.from_conn_string("sqlite:///checkpoints.db")

# PostgreSQL (scale)
from langgraph.checkpoint.postgres import PostgresSaver
checkpointer = PostgresSaver.from_conn_string("postgresql://...")

# Compile with checkpointer
app = graph.compile(checkpointer=checkpointer)

# Run with thread_id (enables persistence)
config = {"configurable": {"thread_id": "session-1"}}
result = app.invoke(input, config=config)

# RESUME LATER - same thread_id continues from last checkpoint
result2 = app.invoke({"messages": [HumanMessage("continue")]}, config=config)

# TIME TRAVEL - Get state at any checkpoint
history = list(app.get_state_history(config))
# history[0] = latest, history[-1] = first

# Restore to specific checkpoint
past_config = history[3].config  # 4th checkpoint
state_at_that_point = app.get_state(past_config)
# Can resume from there with modified input
app.invoke(modified_input, config=past_config)
```

---

## 8. Visual Debugging & Graph Inspection

```python
# 1. MERMAID DIAGRAM (for documentation)
mermaid_code = app.get_graph().draw_mermaid()
print(mermaid_code)
# Paste into https://mermaid.live/ to visualize

# 2. ASCII GRAPH (terminal)
print(app.get_graph().draw_ascii())

# 3. GRAPH STRUCTURE PROGRAMMATIC
graph = app.get_graph()
nodes = graph.nodes  # Dict[node_id, Node]
edges = graph.edges  # List[Edge]

for node_id, node in nodes.items():
    print(f"Node: {node_id}, Type: {type(node).__name__}")

for edge in edges:
    print(f"Edge: {edge.source} → {edge.target}, Conditional: {edge.conditional}")

# 4. LANGSMITH TRACING (automatic with env vars)
# LANGCHAIN_TRACING_V2=true
# LANGCHAIN_API_KEY=...
# View at https://smith.langchain.com/
```

---

## 9. Complete Example: Research Agent with All Concepts

```python
# ============================================
# COMPLETE RESEARCH AGENT: ALL CORE CONCEPTS
# ============================================

from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
from langgraph.graph import add_messages
import operator

# 1. STATE WITH REDUCERS
class ResearchState(TypedDict):
    topic: str
    queries: Annotated[list, operator.add]
    results: Annotated[list, operator.add]
    summaries: Annotated[list, operator.add]
    current_draft: str
    review_score: float
    feedback: str
    iteration: int
    final_report: str

# 2. NODES

def generate_queries(state):
    """LLM Node: Generate search queries"""
    prompt = f"Generate 3 diverse search queries for: {state['topic']}"
    queries = llm.invoke(prompt).split("\n")
    return {"queries": queries, "iteration": state.get("iteration", 0) + 1}

def parallel_search(state):
    """Tool Node (Parallel): Search all queries simultaneously"""
    # This runs as superstep with 3 parallel invocations
    results = []
    for q in state["queries"]:
        results.append(search_tool(q))
    return {"results": results}

def summarize_results(state):
    """LLM Node: Summarize each result"""
    summaries = []
    for r in state["results"]:
        summary = llm.invoke(f"Summarize key points: {r}")
        summaries.append(summary)
    return {"summaries": summaries}

def draft_report(state):
    """LLM Node: Create draft from summaries"""
    draft = llm.invoke(f"Write report on {state['topic']} using: {state['summaries']}")
    return {"current_draft": draft}

def review_draft(state):
    """Evaluator Node: Score quality"""
    score = evaluate_quality(state["current_draft"], criteria=["accuracy", "completeness"])
    feedback = generate_feedback(state["current_draft"]) if score < 0.85 else ""
    return {"review_score": score, "feedback": feedback}

def refine_draft(state):
    """LLM Node: Improve based on feedback"""
    refined = llm.invoke(f"Improve: {state['current_draft']}\nFeedback: {state['feedback']}")
    return {"current_draft": refined}

def finalize(state):
    """Final Node: Polish and format"""
    final = llm.invoke(f"Final polish: {state['current_draft']}")
    return {"final_report": final}

# 3. ROUTER FOR EVALUATOR-OPTIMIZER LOOP
def should_refine(state):
    if state["review_score"] >= 0.85 or state["iteration"] >= 3:
        return "finalize"
    return "refine"

# 4. BUILD GRAPH
graph = StateGraph(ResearchState)

# Add all nodes
graph.add_node("generate_queries", generate_queries)
graph.add_node("parallel_search", parallel_search)
graph.add_node("summarize", summarize_results)
graph.add_node("draft", draft_report)
graph.add_node("review", review_draft)
graph.add_node("refine", refine_draft)
graph.add_node("finalize", finalize)

# Edges
graph.add_edge(START, "generate_queries")
graph.add_edge("generate_queries", "parallel_search")  # Parallel superstep
graph.add_edge("parallel_search", "summarize")
graph.add_edge("summarize", "draft")
graph.add_edge("draft", "review")

# Conditional: Evaluator-Optimizer loop
graph.add_conditional_edges("review", should_refine, {
    "refine": "refine",
    "finalize": "finalize"
})

graph.add_edge("refine", "review")  # Loop back
graph.add_edge("finalize", END)

# 5. COMPILE WITH PERSISTENCE
app = graph.compile(checkpointer=MemorySaver())

# 6. RUN
config = {"configurable": {"thread_id": "research-001"}}
result = app.invoke({"topic": "LangGraph vs LangChain"}, config=config)

# 7. STREAM PROGRESS
async for update in app.astream({"topic": "AI Agents"}, config, stream_mode="updates"):
    print(f"Completed: {list(update.keys())[0]}")
```

---

## 10. Quick Reference: Core Concepts Cheat Sheet

```python
# ============================================
# LANGGRAPH CORE CONCEPTS CHEAT SHEET
# ============================================

# 1. WORKFLOW PATTERNS
# ────────────────────
PROMPT_CHAINING    → Linear: A → B → C → D
ROUTING            → Classify → Branch: A → (if/else) → B/C
PARALLELIZATION    → Fan-out/Fan-in: A → [B,C,D] → E
ORCHESTRATOR-WORKERS → Plan → Parallel Workers → Aggregate
EVALUATOR-OPTIMIZER → Generate → Evaluate → Loop until pass

# 2. GRAPH COMPONENTS
# ───────────────────
State  → TypedDict with Annotated fields (shared context)
Nodes  → Functions: (state) → Partial[State] updates
Edges  → Direct, Conditional, START, END
Graph  → StateGraph(State) + nodes + edges → compile()

# 3. NODE TYPES
# ─────────────
LLM Node      → prompt | model | parser
Tool Node     → tool.invoke(args)
Logic Node    → deterministic code
Human Node    → interrupt({question}) → Command(resume=...)
Subgraph Node → subgraph_app.invoke(input)

# 4. REDUCERS (Annotated fields)
# ──────────────────────────────
messages:      Annotated[list, add_messages]      # Smart message merge
documents:     Annotated[list, operator.add]      # Concatenate
metadata:      Annotated[dict, operator.or_]      # Shallow merge
best_score:    Annotated[float, max]              # Custom: max
total_cost:    Annotated[float, lambda a,b: a+b]  # Custom: sum

# 5. EXECUTION MODEL
# ──────────────────
Superstep = One wave of parallel node executions
1. Nodes receive state
2. Execute & return updates
3. Reducers merge (if parallel)
4. State sent via edges
5. Router evaluates conditional edges
6. Next superstep begins
7. Stop when no active nodes + no pending messages

# 6. COMPILATION & RUN
# ────────────────────
app = graph.compile(checkpointer=MemorySaver())

# Invoke
result = app.invoke(input, config={"thread_id": "x"})

# Stream
async for chunk in app.astream(input, config, stream_mode="values"):
    ...

# 7. PERSISTENCE & RESUME
# ───────────────────────
checkpointer = SqliteSaver("sqlite:///db")
app = graph.compile(checkpointer=checkpointer)

# Resume
app.invoke(new_input, config={"thread_id": "same"})

# Time travel
history = app.get_state_history(config)
app.invoke(input, config=history[3].config)

# 8. DEBUGGING
# ────────────
app.get_graph().draw_mermaid()    # Visual diagram
app.get_graph().draw_ascii()      # Terminal view
LangSmith tracing (auto with env) # Full observability
```

---

## 11. Key Takeaways

| Concept | Essence |
|---------|---------|
| **5 Workflow Patterns** | Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer |
| **Graph = State + Nodes + Edges** | State (shared context), Nodes (processing), Edges (flow) |
| **State with Reducers** | Annotated fields define merge behavior for parallel updates |
| **Supersteps** | Atomic execution unit: parallel nodes run simultaneously, reducers merge |
| **Message Passing** | State flows along edges; conditional edges route dynamically |
| **Persistence** | Checkpointer saves after each superstep → resume, HITL, time travel |
| **Execution Modes** | invoke, ainvoke, astream (values/updates/messages) |

---

## 12. Resources & References

### Videos
- **Playlist:** https://www.youtube.com/watch?v=yC36gN-rqjo&list=PLKnIA16_RmvYsvB8qkUQuJmJNuiCUJFPL

### Official Documentation
- **Graph API:** https://docs.langchain.com/oss/python/langgraph/graph-api
- **State:** https://docs.langchain.com/oss/python/langgraph/concepts/state
- **Persistence:** https://docs.langchain.com/oss/python/langgraph/persistence
- **Streaming:** https://docs.langchain.com/oss/python/langgraph/streaming
- **Human-in-the-Loop:** https://docs.langchain.com/oss/python/langgraph/interrupts
