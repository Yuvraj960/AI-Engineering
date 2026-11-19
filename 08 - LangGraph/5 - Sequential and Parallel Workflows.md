# LangGraph Lecture 5: Sequential & Parallel Workflows

## 1. Overview: The Two Fundamental Workflow Topologies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WORKFLOW TOPOLOGY SPECTRUM                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SEQUENTIAL (Linear)                    PARALLEL (Fan-out/Fan-in)           │
│  ─────────────────────                  ─────────────────────────           │
│                                                                             │
│  A → B → C → D → E                        A → [B, C, D] → E                 │
│  │   │   │   │   │                        │   │   │   │                     │
│  │   │   │   │   └── Step 5               │   │   │   └── Task 3            │
│  │   │   │   └────── Step 4               │   │   └────── Task 2            │
│  │   │   └────────── Step 3               │   └────────── Task 1            │
│  │   └────────────── Step 2               │                                 │
│  └────────────────── Step 1               └── Then MERGE results            │
│                                                                             │
│  CHARACTERISTICS:                        CHARACTERISTICS:                   │
│  • One active node at a time             • Multiple nodes simultaneous      │
│  • Simple, predictable                   • Faster (wall-clock time)         │
│  • Easy to debug                         • Complex state merging            │
│  • Output of N = Input of N+1            • Reducers combine results         │
│  • No race conditions                    • Independent tasks required       │
│                                                                             │
│  USE WHEN:                                USE WHEN:                         │
│  • Steps depend on previous output       • Tasks are independent            │
│  • Strict ordering required              • Speed is critical                │
│  • Simple pipelines                      • Batch processing                 │
│  • Debugging priority                    • Multi-source aggregation         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Key Insight:** Real applications COMBINE both. Sequential for dependent steps, parallel for independent ones.

---

## PART A: SEQUENTIAL WORKFLOWS

---

## 2. Sequential Workflow Fundamentals

### 2.1 Definition & Characteristics

> **Sequential Workflow:** A linear chain of tasks where each task executes only after the previous one completes. The output of task N becomes the input to task N+1.

```
SEQUENTIAL WORKFLOW ANATOMY:
─────────────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────────────────┐
│  START                                                                  │
│   │                                                                     │
│   ▼                                                                     │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │   NODE 1    │────▶│   NODE 2    │────▶│   NODE 3    │  ...           │
│  │  (Process)  │     │  (Process)  │     │  (Process)  │                │
│  └─────────────┘     └─────────────┘     └─────────────┘                │
│       │                    │                    │                       │
│       ▼                    ▼                    ▼                       │
│  State Update         State Update         State Update                 │
│       │                    │                    │                       │
│       └────────────────────┴────────────────────┘                       │
│                            │                                            │
│                            ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    SHARED STATE (Accumulates)                   │    │
│  │  { input, step1_result, step2_result, step3_result, ... }       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                            │                                            │
│                            ▼                                            │
│                           END                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 When Sequential is the Right Choice

| Scenario | Sequential? | Why |
|----------|-------------|-----|
| Document: Parse → Validate → Transform → Store | ✅ Yes | Each step needs previous output |
| RAG: Retrieve → Rerank → Prompt → Generate | ✅ Yes | Pipeline dependency |
| Code: Generate → Lint → Test → Deploy | ✅ Yes | Gates depend on prior results |
| Multi-source Research: Search A, B, C simultaneously | ❌ No | Independent → Use Parallel |
| Classification → Route to Handler | ✅ Partially | Route is sequential after classification |

---

## 3. Building Sequential Workflows: Complete Guide

### 3.1 Basic Sequential Pattern

```python
# ============================================
# BASIC SEQUENTIAL WORKFLOW IN LANGGRAPH
# ============================================

from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from langgraph.graph import add_messages
import operator

# 1. DEFINE STATE (accumulates through pipeline)
class SequentialState(TypedDict):
    # Input
    raw_input: str
    
    # Pipeline results (accumulated)
    step1_output: str
    step2_output: str
    step3_output: str
    final_result: str
    
    # Metadata
    errors: Annotated[list, operator.add]
    processing_time: float

# 2. DEFINE NODES (each does ONE thing)
def step1_preprocess(state: SequentialState) -> SequentialState:
    """Step 1: Clean and normalize input"""
    import time
    start = time.time()
    
    cleaned = state["raw_input"].strip().lower()
    # ... preprocessing logic ...
    
    return {
        "step1_output": cleaned,
        "processing_time": time.time() - start
    }

def step2_transform(state: SequentialState) -> SequentialState:
    """Step 2: Transform processed data"""
    import time
    start = time.time()
    
    # Uses step1_output
    transformed = f"TRANSFORMED: {state['step1_output'].upper()}"
    # ... transformation logic ...
    
    return {
        "step2_output": transformed,
        "processing_time": time.time() - start
    }

def step3_finalize(state: SequentialState) -> SequentialState:
    """Step 3: Produce final result"""
    import time
    start = time.time()
    
    # Uses step2_output
    final = f"FINAL: {state['step2_output']} [COMPLETE]"
    # ... finalization logic ...
    
    return {
        "step3_output": final,
        "final_result": final,
        "processing_time": time.time() - start
    }

# 3. BUILD GRAPH
graph = StateGraph(SequentialState)

# Add nodes
graph.add_node("preprocess", step1_preprocess)
graph.add_node("transform", step2_transform)
graph.add_node("finalize", step3_finalize)

# Add SEQUENTIAL edges
graph.add_edge(START, "preprocess")
graph.add_edge("preprocess", "transform")
graph.add_edge("transform", "finalize")
graph.add_edge("finalize", END)

# 4. COMPILE & RUN
app = graph.compile()

# Execute
result = app.invoke({"raw_input": "  hello world  "})
print(result["final_result"])
# Output: "FINAL: TRANSFORMED: HELLO WORLD [COMPLETE]"
```

### 3.2 Sequential with LangChain Components (Real-World)

```python
# ============================================
# REAL-WORLD: DOCUMENT PROCESSING PIPELINE
# ============================================

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import PyPDFLoader

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class DocProcessingState(TypedDict):
    file_path: str
    raw_text: str
    chunks: list
    summaries: Annotated[list, operator.add]
    combined_summary: str
    key_points: list
    final_report: str

def load_document(state: DocProcessingState):
    """Step 1: Load PDF"""
    loader = PyPDFLoader(state["file_path"])
    docs = loader.load()
    full_text = "\n".join([d.page_content for d in docs])
    return {"raw_text": full_text}

def chunk_text(state: DocProcessingState):
    """Step 2: Split into chunks"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=2000, chunk_overlap=200
    )
    chunks = splitter.split_text(state["raw_text"])
    return {"chunks": chunks}

def summarize_chunk(state: DocProcessingState):
    """Step 3: Summarize EACH chunk (can be parallelized later)"""
    prompt = ChatPromptTemplate.from_template(
        "Summarize this text in 3 bullet points:\n{text}"
    )
    chain = prompt | llm | StrOutputParser()
    
    summaries = []
    for chunk in state["chunks"]:
        summary = chain.invoke({"text": chunk})
        summaries.append(summary)
    
    return {"summaries": summaries}

def combine_summaries(state: DocProcessingState):
    """Step 4: Combine all summaries"""
    combined = "\n\n---\n\n".join(state["summaries"])
    return {"combined_summary": combined}

def extract_key_points(state: DocProcessingState):
    """Step 5: Extract key points from combined summary"""
    prompt = ChatPromptTemplate.from_template(
        "Extract 5 key points from this summary:\n{summary}"
    )
    parser = JsonOutputParser()
    chain = prompt | llm | parser
    
    points = chain.invoke({"summary": state["combined_summary"]})
    return {"key_points": points.get("key_points", [])}

def generate_report(state: DocProcessingState):
    """Step 6: Generate final report"""
    prompt = ChatPromptTemplate.from_template(
        """Create a professional report from:
        Summary: {summary}
        Key Points: {points}
        
        Format: Title, Executive Summary, Details, Conclusion"""
    )
    chain = prompt | llm | StrOutputParser()
    
    report = chain.invoke({
        "summary": state["combined_summary"],
        "points": state["key_points"]
    })
    return {"final_report": report}

# BUILD PIPELINE
graph = StateGraph(DocProcessingState)

graph.add_node("load", load_document)
graph.add_node("chunk", chunk_text)
graph.add_node("summarize", summarize_chunk)
graph.add_node("combine", combine_summaries)
graph.add_node("extract", extract_key_points)
graph.add_node("report", generate_report)

# SEQUENTIAL FLOW
graph.add_edge(START, "load")
graph.add_edge("load", "chunk")
graph.add_edge("chunk", "summarize")
graph.add_edge("summarize", "combine")
graph.add_edge("combine", "extract")
graph.add_edge("extract", "report")
graph.add_edge("report", END)

app = graph.compile()

# Run
result = app.invoke({"file_path": "research_paper.pdf"})
print(result["final_report"])
```

### 3.3 Error Handling in Sequential Workflows

```python
# ============================================
# ROBUST SEQUENTIAL: ERROR HANDLING & RETRY
# ============================================

from langgraph.graph import RetryPolicy
import logging

class RobustState(TypedDict):
    input_data: str
    step_results: dict
    errors: Annotated[list, operator.add]
    retry_count: dict
    final_output: str

def safe_node(node_name: str, func, max_retries=3):
    """Wrapper that adds error handling to any node"""
    def wrapper(state: RobustState):
        retries = state.get("retry_count", {}).get(node_name, 0)
        
        try:
            result = func(state)
            # Success: reset retry count
            return {
                **result,
                "retry_count": {**state.get("retry_count", {}), node_name: 0}
            }
        except Exception as e:
            logging.error(f"{node_name} failed (attempt {retries+1}): {e}")
            
            if retries < max_retries:
                # Will retry via conditional edge
                return {
                    "errors": [f"{node_name}: {str(e)}"],
                    "retry_count": {**state.get("retry_count", {}), node_name: retries + 1}
                }
            else:
                # Max retries exceeded
                return {
                    "errors": [f"{node_name} FAILED after {max_retries} retries: {e}"],
                    "final_output": f"ERROR: {node_name} failed permanently"
                }
    
    return wrapper

# Usage with RetryPolicy
graph.add_node(
    "unreliable_step",
    safe_node("unreliable_step", my_risky_function),
    retry=RetryPolicy(max_attempts=3, backoff_factor=2.0)
)

# Conditional edge for retry logic
def should_retry(state):
    if state.get("errors") and state["retry_count"].get("unreliable_step", 0) < 3:
        return "retry"
    return "continue"

graph.add_conditional_edges("unreliable_step", should_retry, {
    "retry": "unreliable_step",  # Loop back
    "continue": "next_step"
})
```

### 3.4 Streaming Sequential Results

```python
# ============================================
# STREAMING INTERMEDIATE RESULTS
# ============================================

# Stream mode "values" = state after each node
async for state in app.astream(input, config, stream_mode="values"):
    # Shows accumulated state after each step
    print(f"Completed: {list(state.keys())[-1]}")
    print(f"Current state keys: {list(state.keys())}")

# Stream mode "updates" = node-level events
async for update in app.astream(input, config, stream_mode="updates"):
    node_name = list(update.keys())[0]
    node_output = update[node_name]
    print(f"Node '{node_name}' finished with keys: {list(node_output.keys())}")

# Stream mode "messages" = LLM tokens (if using messages)
async for msg in app.astream(input, config, stream_mode="messages"):
    print(msg.content, end="", flush=True)
```

---

## 4. Advanced Sequential Patterns

### 4.1 Conditional Sequential (Branching within Sequence)

```python
# ============================================
# CONDITIONAL SEQUENTIAL: Dynamic Path Selection
# ============================================

class ConditionalState(TypedDict):
    input: str
    route: str
    path_a_result: str
    path_b_result: str
    final_result: str

def classifier(state):
    """Determine which path to take"""
    if "technical" in state["input"].lower():
        return {"route": "technical"}
    elif "billing" in state["input"].lower():
        return {"route": "billing"}
    return {"route": "general"}

def technical_handler(state):
    return {"path_a_result": "Technical analysis complete"}

def billing_handler(state):
    return {"path_b_result": "Billing analysis complete"}

def general_handler(state):
    return {"path_a_result": "General analysis complete"}

def merge_results(state):
    result = state.get("path_a_result") or state.get("path_b_result")
    return {"final_result": f"Processed via {state['route']}: {result}"}

graph = StateGraph(ConditionalState)

graph.add_node("classify", classifier)
graph.add_node("technical", technical_handler)
graph.add_node("billing", billing_handler)
graph.add_node("general", general_handler)
graph.add_node("merge", merge_results)

graph.add_edge(START, "classify")

# CONDITIONAL BRANCHING within sequence
graph.add_conditional_edges("classify", lambda s: s["route"], {
    "technical": "technical",
    "billing": "billing",
    "general": "general"
})

# All paths converge
graph.add_edge("technical", "merge")
graph.add_edge("billing", "merge")
graph.add_edge("general", "merge")
graph.add_edge("merge", END)
```

### 4.2 Sequential with Checkpointing (Resume Capability)

```python
# ============================================
# SEQUENTIAL WITH PERSISTENCE
# ============================================

from langgraph.checkpoint.sqlite import SqliteSaver

# SQLite checkpointer for production
checkpointer = SqliteSaver.from_conn_string("sqlite:///sequential_workflows.db")

app = graph.compile(checkpointer=checkpointer)

# Run with thread_id
config = {"configurable": {"thread_id": "doc-processing-001"}}
result = app.invoke({"file_path": "large_doc.pdf"}, config=config)

# If crashes at step 3, RESUME from checkpoint:
# Same thread_id continues from last successful node
result2 = app.invoke({"resume": True}, config=config)

# Time travel: inspect history
history = list(app.get_state_history(config))
for i, state in enumerate(history):
    print(f"Checkpoint {i}: {state.values.keys()}")

# Resume from specific checkpoint
past_config = history[2].config  # After "chunk" step
app.invoke({"chunks": custom_chunks}, config=past_config)
```

---

## PART B: PARALLEL WORKFLOWS

---

## 5. Parallel Workflow Fundamentals

### 5.1 Definition & Architecture

> **Parallel Workflow:** Multiple independent tasks execute simultaneously (fan-out), then results are merged (fan-in) before continuing.

```
PARALLEL WORKFLOW ANATOMY:
─────────────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────────────────┐
│           START                                                         │
│             │                                                           │
│             ▼                                                           │
│     ┌─────────────┐                                                     │
│     │  SPLITTER   │  (Single node that fans out)                        │
│     └──────┬──────┘                                                     │
│            │                                                            │
│    ┌───────┼───────┐                                                    │
│    ▼       ▼       ▼                                                    │
│ ┌──────┐┌──────┐┌──────┐                                                │
│ │Task A││Task B││Task C│  ← SUPERSTEP (all run in PARALLEL)             │
│ └──┬───┘└──┬───┘└──┬───┘                                                │
│    │       │       │                                                    │
│    ▼       ▼       ▼                                                    │
│ ┌─────────────────────┐                                                 │
│ │        MERGER       │  (Reducers combine results)                     │
│ └──────────┬──────────┘                                                 │
│            │                                                            │
│            ▼                                                            │
│           END                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Key Concept: Supersteps

```
SUPERSTEP EXECUTION:
─────────────────────────────────────────────────────────────────────────────
When multiple nodes have the SAME predecessor, they form a SUPERSTEP.

Sequential:     Superstep 1: [A] → Superstep 2: [B] → Superstep 3: [C]
Parallel:       Superstep 1: [A] → Superstep 2: [B, C, D] → Superstep 3: [E]
                                    ↑
                         These 3 run SIMULTANEOUSLY
                         Reducers merge their outputs
```

> **Critical:** Parallel nodes in a superstep see the SAME input state. Their outputs are merged via reducers.

---

## 6. Building Parallel Workflows: Complete Guide

### 6.1 Basic Parallel Pattern (Non-LLM)

```python
# ============================================
# BASIC PARALLEL WORKFLOW
# ============================================

from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
import operator

class CricketState(TypedDict):
    batsman_name: str
    # Input stats
    runs: int
    balls: int
    fours: int
    sixes: int
    matches: int
    
    # Parallel computations (use reducers to merge)
    strike_rate: Annotated[float, operator.add]  # Will be set by one node
    batting_average: Annotated[float, operator.add]
    boundary_percentage: Annotated[float, operator.add]
    consistency_score: Annotated[float, operator.add]
    
    # Final merged result
    analysis_report: str

def calc_strike_rate(state: CricketState):
    """Task 1: Calculate strike rate"""
    sr = (state["runs"] / state["balls"]) * 100 if state["balls"] > 0 else 0
    return {"strike_rate": sr}

def calc_batting_avg(state: CricketState):
    """Task 2: Calculate batting average"""
    avg = state["runs"] / state["matches"] if state["matches"] > 0 else 0
    return {"batting_average": avg}

def calc_boundary_pct(state: CricketState):
    """Task 3: Calculate boundary percentage"""
    boundary_runs = (state["fours"] * 4) + (state["sixes"] * 6)
    pct = (boundary_runs / state["runs"]) * 100 if state["runs"] > 0 else 0
    return {"boundary_percentage": pct}

def calc_consistency(state: CricketState):
    """Task 4: Calculate consistency score"""
    # Simplified: runs per match variance proxy
    avg = state["runs"] / state["matches"] if state["matches"] > 0 else 0
    consistency = min(avg / 50, 1.0)  # Normalize
    return {"consistency_score": consistency}

def generate_report(state: CricketState):
    """Merge: Generate final report from all computed stats"""
    report = f"""
    BATSMAN ANALYSIS: {state['batsman_name']}
    ═══════════════════════════════
    Strike Rate: {state['strike_rate']:.2f}
    Batting Average: {state['batting_average']:.2f}
    Boundary %: {state['boundary_percentage']:.2f}%
    Consistency: {state['consistency_score']:.2f}/1.0
    """
    return {"analysis_report": report}

# BUILD GRAPH
graph = StateGraph(CricketState)

# Add parallel nodes
graph.add_node("strike_rate", calc_strike_rate)
graph.add_node("batting_avg", calc_batting_avg)
graph.add_node("boundary_pct", calc_boundary_pct)
graph.add_node("consistency", calc_consistency)
graph.add_node("report", generate_report)

# FAN-OUT: All from START
graph.add_edge(START, "strike_rate")
graph.add_edge(START, "batting_avg")
graph.add_edge(START, "boundary_pct")
graph.add_edge(START, "consistency")

# FAN-IN: All to report
graph.add_edge("strike_rate", "report")
graph.add_edge("batting_avg", "report")
graph.add_edge("boundary_pct", "report")
graph.add_edge("consistency", "report")

graph.add_edge("report", END)

# COMPILE & RUN
app = graph.compile()

result = app.invoke({
    "batsman_name": "Virat Kohli",
    "runs": 12000,
    "balls": 13000,
    "fours": 1100,
    "sixes": 120,
    "matches": 250
})

print(result["analysis_report"])
```

### 6.2 Parallel LLM Workflows

```python
# ============================================
# PARALLEL LLM WORKFLOW: MULTI-SOURCE RESEARCH
# ============================================

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class ResearchState(TypedDict):
    topic: str
    # Parallel results (reducers merge lists)
    google_results: Annotated[list, operator.add]
    bing_results: Annotated[list, operator.add]
    arxiv_results: Annotated[list, operator.add]
    wiki_results: Annotated[list, operator.add]
    
    # Merged
    all_sources: list
    synthesis: str
    final_report: str

def search_google(state: ResearchState):
    """Parallel Task 1: Google search"""
    prompt = ChatPromptTemplate.from_template(
        "Generate 3 Google search queries for: {topic}"
    )
    chain = prompt | llm | StrOutputParser()
    queries = chain.invoke({"topic": state["topic"]}).split("\n")
    
    # Simulate search (replace with real search API)
    results = [f"Google result for: {q}" for q in queries[:3]]
    return {"google_results": results}

def search_bing(state: ResearchState):
    """Parallel Task 2: Bing search"""
    prompt = ChatPromptTemplate.from_template(
        "Generate 3 Bing search queries for: {topic}"
    )
    chain = prompt | llm | StrOutputParser()
    queries = chain.invoke({"topic": state["topic"]}).split("\n")
    
    results = [f"Bing result for: {q}" for q in queries[:3]]
    return {"bing_results": results}

def search_arxiv(state: ResearchState):
    """Parallel Task 3: ArXiv search"""
    prompt = ChatPromptTemplate.from_template(
        "Generate 3 ArXiv search queries for: {topic}"
    )
    chain = prompt | llm | StrOutputParser()
    queries = chain.invoke({"topic": state["topic"]}).split("\n")
    
    results = [f"ArXiv paper: {q}" for q in queries[:3]]
    return {"arxiv_results": results}

def search_wikipedia(state: ResearchState):
    """Parallel Task 4: Wikipedia search"""
    prompt = ChatPromptTemplate.from_template(
        "Generate 3 Wikipedia search queries for: {topic}"
    )
    chain = prompt | llm | StrOutputParser()
    queries = chain.invoke({"topic": state["topic"]}).split("\n")
    
    results = [f"Wiki article: {q}" for q in queries[:3]]
    return {"wiki_results": results}

def merge_sources(state: ResearchState):
    """Merge all sources into single list"""
    all_src = (
        state.get("google_results", []) +
        state.get("bing_results", []) +
        state.get("arxiv_results", []) +
        state.get("wiki_results", [])
    )
    return {"all_sources": all_src}

def synthesize(state: ResearchState):
    """Synthesize findings from all sources"""
    sources_text = "\n".join(state["all_sources"])
    prompt = ChatPromptTemplate.from_template(
        "Synthesize a comprehensive overview from these sources:\n{sources}"
    )
    chain = prompt | llm | StrOutputParser()
    synthesis = chain.invoke({"sources": sources_text})
    return {"synthesis": synthesis}

def final_report(state: ResearchState):
    """Generate final formatted report"""
    prompt = ChatPromptTemplate.from_template(
        "Create a professional research report on {topic}:\n{synthesis}"
    )
    chain = prompt | llm | StrOutputParser()
    report = chain.invoke({
        "topic": state["topic"],
        "synthesis": state["synthesis"]
    })
    return {"final_report": report}

# BUILD PARALLEL GRAPH
graph = StateGraph(ResearchState)

# Add all nodes
graph.add_node("google", search_google)
graph.add_node("bing", search_bing)
graph.add_node("arxiv", search_arxiv)
graph.add_node("wiki", search_wikipedia)
graph.add_node("merge", merge_sources)
graph.add_node("synthesize", synthesize)
graph.add_node("report", final_report)

# FAN-OUT: All 4 search nodes from START (run in PARALLEL)
graph.add_edge(START, "google")
graph.add_edge(START, "bing")
graph.add_edge(START, "arxiv")
graph.add_edge(START, "wiki")

# FAN-IN: All to merge
graph.add_edge("google", "merge")
graph.add_edge("bing", "merge")
graph.add_edge("arxiv", "merge")
graph.add_edge("wiki", "merge")

# Sequential after merge
graph.add_edge("merge", "synthesize")
graph.add_edge("synthesize", "report")
graph.add_edge("report", END)

app = graph.compile()

# Run - all 4 searches execute simultaneously!
result = app.invoke({"topic": "LangGraph parallel workflows"})
print(result["final_report"])
```

### 6.3 Advanced: Dynamic Parallelism (Orchestrator-Workers)

```python
# ============================================
# DYNAMIC PARALLEL: ORCHESTRATOR CREATES WORKERS
# ============================================

class DynamicParallelState(TypedDict):
    goal: str
    tasks: list  # Generated by orchestrator
    task_results: Annotated[dict, operator.or_]  # Merge dicts
    completed_tasks: Annotated[list, operator.add]
    final_output: str

def orchestrator_plan(state: DynamicParallelState):
    """Orchestrator: Break goal into parallel tasks"""
    prompt = f"""
    Break this goal into 3-5 INDEPENDENT parallel tasks:
    Goal: {state['goal']}
    
    Return JSON: {{"tasks": [{{"id": "task1", "description": "..."}}, ...]}}
    """
    plan = llm.invoke(prompt)
    tasks = parse_json(plan)["tasks"]
    return {"tasks": tasks, "task_results": {}, "completed_tasks": []}

def worker_execute(state: DynamicParallelState):
    """Worker: Execute ONE task (called in parallel for each task)"""
    # This node will be invoked once PER task via fan-out
    current_task = state["tasks"][len(state["completed_tasks"])]
    
    result = execute_task(current_task)
    return {
        "task_results": {current_task["id"]: result},
        "completed_tasks": [current_task["id"]]
    }

def orchestrator_aggregate(state: DynamicParallelState):
    """Aggregate all worker results"""
    all_results = state["task_results"]
    synthesis = synthesize_results(all_results)
    return {"final_output": synthesis}

# DYNAMIC GRAPH BUILDING
# (LangGraph 0.2+ supports dynamic fan-out via Send API)

from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

def create_dynamic_graph():
    graph = StateGraph(DynamicParallelState)
    
    graph.add_node("plan", orchestrator_plan)
    graph.add_node("worker", worker_execute)
    graph.add_node("aggregate", orchestrator_aggregate)
    
    graph.add_edge(START, "plan")
    
    # DYNAMIC FAN-OUT: Send to worker for each task
    def assign_workers(state):
        return [
            Send("worker", {"task": task, **state})
            for task in state["tasks"]
        ]
    
    graph.add_conditional_edges("plan", assign_workers)
    
    # Fan-in: all workers → aggregate
    graph.add_edge("worker", "aggregate")
    graph.add_edge("aggregate", END)
    
    return graph.compile()
```

---

## 7. Reducers Deep Dive: Merging Parallel Results

### 7.1 Built-in Reducers for Parallel Workflows

```python
# ============================================
# REDUCER PATTERNS FOR PARALLEL MERGING
# ============================================

from typing import TypedDict, Annotated
from langgraph.graph import add_messages
import operator

class ParallelState(TypedDict):
    # 1. LIST CONCATENATION (most common)
    search_results: Annotated[list, operator.add]
    documents: Annotated[list, operator.add]
    errors: Annotated[list, operator.add]
    
    # 2. DICT MERGE (for keyed results)
    task_results: Annotated[dict, operator.or_]
    metrics: Annotated[dict, operator.or_]
    
    # 3. MESSAGE MERGING (for chat)
    messages: Annotated[list, add_messages]
    
    # 4. NUMERIC AGGREGATION (custom reducers)
    total_cost: Annotated[float, lambda a, b: a + b]
    max_score: Annotated[float, max]
    min_latency: Annotated[float, min]
    avg_quality: Annotated[float, lambda a, b: (a + b) / 2]  # Simplified

# CUSTOM REDUCERS FOR COMPLEX MERGING
def merge_unique_by_id(left: list, right: list) -> list:
    """Merge lists, deduplicate by 'id' field"""
    seen = {}
    for item in left + right:
        key = item.get("id") if isinstance(item, dict) else item
        if key not in seen:
            seen[key] = item
    return list(seen.values())

def merge_sets(left: set, right: set) -> set:
    """Merge as sets (auto-dedupe)"""
    return left.union(right)

class AdvancedParallelState(TypedDict):
    unique_docs: Annotated[list, merge_unique_by_id]
    all_keywords: Annotated[set, merge_sets]
```

### 7.2 Reducer Behavior in Supersteps

```
SUPERSTEP MERGE EXAMPLE:
─────────────────────────────────────────────────────────────────────────────

PARALLEL NODES (same superstep):
──────────────────────────────
Node A (Google):   returns {"search_results": ["g1", "g2"], "total_cost": 0.05}
Node B (Bing):     returns {"search_results": ["b1", "b2"], "total_cost": 0.03}
Node C (ArXiv):    returns {"search_results": ["a1"], "total_cost": 0.02}

REDUCERS APPLY:
──────────────────────────────
search_results: operator.add     → ["g1", "g2", "b1", "b2", "a1"]
total_cost:     lambda a,b: a+b  → 0.10

MERGED STATE passed to next node:
──────────────────────────────
{
    "search_results": ["g1", "g2", "b1", "b2", "a1"],
    "total_cost": 0.10
}
```

---

## 8. Combining Sequential + Parallel (Real-World Patterns)

### 8.1 Hybrid: Sequential Pipeline with Parallel Stages

```python
# ============================================
# HYBRID: SEQUENTIAL STAGES WITH PARALLEL EXECUTION
# ============================================

class HybridState(TypedDict):
    topic: str
    
    # Stage 1: Sequential (Planning)
    research_plan: list
    
    # Stage 2: Parallel (Multi-source search)
    google_results: Annotated[list, operator.add]
    bing_results: Annotated[list, operator.add]
    arxiv_results: Annotated[list, operator.add]
    all_sources: list
    
    # Stage 3: Sequential (Processing)
    summaries: Annotated[list, operator.add]
    combined: str
    
    # Stage 4: Parallel (Multi-format output)
    blog_post: str
    twitter_thread: str
    linkedin_post: str
    executive_summary: str
    final_deliverables: dict

# GRAPH STRUCTURE:
# ─────────────────────────────────────────────
# START
#   │
#   ▼
# PLAN (sequential)
#   │
#   ▼
# [GOOGLE, BING, ARXIV] ← PARALLEL (Superstep 1)
#   │     │      │
#   ▼     ▼      ▼
# MERGE (sequential)
#   │
#   ▼
# SUMMARIZE (sequential)
#   │
#   ▼
# [BLOG, TWITTER, LINKEDIN, SUMMARY] ← PARALLEL (Superstep 2)
#   │     │      │         │
#   ▼     ▼      ▼         ▼
# FINAL MERGE (sequential)
#   │
#   ▼
# END

def build_hybrid_graph():
    graph = StateGraph(HybridState)
    
    # Stage 1: Sequential planning
    graph.add_node("plan", create_research_plan)
    
    # Stage 2: Parallel search
    graph.add_node("google", search_google)
    graph.add_node("bing", search_bing)
    graph.add_node("arxiv", search_arxiv)
    
    # Stage 3: Sequential merge & summarize
    graph.add_node("merge", merge_sources)
    graph.add_node("summarize", summarize_all)
    
    # Stage 4: Parallel output generation
    graph.add_node("blog", generate_blog)
    graph.add_node("twitter", generate_twitter)
    graph.add_node("linkedin", generate_linkedin)
    graph.add_node("summary", generate_exec_summary)
    
    # Stage 5: Final merge
    graph.add_node("finalize", compile_deliverables)
    
    # EDGES
    graph.add_edge(START, "plan")
    
    # Fan-out to parallel search
    graph.add_edge("plan", "google")
    graph.add_edge("plan", "bing")
    graph.add_edge("plan", "arxiv")
    
    # Fan-in to merge
    graph.add_edge("google", "merge")
    graph.add_edge("bing", "merge")
    graph.add_edge("arxiv", "merge")
    
    # Sequential
    graph.add_edge("merge", "summarize")
    
    # Fan-out to parallel outputs
    graph.add_edge("summarize", "blog")
    graph.add_edge("summarize", "twitter")
    graph.add_edge("summarize", "linkedin")
    graph.add_edge("summarize", "summary")
    
    # Fan-in to finalize
    graph.add_edge("blog", "finalize")
    graph.add_edge("twitter", "finalize")
    graph.add_edge("linkedin", "finalize")
    graph.add_edge("summary", "finalize")
    
    graph.add_edge("finalize", END)
    
    return graph.compile()
```

### 8.2 Map-Reduce Pattern (Parallel Processing of Collections)

```python
# ============================================
# MAP-REDUCE: Process Each Item in Parallel
# ============================================

class MapReduceState(TypedDict):
    items: list  # Input collection
    item_results: Annotated[dict, operator.or_]  # {item_id: result}
    reduced_result: str

def mapper(state: MapReduceState):
    """Generate Send for EACH item (dynamic fan-out)"""
    from langgraph.types import Send
    return [
        Send("process_item", {"item": item, "item_id": f"item_{i}"})
        for i, item in enumerate(state["items"])
    ]

def process_item(state):
    """Process single item (runs in parallel for each)"""
    item = state["item"]
    item_id = state["item_id"]
    result = process_single_item(item)
    return {"item_results": {item_id: result}}

def reducer(state: MapReduceState):
    """Reduce all item results"""
    all_results = state["item_results"]
    combined = combine_results(all_results)
    return {"reduced_result": combined}

# Graph with dynamic fan-out
graph = StateGraph(MapReduceState)
graph.add_node("process_item", process_item)
graph.add_node("reduce", reducer)

graph.add_conditional_edges(START, mapper)  # Dynamic fan-out
graph.add_edge("process_item", "reduce")     # Fan-in
graph.add_edge("reduce", END)

# Usage
result = app.invoke({
    "items": ["doc1.pdf", "doc2.pdf", "doc3.pdf", "doc4.pdf"]
})
# All 4 documents processed in PARALLEL
```

---

## 9. Performance Optimization for Parallel Workflows

### 9.1 Controlling Parallelism

```python
# ============================================
# PARALLELISM CONTROL
# ============================================

# 1. LIMIT CONCURRENT NODES (via semaphore pattern)
import asyncio

class ConcurrencyControlledState(TypedDict):
    tasks: list
    results: Annotated[dict, operator.or_]
    semaphore_limit: int

async def controlled_worker(state):
    semaphore = asyncio.Semaphore(state.get("semaphore_limit", 3))
    
    async def run_with_limit(task):
        async with semaphore:
            return await process_task(task)
    
    # Process in batches of semaphore_limit
    results = {}
    for task in state["tasks"]:
        result = await run_with_limit(task)
        results[task["id"]] = result
    
    return {"results": results}

# 2. CONFIGURE RECURSION LIMIT (prevents infinite parallel loops)
config = {
    "configurable": {
        "thread_id": "parallel-task",
        "recursion_limit": 100  # Default 25, increase for many parallel tasks
    }
}

# 3. TIMEOUT PROTECTION
from langgraph.graph import RetryPolicy

graph.add_node(
    "slow_task",
    slow_node,
    retry=RetryPolicy(
        max_attempts=2,
        backoff_factor=1.5,
        retry_on=(TimeoutError,)
    )
)
```

### 9.2 Monitoring Parallel Execution

```python
# ============================================
# PARALLEL EXECUTION MONITORING
# ============================================

# Stream with "updates" mode to see parallel completion
async for update in app.astream(input, config, stream_mode="updates"):
    for node_name, output in update.items():
        print(f"✅ {node_name} completed at {datetime.now()}")
        # Track timing per parallel branch

# LangSmith automatically shows:
# - Parallel execution timeline
# - Duration of each branch
# - Superstep boundaries
# - Resource utilization
```

---

## 10. Common Pitfalls & Best Practices

### 10.1 Sequential Pitfalls

| Pitfall | Solution |
|---------|----------|
| **State grows unbounded** | Use `operator.add` only for needed fields; clean up periodically |
| **No error recovery** | Add `RetryPolicy` + conditional retry edges |
| **Can't resume after crash** | Always use `checkpointer` with `thread_id` |
| **Blocking on slow step** | Consider if step can be parallelized |
| **Debugging hard** | Use `stream_mode="updates"` + LangSmith |

### 10.2 Parallel Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Race conditions in reducers** | Use proper reducers (`operator.add`, `operator.or_`, custom) |
| **Shared mutable state** | Never mutate state directly; return new values |
| **Too many parallel nodes** | Limit with semaphore pattern; batch large collections |
| **Uneven task durations** | Use `asyncio.gather` with timeout; consider dynamic scheduling |
| **Merge logic errors** | Test reducers independently; log merged state |

### 10.3 Best Practices Summary

```python
# ============================================
# BEST PRACTICES CHECKLIST
# ============================================

# SEQUENTIAL:
✅ Use Annotated fields with reducers for accumulation
✅ Add RetryPolicy to unreliable nodes
✅ Use checkpointer for persistence
✅ Stream with "updates" for progress visibility
✅ Keep nodes single-purpose

# PARALLEL:
✅ Ensure tasks are truly independent
✅ Use proper reducers for merge (operator.add, operator.or_)
✅ Test reducer logic in isolation
✅ Limit concurrency for resource-heavy tasks
✅ Monitor superstep timing in LangSmith

# HYBRID:
✅ Identify sequential dependencies vs parallel opportunities
✅ Use fan-out/fan-in deliberately
✅ Minimize sequential bottlenecks
✅ Profile to find parallelizable sections

# GENERAL:
✅ State = minimal, serializable, typed
✅ Nodes = pure functions (state → partial update)
✅ Edges = explicit, tested routing logic
✅ Compile once, invoke many
```

---

## 11. Complete Examples Repository

### 11.1 Sequential Examples

| Example | File | Description |
|---------|------|-------------|
| **Basic Sequential** | `sequential/basic.py` | 3-step linear pipeline |
| **Document Processing** | `sequential/doc_pipeline.py` | PDF → Chunk → Summarize → Report |
| **Code Generation** | `sequential/code_gen.py` | Plan → Write → Lint → Test → Fix |
| **Data ETL** | `sequential/etl.py` | Extract → Transform → Validate → Load |
| **With Error Handling** | `sequential/robust.py` | Retry, fallback, checkpoint resume |

### 11.2 Parallel Examples

| Example | File | Description |
|---------|------|-------------|
| **Basic Parallel** | `parallel/basic.py` | 4 independent calculations |
| **Multi-Search** | `parallel/multi_search.py` | Google + Bing + ArXiv + Wiki |
| **Orchestrator-Workers** | `parallel/orchestrator.py` | Dynamic task planning |
| **Map-Reduce** | `parallel/map_reduce.py` | Process collection in parallel |
| **Ensemble Voting** | `parallel/ensemble.py` | Multiple models, vote on result |

### 11.3 Hybrid Examples

| Example | File | Description |
|---------|------|-------------|
| **Research Agent** | `hybrid/research_agent.py` | Plan → Parallel Search → Sequential Synthesize → Parallel Output |
| **Content Pipeline** | `hybrid/content_pipeline.py` | Research → Write → Multi-format Publish |
| **Code Review** | `hybrid/code_review.py` | Parallel Analyzers → Sequential Aggregation |

---

## 12. Quick Reference: Sequential vs Parallel Cheat Sheet

```python
# ============================================
# QUICK REFERENCE: SEQUENTIAL vs PARALLEL
# ============================================

# SEQUENTIAL PATTERN
# ──────────────────
graph.add_edge("A", "B")
graph.add_edge("B", "C")
# A → B → C (one at a time)

# PARALLEL PATTERN (Fan-out/Fan-in)
# ──────────────────────────────────
graph.add_edge(START, "A")
graph.add_edge(START, "B")
graph.add_edge(START, "C")          # Fan-out
graph.add_edge("A", "MERGE")
graph.add_edge("B", "MERGE")
graph.add_edge("C", "MERGE")        # Fan-in

# STATE WITH REDUCERS (for parallel)
# ──────────────────────────────────
class State(TypedDict):
    results: Annotated[list, operator.add]    # List concat
    metrics: Annotated[dict, operator.or_]    # Dict merge
    messages: Annotated[list, add_messages]   # Smart merge

# CONDITIONAL SEQUENTIAL
# ──────────────────────
graph.add_conditional_edges("A", router, {"path1": "B", "path2": "C"})

# DYNAMIC PARALLEL (Map-Reduce)
# ─────────────────────────────
from langgraph.types import Send
def mapper(state): return [Send("worker", item) for item in state["items"]]
graph.add_conditional_edges(START, mapper)
graph.add_edge("worker", "reduce")

# EXECUTION MODES
# ───────────────
app.invoke(input, config)                           # Sync
await app.ainvoke(input, config)                    # Async
async for chunk in app.astream(..., stream_mode="values")    # State stream
async for chunk in app.astream(..., stream_mode="updates")   # Node events
async for chunk in app.astream(..., stream_mode="messages")  # LLM tokens

# PERSISTENCE
# ───────────
checkpointer = SqliteSaver("sqlite:///checkpoints.db")
app = graph.compile(checkpointer=checkpointer)
app.invoke(input, config={"thread_id": "session-1"})
# Resume: same thread_id
```

---

## 13. Key Takeaways

### Sequential Workflows
| Concept | Essence |
|---------|---------|
| **Definition** | Linear A→B→C, each step depends on previous |
| **State** | Accumulates through pipeline (use `operator.add` reducers) |
| **Error Handling** | `RetryPolicy` + conditional retry edges |
| **Persistence** | Checkpointer enables resume from any step |
| **Streaming** | `stream_mode="updates"` shows each step completion |

### Parallel Workflows
| Concept | Essence |
|---------|---------|
| **Definition** | Fan-out (START→A,B,C) → Fan-in (A,B,C→MERGE) |
| **Supersteps** | Parallel nodes run simultaneously in one superstep |
| **Reducers** | Critical for merging: `operator.add` (lists), `operator.or_` (dicts) |
| **Dynamic** | `Send` API for map-reduce patterns |
| **Control** | Semaphore pattern for concurrency limits |

### Hybrid (Real-World)
| Concept | Essence |
|---------|---------|
| **Pattern** | Sequential stages with parallel execution within stages |
| **Design** | Identify dependencies → Sequential; Independent → Parallel |
| **Optimization** | Profile → Find bottlenecks → Parallelize independent sections |

---

## 14. Resources & References

### Videos
- **Playlist:** https://www.youtube.com/watch?v=yC36gN-rqjo&list=PLKnIA16_RmvYsvB8qkUQuJmJNuiCUJFPL

### Official Documentation
- **Graph API:** https://docs.langchain.com/oss/python/langgraph/graph-api
- **State & Reducers:** https://docs.langchain.com/oss/python/langgraph/concepts/state
- **Persistence:** https://docs.langchain.com/oss/python/langgraph/persistence
- **Streaming:** https://docs.langchain.com/oss/python/langgraph/streaming
- **GitHub Repo:** https://github.com/campusx-official/langgraph-tutorials
- **Send API:** https://docs.langchain.com/oss/python/langgraph/concepts/send
