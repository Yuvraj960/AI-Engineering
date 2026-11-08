# Context Engineering — Complete Technical Guide

## 1. What Is Context Engineering?

### 1.1 Definition

> **Context Engineering** is the discipline of curating and maintaining the optimal set of tokens (information) during LLM inference, including all the other information that may land there outside of the prompts.

> **Karpathy's Analogy:** LLMs are a new kind of operating system. The LLM is the CPU; its context window is the RAM. Context engineering is the OS's memory management — deciding what gets loaded into RAM at each step.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CONTEXT ENGINEERING vs PROMPT ENGINEERING            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROMPT ENGINEERING                    │  CONTEXT ENGINEERING               │
│  ─────────────────────────             │  ────────────────────────          │
│  Scope: Single instruction string      │  Scope: Full token set at inference│
│  Surface: System prompt + user message │  Surface: Instructions + RAG +     │
│                                        │    Memory + Tools + History        │
│  State: Stateless/single-turn          │  Stateful, multi-turn, hours       │
│  Optimization: Better phrasing         │  Optimization: Signal-to-noise     │
│  Failure: Misunderstands task          │  Failure: Too much/little/wrong    │
│  Owner: Prompt writers                 │  Owner: Platform/agent engineers   │
│                                        │                                    │
│  "How to phrase the question"          │  "How to wire the pipeline"        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Context Engineering Matters Now

| Factor | Impact |
|--------|--------|
| **Agents run in loops** | Accumulate tool results, reasoning, history across turns |
| **Context rot** | Larger context → lower recall accuracy (Chroma research) |
| **Cost & latency** | Quadratic scaling (dense attention); larger windows = more $ and slower |
| **Failure modes** | Poisoning, distraction, confusion, clash (see §3) |
| **Production reality** | Agents run for hours, hundreds of turns — not one-shot prompts |

> **Key Insight:** "If you're swapping nouns and adjectives, you're doing prompt engineering. If you're changing what data the agent retrieves, in what order, with what re-ranking, and what gets evicted — you're doing context engineering."

---

## 2. The Context Stack: 7 Components (Google Cloud / Smitha Kolan)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        THE 7-COMPONENT CONTEXT STACK                       │
├──────────┬──────────────────────┬──────────────────────────────────────────┤
│ Layer    │ Component            │ Description                              │
├──────────┼──────────────────────┼──────────────────────────────────────────┤
│ 1        │ System Instructions  │ Role, constraints, output format, rules  │
│ 2        │ Few-Shot Examples    │ Demonstrations of desired behavior       │
│ 3        │ User Input/Query     │ Current task or question                 │
│ 4        │ Conversation History │ Prior turns (user + assistant + tools)   │
│ 5        │ Retrieved Facts      │ RAG chunks, search results, facts        │
│ 6        │ Tool Definitions     │ Function schemas + descriptions          │
│ 7        │ Tool Results         │ Outputs from previous tool calls         │
└──────────┴──────────────────────┴──────────────────────────────────────────┘
```

**Anthropic's 4 Pillars (Alternative Taxonomy):**

| Pillar | Contents | Key Challenge |
|--------|----------|---------------|
| **Instructions** | System prompt, behavioral framing | Mismatch with model's training |
| **Retrieval** | RAG, structured queries, file reads, JIT retrieval | Bad retrieval → hallucination |
| **Memory** | Short-term (history), long-term (preferences, summaries) | Compaction vs. loss |
| **Tools** | Function definitions, schemas, results | Bloated tool sets → confusion |

---

## 3. The 4 Failure Modes (Google Cloud / Smitha Kolan)

| # | Failure Mode | Description | Root Cause | Mitigation |
|---|--------------|-------------|------------|------------|
| **1** | **Poisoning** | A hallucination enters context and corrupts future turns | Bad output accepted as fact | Output validation, fact-checking rails |
| **2** | **Distraction** | Model fixates on huge history instead of fresh plan | Excessive history, no compaction | Summarization, sliding window |
| **3** | **Confusion** | Extra unrelated details nudge model to wrong answer | Low signal-to-noise ratio | Selective retrieval, re-ranking |
| **4** | **Clash** | Two sources disagree; model picks wrong one | Contradictory sources in context | Source prioritization, conflict resolution |

> **Root Cause:** All four stem from **poor signal-to-noise ratio** in the context window.

---

## 4. The 4 Steps of Context Engineering (Google Cloud / Smitha Kolan)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE 4-STEP CONTEXT ENGINEERING LOOP                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   STEP 1: WRITE          STEP 2: SELECT         STEP 3: COMPRESS            │
│   ─────────────          ─────────────          ──────────────              │
│   • System instructions  • Retrieve relevant    • Summarize history         │
│   • Few-shot examples    • Filter by relevance  • Compact old turns         │
│   • Tool definitions     • Rank by importance   • Drop low-signal chunks    │
│   • Output schemas       • Budget tokens        • Cap tool outputs          │
│                                                                             │
│                           STEP 4: ISOLATE                                   │
│                           ──────────────                                    │
│                           • Subagents for sub-tasks                         │
│                           • Isolate tool results                            │
│                           • Separate contexts per concern                   │
│                                                                             │
│   "CURATE, DON'T DUMP" — The golden rule                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Step 1: Write — Curate the Input

| Component | Best Practice |
|-----------|---------------|
| **System Instructions** | Specific but not brittle; align with model's training |
| **Few-Shot Examples** | Diverse, representative, correct format |
| **Tool Definitions** | Clear names, unambiguous params, minimal set |
| **Output Schemas** | Pydantic/JSON Schema for structured output |

```python
# Example: Well-engineered system prompt
SYSTEM_PROMPT = """You are a code review agent. Your role:
1. Analyze code for security, performance, style
2. Use tools: read_file, grep, run_tests
3. Output: JSON with findings {file, line, severity, message, fix}
4. Never suggest changes without evidence
5. If unsure, say "I need more context" — never guess."""
```

### 4.2 Step 2: Select — Filter & Rank

```python
# Retrieval with re-ranking (critical!)
def retrieve_context(query, k=5, fetch_k=20):
    # 1. Fetch broad candidates (high recall)
    candidates = vector_store.similarity_search(query, k=fetch_k)
    
    # 2. Re-rank with cross-encoder (high precision)
    reranked = cross_encoder.rerank(query, candidates, top_k=k)
    
    # 3. Filter by relevance threshold
    filtered = [c for c in reranked if c.score > 0.7]
    
    return filtered
```

**Token Budget Allocation:**

| Component | Budget | Strategy |
|-----------|--------|----------|
| System prompt | 5-10% | Fixed |
| Few-shot | 10-15% | Fixed, curate |
| Tool definitions | 5-10% | Minimize, deduplicate |
| Retrieval | 40-50% | Re-rank, threshold |
| History | 20-30% | Compact, sliding window |
| Tool results | 10-15% | Clear old, truncate |

### 4.3 Step 3: Compress — Manage Budget

```python
# Compaction strategies (Anthropic Cookbook)
def compact_conversation(messages, budget_ratio=0.5):
    """Summarize older turns, keep recent turns intact."""
    if token_count(messages) < MAX_TOKENS * budget_ratio:
        return messages  # No compaction needed
    
    # Keep last N turns intact
    recent = messages[-6:]  # Last 3 exchanges
    older = messages[:-6]
    
    # Summarize older turns
    summary = llm.invoke(f"""
    Summarize this conversation preserving:
    - Key decisions made
    - Facts discovered
    - Unresolved questions
    - Current goal
    
    Conversation: {older}
    """)
    
    # Return: summary + recent messages
    return [SystemMessage(content=f"Previous context: {summary}")] + recent
```

### 4.4 Step 4: Isolate — Subagents & Tool Clearing

```python
# Tool-result clearing (Anthropic Cookbook)
def clear_tool_results(messages, max_age=10):
    """Replace old tool results with placeholders."""
    for msg in messages[:-max_age]:
        if isinstance(msg, ToolMessage):
            msg.content = f"[Tool result cleared — re-call {msg.tool_call_id} if needed]"
    return messages

# Subagent isolation (LangGraph pattern)
def create_subagent(task, context_budget=4000):
    """Run sub-task in isolated context."""
    subagent = create_react_agent(
        llm, tools, 
        checkpointer=InMemorySaver(),
        config={"configurable": {"thread_id": f"subagent-{task.id}"}}
    )
    return subagent
```

---

## 5. Anthropic Cookbook: 3 Primitives (Memory, Compaction, Tool Clearing)

### 5.1 Compaction — Whole-Transcript Summarization

```python
# Anthropic API: Server-side compaction
from anthropic import Anthropic

client = Anthropic()

# Automatic compaction (configured on client)
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    messages=messages,
    system=system_prompt,
    # Compaction triggers automatically near limit
    compaction={
        "type": "server_side",
        "summary_prompt": """Preserve:
        - Key decisions & rationale
        - Facts discovered
        - Unresolved questions
        - Current goal & plan
        - Tool usage patterns"""
    }
)

# Custom compaction prompt (critical!)
CUSTOM_COMPACTION_PROMPT = """Summarize this conversation for a research agent.
Preserve:
- All specific facts, numbers, citations from source documents
- Hypotheses formed and tested
- Methodology decisions & rationale
- Unresolved questions & next steps
- Tool usage patterns that worked/failed
Discard:
- Greetings, meta-commentary
- Redundant confirmations
- Failed tool calls that were retried successfully"""
```

### 5.2 Tool-Result Clearing — Sub-Transcript Surgery

```python
# Clear old tool results, keep tool_use records
def clear_tool_results(messages, keep_last_n=5):
    """Replace tool_result content with placeholders."""
    cleared = []
    tool_results_seen = 0
    
    for msg in reversed(messages):
        if msg.type == "tool_result":
            tool_results_seen += 1
            if tool_results_seen > keep_last_n:
                # Replace content, keep tool_use_id
                cleared.append({
                    "type": "tool_result",
                    "tool_use_id": msg.tool_use_id,
                    "content": f"[Cleared — re-call tool if needed]",
                    "is_error": False
                })
                continue
        cleared.append(msg)
    
    return list(reversed(cleared))

# Key insight: "Once a tool has been called deep in history, the agent rarely 
# needs to see the raw result again. Clearing is one of the safest, lightest-touch 
# ways to recover space."
```

### 5.3 Memory — Structured Note-Taking

```python
# File-backed memory (Anthropic pattern)
import json
from pathlib import Path

class FileMemory:
    def __init__(self, memory_dir="/memories"):
        self.dir = Path(memory_dir)
        self.dir.mkdir(exist_ok=True)
    
    def write(self, key, content):
        (self.dir / key).write_text(content)
    
    def read(self, key):
        path = self.dir / key
        return path.read_text() if path.exists() else None
    
    def list(self):
        return list(self.dir.glob("*.md"))

# Agent-driven memory (model writes its own notes)
MEMORY_TOOL = {
    "name": "write_memory",
    "description": "Save findings to persistent memory",
    "parameters": {
        "type": "object",
        "properties": {
            "key": {"type": "string"},
            "content": {"type": "string"}
        },
        "required": ["key", "content"]
    }
}

# Session 2 reads from /memories instead of re-reading 40K-token docs
```

**Memory Effectiveness Depends on Quality:**
> "Memory's value depends on the agent's judgment about what to write. If Session 1 saved sparse notes, Session 2 falls back to re-reading source documents."

---

## 6. Context Engineering for Coding Agents (Sourcegraph / Anthropic)

### 6.1 Why Coding Agents Need Special Treatment

| Challenge | Context Engineering Solution |
|-----------|-------------------------------|
| **Million-line repos** | Code-graph queries (MCP), not grep |
| **Cross-repo dependencies** | Multi-repo indexing, dependency graphs |
| **Symbol resolution** | LSP-backed "go to definition" tools |
| **Test feedback loops** | Tool-result clearing (test output = bulky) |
| **Multi-file edits** | Subagent isolation per file |

### 6.2 Code-Aware Retrieval (Sourcegraph MCP)

```python
# Sourcegraph MCP Server provides:
# - Code graph queries (callers, references, definitions)
# - Precise symbol search (not keyword grep)
# - Cross-repo dependency resolution
# - Deterministic, structured context

# Example MCP tool definition
CODE_GRAPH_TOOL = {
    "name": "code_search",
    "description": "Search code using semantic graph, not keywords",
    "parameters": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Symbol or concept"},
            "repo": {"type": "string", "description": "Repository name"},
            "depth": {"type": "int", "default": 3}
        }
    }
}
```

### 6.3 Coding Agent Context Pipeline

```python
def coding_agent_pipeline(task, repo):
    # 1. Understand task → generate search queries
    queries = llm.generate_queries(task)
    
    # 2. Parallel code graph retrieval
    candidates = parallel_retrieve(queries, repo)
    
    # 3. Re-rank by relevance to task
    ranked = cross_encoder.rerank(task, candidates, top_k=20)
    
    # 4. Select top-k within budget
    selected = select_within_budget(ranked, budget=15000)
    
    # 5. Build context: instructions + code + tests
    context = assemble_context(
        instructions=CODING_SYSTEM_PROMPT,
        code=selected,
        tests=find_related_tests(selected),
        history=compacted_history
    )
    
    return agent.invoke(context)
```

---

## 7. Token Budget Management Strategies

### 6.1 Budget Allocation Framework

```python
class TokenBudget:
    def __init__(self, max_tokens=200000):
        self.max = max_tokens
        self.reserved = {
            "system": 0.05,
            "few_shot": 0.10,
            "tools": 0.08,
            "retrieval": 0.45,
            "history": 0.25,
            "tool_results": 0.07
        }
    
    def allocate(self, component, tokens):
        budget = int(self.max * self.reserved.get(component, 0))
        return min(tokens, budget)

# Dynamic rebalancing based on task phase
def rebalance_budget(phase, budget):
    if phase == "exploration":
        return {"retrieval": 0.60, "history": 0.15, "tools": 0.10}
    elif phase == "implementation":
        return {"retrieval": 0.30, "history": 0.40, "tools": 0.15}
    elif phase == "verification":
        return {"retrieval": 0.20, "history": 0.30, "tools": 0.35}
```

### 6.2 Context Rot Mitigation

```python
# Context rot: model's recall accuracy drops as context grows
# Mitigations (from Chroma research + Anthropic):

def mitigate_context_rot(messages):
    # 1. Keep critical info at edges (primacy/recency effect)
    critical = extract_critical_facts(messages)
    
    # 2. Reorder: critical → recent → supporting
    reordered = prioritize_messages(messages, critical)
    
    # 3. Chunk with overlap (like RAG)
    chunks = chunk_with_overlap(messages, size=4000, overlap=500)
    
    # 4. Insert periodic summaries
    summarized = inject_summaries(chunks, interval=5)
    
    return summarized
```

---

## 8. Evaluation & Testing

### 8.1 Context Engineering Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Token Efficiency** | Tokens per correct answer | Minimize |
| **Recall Accuracy** | Needle-in-haystack recall | > 95% at 50% window |
| **Latency** | Time to first token | < 2s p99 |
| **Cost per Task** | $ per completed task | Minimize |
| **Hallucination Rate** | Unsupported claims | < 1% |
| **Tool Efficiency** | Tool calls per task | Minimize |

### 8.2 Needle-in-Haystack Evaluation

```python
def needle_in_haystack_test(agent, context_sizes=[10k, 50k, 100k, 200k]):
    """Test recall at different context sizes."""
    results = {}
    for size in context_sizes:
        # Embed needle at random position
        haystack = generate_haystack(size)
        needle = "The secret code is ALPHA-7-OMEGA."
        position = random.randint(0, len(haystack))
        haystack = haystack[:position] + needle + haystack[position:]
        
        # Ask agent
        answer = agent.invoke(f"Context: {haystack}\nQuestion: What is the secret code?")
        
        results[size] = "ALPHA-7-OMEGA" in answer
    
    return results
```

### 8.3 Regression Testing for Context Changes

```python
# Regression suite for context engineering changes
CONTEXT_REGRESSION_SUITE = [
    {"name": "long_conversation", "turns": 50, "max_tokens": 100000},
    {"name": "multi_file_code_edit", "files": 10, "edits": 5},
    {"name": "rag_multi_hop", "hops": 3, "docs_per_hop": 5},
    {"name": "tool_heavy_workflow", "tool_calls": 20},
    {"name": "session_resumption", "gap_turns": 100},
]

def run_regression(agent):
    results = {}
    for test in CONTEXT_REGRESSION_SUITE:
        start = time.time()
        result = run_test(agent, test)
        results[test["name"]] = {
            "passed": result.success,
            "tokens": result.tokens_used,
            "latency": time.time() - start,
            "quality": result.quality_score
        }
    return results
```

---

## 9. Production Patterns & Patterns Library

### 9.1 Common Patterns

| Pattern | When to Use | Implementation |
|---------|-------------|----------------|
| **Sliding Window** | Long conversations | Keep last N turns + summary |
| **Hierarchical Summary** | Very long tasks | Tree of summaries (turn → section → chapter) |
| **Just-in-Time Retrieval** | Large corpora | Fetch on demand, not upfront |
| **Subagent Isolation** | Independent sub-tasks | Separate context per sub-task |
| **Tool Result Clearing** | Bulky tool outputs | Clear old, keep tool_use record |
| **Structured Note-Taking** | Cross-session memory | Agent writes/reads /memories |

### 9.2 Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| **Dump everything** | Context rot, high cost | Select, rank, compress |
| **No memory** | Re-reading docs every session | Structured note-taking |
| **Bloated tool set** | Confusion, wrong tool choice | Minimize, deduplicate |
| **No compaction** | Context rot, cost explosion | Auto-compaction + custom prompt |
| **No tool clearing** | Tool results dominate context | Clear old, keep tool_use record |

---

## 10. Decision Framework

```
NEED CONTEXT ENGINEERING?
│
├─► Single-turn, stateless?
│   └─► Prompt engineering sufficient
│
├─► Multi-turn conversation?
│   ├─► Short (< 10 turns)?
│   │   └─► Sliding window + basic compaction
│   │
│   └─► Long-running agent?
│       ├─► Dominated by tool results?
│       │   └─► Tool-result clearing + subagents
│       │
│       ├─► Dominated by conversation history?
│       │   └─► Compaction + sliding window
│       │
│       ├─► Cross-session persistence needed?
│       │   └─► Structured memory (/memories)
│       │
│       └─► All of the above?
│           └─► All 3 primitives + subagents
│
└─► Coding agent?
    └─► Code-graph retrieval (MCP) + subagent isolation + test feedback loops
```

---

## 10. Production Checklist

### Pre-Launch

- [ ] **Context budget** defined with allocations
- [ ] **Retrieval pipeline** with re-ranking
- [ ] **Compaction** configured with custom prompt
- [ ] **Tool-result clearing** enabled
- [ ] **Memory system** implemented (/memories)
- [ ] **Subagent isolation** for independent tasks
- [ ] **Token monitoring** (dashboards, alerts)
- [ ] **Needle-in-haystack** test passed at max budget
- [ ] **Regression suite** for context changes
- [ ] **Cost/latency** within targets

### Ongoing

- [ ] **Daily** token usage review
- [ ] **Weekly** compaction quality review
- [ ] **Monthly** retrieval quality audit
- [ ] **Quarterly** context architecture review

---

## 11. Decision Flowchart

```
NEED CONTEXT ENGINEERING?
│
├─► Single-turn, stateless?
│   └─► Prompt engineering sufficient
│
├─► Multi-turn, < 10 turns?
│   └─► Sliding window + basic compaction
│
├─► Long-running agent?
│   │
│   ├─► Context dominated by tool results?
│   │   └─► Tool-result clearing + subagent isolation
│   │
│   ├─► Context dominated by history?
│   │   └─► Compaction + sliding window
│   │
│   ├─► Need cross-session persistence?
│   │   └─► Structured memory (/memories)
│   │
│   └─► All of the above?
│       └─► All 3 primitives + subagent orchestration
│
└─► Coding agent?
    └─► Code-graph retrieval (MCP) + subagent per file + test feedback loops
```

---

## 11. Quick Reference: Context Engineering Cheat Sheet

```python
# ============================================
# CONTEXT ENGINEERING QUICK START
# ============================================

# 1. TOKEN BUDGET
MAX_TOKENS = 200000
BUDGET = {
    "system": 0.05, "few_shot": 0.10, "tools": 0.08,
    "retrieval": 0.45, "history": 0.25, "tool_results": 0.07
}

# 2. RETRIEVAL WITH RE-RANKING
def retrieve(query, k=5, fetch_k=20):
    candidates = vector_search(query, k=fetch_k)
    reranked = cross_encoder.rerank(query, candidates, top_k=k)
    return [c for c in reranked if c.score > 0.7]

# 3. COMPACTION
def compact(messages, keep_recent=6):
    if count_tokens(messages) < MAX_TOKENS * 0.7:
        return messages
    recent = messages[-keep_recent:]
    older = messages[:-keep_recent]
    summary = summarize(older)
    return [SystemMessage(f"Summary: {summary}")] + recent

# 4. TOOL CLEARING
def clear_tool_results(messages, keep_last=5):
    cleared = []
    seen = 0
    for msg in reversed(messages):
        if msg.type == "tool_result":
            seen += 1
            if seen > keep_last:
                msg = replace(msg, content=f"[Cleared — re-call if needed]")
        cleared.append(msg)
    return list(reversed(cleared))

# 5. MEMORY
class Memory:
    def __init__(self, dir="/memories"): self.dir = Path(dir)
    def write(self, key, content): (self.dir/key).write_text(content)
    def read(self, key): return (self.dir/key).read_text() if (self.dir/key).exists() else None

# 5. FULL PIPELINE
def build_context(query, history, memory):
    # Retrieve
    facts = retrieve(query)
    # Compact
    history = compact(history)
    # Clear tools
    history = clear_tool_results(history)
    # Memory
    notes = memory.read("session_notes")
    # Assemble
    return assemble(
        system=SYSTEM_PROMPT,
        few_shot=FEW_SHOT,
        tools=TOOLS,
        retrieval=facts,
        history=history,
        memory=notes,
        query=query
    )
```

---

## 12. Resources & References

### Key Papers & Articles
- **Anthropic:** "Effective Context Engineering for AI Agents" (Sep 2025)
- **Google Cloud:** "Context Engineering Explained" (Smitha Kolan, Jul 2026)
- **Sourcegraph:** "Context Engineering" (Phil Schmid, 2025)
- **LangChain:** "Context Engineering for Agents" (Jul 2025)
- **Anthropic Cookbook:** "Memory, Compaction, Tool Clearing" (Mar 2026)
- **Chroma Research:** "Context Rot" (2024)

### Key Videos
- **Google Cloud Tech:** "Context Engineering Explained" — Smitha Kolan
- **Anthropic:** Multiple cookbook tutorials on YouTube

### Official Documentation
- **Anthropic API:** Compaction, Context Editing, Memory Tool
- **LangGraph:** Context management patterns
- **MCP Spec:** https://modelcontextprotocol.io/
- **A2A Spec:** https://a2a-protocol.org/

### Tools & Frameworks
- **Memory:** mem0, Letta, custom file-backed
- **Retrieval:** Cross-encoders, hybrid search, code-graph (Sourcegraph MCP)
- **Compaction:** Anthropic server-side, custom prompts
- **Evaluation:** Needle-in-haystack, RAGAS, custom regression suites

---

## 13. Summary: Context Engineering in Your Stack

| Layer | Technology | Context Engineering Role |
|-------|------------|-------------------------|
| **Framework** | LangGraph, CrewAI | Orchestration, subagents |
| **Retrieval** | Vector DB + Cross-encoder + MCP | High-precision retrieval |
| **Compaction** | Anthropic server-side / Custom | History management |
| **Tool Clearing** | Native (Anthropic) / Custom | Tool result hygiene |
| **Memory** | File-backed / mem0 / Letta | Cross-session persistence |
| **Token Budget** | Dynamic allocation | Cost/latency control |
| **Observability** | LangSmith / Langfuse | Token tracking, quality |
