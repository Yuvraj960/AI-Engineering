# Token Optimization & Prompt Caching — Complete Technical Guide

## 1. What Is Prompt Caching?

### 1.1 Core Concept

> **Prompt Caching** stores precomputed **Key-Value (KV) pairs** from the model's attention layers, so repeated prompt prefixes don't need to be reprocessed.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PROMPT CACHING vs OUTPUT CACHING                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  OUTPUT CACHING (Traditional)          │  PROMPT CACHING (KV Cache)         │
│  ─────────────────────────────         │  ─────────────────────────         │
│  Input → LLM → Output → Cache          │  Input → KV Cache → LLM → Output   │
│  Next: Same input → Return cached      │  Next: Same prefix → Reuse KV      │
│  ✗ Only helps with EXACT same query    │  ✓ Helps with SAME PREFIX + new    │
│  ✗ Doesn't reduce LLM computation      │  ✓ Reduces PREFILL computation     │
│                                        │                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 How KV Caching Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KV CACHE MECHANICS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INPUT PROMPT: "Summarize this 50-page document: [DOC] ..."                 │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ TRANSFORMER LAYERS (x N layers)                                     │    │
│  │                                                                     │    │
│  │  Layer 1: Token → Q,K,V → Attention → KV Cache₁                     │    │
│  │  Layer 2: Token → Q,K,V → Attention → KV Cache₂                     │    │
│  │    ...                                                              │    │
│  │  Layer N: Token → Q,K,V → Attention → KV Cache_N                    │    │
│  │                                                                     │    │
│  │  💾 KV CACHE = [Cache₁, Cache₂, ..., Cache_N]  ← STORED             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│         │                                                                   │
│         ▼                                                                   │
│  NEXT REQUEST: "Question about same doc: What's the warranty?"              │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ PREFIX MATCH: System + Document cached? → YES                       │    │
│  │                                                                     │    │
│  │  Layer 1: LOAD Cache₁ → Skip prefill                                │    │
│  │  Layer 2: LOAD Cache₂ → Skip prefill                                │    │
│  │    ...                                                              │    │
│  │  Layer N: LOAD Cache_N → Skip prefill                               │    │
│  │                                                                     │    │
│  │  ONLY PROCESS: New question tokens                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 What Can Be Cached

| Content Type | Example | Cache Benefit |
|--------------|---------|---------------|
| **System Prompts** | "You are a helpful customer service agent..." | Every conversation |
| **Large Documents** | 50-page PDF, legal contract, research paper | Multi-question on same doc |
| **Few-Shot Examples** | "Format: Input → Output: {json}" | Consistent formatting |
| **Tool/Function Definitions** | `{"name": "search", "params": {...}}` | Tool-use agents |
| **Conversation History** | Previous turns in multi-turn chat | Long conversations |
| **Tool Results (re-fetchable)** | File contents, API responses | Can re-fetch if needed |

---

## 2. How Prompt Caching Works

### 2.1 Prefix Matching Algorithm

```
CACHE LOOKUP PROCESS:
─────────────────────────────────────────────────────────────────────────────
1. Request arrives with: System + Tools + Messages
2. System scans from START, token-by-token
3. Finds longest matching prefix in cache
4. Match found → Load KV cache up to breakpoint
3. No match → Process entire prompt, cache new prefix
─────────────────────────────────────────────────────────────────────────────

PREFIX MATCHING RULES:
├── Matches from BEGINNING (token 0)
├── Token-by-token exact match required
├── Stops at FIRST differing token
├── Caches: System → Tools → Messages (in order)
└── Breakpoint = last cacheable block
```

### 2.2 Cache Structure

```
CACHED PREFIX STRUCTURE:
─────────────────────────────────────────────────────────────────────────────
[SYSTEM PROMPT] → [TOOL DEFINITIONS] → [MESSAGES 1..N] → [CACHE BREAKPOINT]
                                                    │
                                                    ▼
                                            CACHED KV PAIRS
                                            (All layers, all tokens)
                                            Up to this point
```

---

## 3. Implementation: Automatic vs Explicit Caching

### 3.1 Automatic Caching (Easiest)

```python
# Automatic: System caches up to last cacheable block automatically
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # Enable auto-caching
    system="You are an expert legal analyst...",
    messages=[
        {"role": "user", "content": "Analyze this contract: [50-page contract]"}
    ]
)

# Response includes cache metrics
print(response.usage)
# {
#   "input_tokens": 50,              # Only new tokens
#   "cache_read_input_tokens": 150000,  # Cached prefix tokens
#   "cache_creation_input_tokens": 0,   # No new cache write
#   "output_tokens": 500
# }
```

### 3.2 Explicit Cache Breakpoints (Fine-Grained Control)

```python
# Explicit: Place cache_control on specific blocks
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a legal expert. Analyze contracts.",
            "cache_control": {"type": "ephemeral"}  # Cache system prompt
        }
    ],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "Here is a 50-page contract: [FULL TEXT]",
                    "cache_control": {"type": "ephemeral"}  # Cache document
                },
                {
                    "type": "text",
                    "text": "Summarize the key terms."  # NOT cached (after breakpoint)
                }
            ]
        ]
    )
```

---

## 4. Pricing & Cost Savings

### 4.1 Anthropic Pricing (2026)

| Model | Base Input | 5-min Cache Write | 1-hr Cache Write | Cache Hit/Refresh | Output |
|-------|------------|-------------------|------------------|-------------------|--------|
| **Claude Opus 5.5** | $4/MTok | $5/MTok | $8/MTok | **$0.20/MTok** | $20/MTok |
| **Claude Opus 5** | $5/MTok | $6.25/MTok | $10/MTok | $0.50/MTok | $25/MTok |
| **Claude Sonnet 5.5** | $3/MTok | $3.75/MTok | $6/MTok | $0.15/MTok | $15/MTok |
| **Claude Haiku 5.5** | $0.25/MTok | $0.30/MTok | $0.50/MTok | **$0.01/MTok** | $1.25/MTok |

> **Key Insight:** Cache hits cost **~95% less** than base input tokens!

### 4.2 Cost Calculation Example

```
SCENARIO: 100,000 token document, 100 questions/day
─────────────────────────────────────────────────────────────────────────────

WITHOUT CACHING:
100 requests × (100,000 + 500) tokens × $3/MTok = $30.15/day

WITH CACHING (5-min TTL, assuming cache hits after first):
├── First request: 100,500 tokens × $3/MTok = $0.30 (cache write)
├── 99 cached: 500 tokens × $0.15/MTok = $0.0075/day (cache hits)
├── TOTAL: ~$0.31/day

SAVINGS: **99% cost reduction** ($30 → $0.31/day)
─────────────────────────────────────────────────────────────────────────────
```

---

## 3. Token Optimization Beyond Caching

### 3.1 Token Reduction Techniques

```python
# 1. SYSTEM PROMPT OPTIMIZATION
# ❌ BAD: Verbose, redundant
"Act as a helpful assistant who answers questions politely and accurately..."

# ✅ GOOD: Concise, specific
"Answer technical questions accurately. Cite sources. Be concise."

# 2. FEW-SHOT MINIMIZATION
# ❌ 10 examples
examples = [ex1, ex2, ..., ex10]

# ✅ 2-3 diverse, high-quality examples
examples = [ex1, ex2, ex3]

# 3. TOOL DEFINITION COMPRESSION
# ❌ Verbose schemas
{"name": "search", "description": "Search the web for information...", 
 "parameters": {"type": "object", "properties": {"q": {"type": "string"}}}}

# ✅ Minimal schemas
{"name": "search", "description": "Web search", "parameters": {"q": "string"}}

# 4. OUTPUT SCHEMA (Pydantic vs verbose)
# ❌ "Return a JSON object with fields: title (string), summary (string)..."
# ✅ Pydantic model (compact, validates automatically)
class Response(BaseModel):
    title: str
    summary: str
    confidence: float = Field(ge=0, le=1)
```

### 3.2 Context Compression Strategies

```python
# 1. SLIDING WINDOW + SUMMARY
def compress_history(messages, max_tokens=50000):
    if count_tokens(messages) <= max_tokens:
        return messages
    
    # Keep: system + last 6 messages + summary
    recent = messages[-6:]
    older = messages[:-6]
    summary = summarize(older)  # LLM summarizes
    return [SystemMessage(f"Summary: {summary}")] + recent

# 2. TOOL RESULT CLEARING (Anthropic pattern)
def clear_tool_results(messages, keep_last=5):
    cleared = []
    seen = 0
    for msg in reversed(messages):
        if msg.type == "tool_result":
            if seen >= keep:
                msg = replace(msg, content="[Cleared — re-call if needed]")
            seen += 1
        cleared.append(msg)
    return list(reversed(cleared))

# 3. RETRIEVAL COMPRESSION
def compress_retrieval(chunks, query, budget=15000):
    # 1. Fetch more, re-rank
    candidates = retrieve(query, k=50)
    # 2. Cross-encoder re-rank
    ranked = cross_encoder.rerank(query, candidates, top_k=20)
    # 3. Budget-aware selection
    selected = []
    used = 0
    for chunk in ranked:
        if used + chunk.tokens > budget:
            break
        selected.append(chunk)
        used += chunk.tokens
    return selected
```

---

## 4. Advanced Optimization Patterns

### 4.1 Parallelization (Anthropic: Building Effective Agents)

```python
# SECTIONING: Parallel independent tasks
async def parallel_research(topics):
    tasks = [research(topic) for topic in topics]
    return await asyncio.gather(*tasks)

# VOTING: Multiple perspectives for reliability
async def voted_decision(prompt, n=3):
    prompts = [f"{prompt}\nPerspective {i+1}:" for i in range(n)]
    results = await asyncio.gather(*[llm.ainvoke(p) for p in prompts])
    # Aggregate votes
    return majority_vote(results)

# AGENT PARALLELIZATION (LangGraph)
from langgraph.prebuilt import create_react_agent

researcher = create_react_agent(llm, [search_web])
analyst = create_react_agent(llm, [analyze_data])

# Run in parallel
results = await asyncio.gather(
    researcher.ainvoke({"messages": [HumanMessage("Research topic")]}),
    analyst.ainvoke({"messages": [HumanMessage("Analyze data")]})
)
```

### 4.2 Subagent Isolation (Token Budget)

```python
# Each subagent gets its own context budget
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver

def create_subagent(task, budget=10000):
    return create_react_agent(
        model=llm,
        tools=[search, calculate],
        checkpointer=MemorySaver(),
        config={
            "configurable": {"thread_id": f"sub-{task.id}"},
            "max_tokens": budget
        }
    )

# Main agent delegates to subagents
async def main_agent(task):
    subagent = create_subagent(task, budget=5000)
    result = await subagent.ainvoke({"messages": [HumanMessage(task)]})
    return result  # Returns only final answer, not intermediate context
```

---

## 4. Prompt Structure for Optimal Caching

### 4.1 Golden Rule: Static First, Dynamic Last

```
✅ OPTIMAL STRUCTURE (Cache-friendly):
─────────────────────────────────────────────────────────────────────────────
[SYSTEM PROMPT]           ← STATIC (cached)
[TOOL DEFINITIONS]        ← STATIC (cached)  
[LARGE DOCUMENT]          ← STATIC (cached)
[FEW-SHOT EXAMPLES]       ← STATIC (cached)
[CACHE BREAKPOINT]        ← ────────────────
[USER QUESTION]           ← DYNAMIC (processed fresh)
─────────────────────────────────────────────────────────────────────────────

❌ BAD STRUCTURE (Cache-killing):
─────────────────────────────────────────────────────────────────────────────
[USER QUESTION]           ← DYNAMIC (breaks cache immediately)
[SYSTEM PROMPT]           ← Never cached (after dynamic)
[DOCUMENT]                ← Never cached
[FEW-SHOT]                ← Never cached
```

### 4.2 Multi-Turn Conversation Caching

```python
# Automatic caching handles growing history
# First turn: System + Doc cached
# Turn 2: System + Doc + Turn1 cached
# Turn 3: System + Doc + Turn1 + Turn2 cached
# ...

# Explicit control for long conversations
config = {
    "cache_control": {"type": "ephemeral"},
    # Or 1-hour cache for very long sessions
    # "cache_control": {"type": "ephemeral", "ttl": "1h"}
}

# Monitor cache efficiency
def check_cache_efficiency(response):
    usage = response.usage
    total = usage.cache_read + usage.cache_creation + usage.input_tokens
    hit_rate = usage.cache_read / total if total > 0 else 0
    return hit_rate  # Target > 0.8 for good caching
```

---

## 5. Monitoring & Debugging

### 5.1 Key Metrics (Anthropic Response)

```python
def analyze_cache_performance(response):
    usage = response.usage
    
    cache_read = usage.cache_read_input_tokens      # Tokens from cache
    cache_write = usage.cache_creation_input_tokens # New tokens cached
    new_tokens = usage.input_tokens                 # Tokens after breakpoint
    output_tokens = usage.output_tokens
    
    total_input = cache_read + cache_write + new_tokens
    hit_rate = cache_read / total_input if total_input > 0 else 0
    
    cost_savings = (cache_write * BASE_COST + cache_read * CACHE_HIT_COST) \
                   / (total_input * BASE_COST)
    
    return {
        "total_input_tokens": total_input,
        "cache_hit_rate": hit_rate,
        "cost_savings_pct": cost_savings * 100,
        "cache_read": cache_read,
        "cache_write": cache_write,
        "new_tokens": new_tokens
    }
```

### 4.2 Debugging Cache Misses

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cache_creation` every request | Dynamic content before static | Reorder: static first |
| Low `cache_read` | TTL expired (5 min default) | Use 1-hour TTL or refresh |
| `cache_creation` high | New content each request | Move dynamic content after breakpoint |
| `input_tokens` ≈ total | Nothing cached | Check `cache_control` placement |

---

## 5. Decision Framework

```
NEED TOKEN OPTIMIZATION?
│
├─► High-volume, repetitive workloads?
│   └─► Prompt Caching (95%+ savings)
│
├─► Long documents, multi-question?
│   └─► Prompt Caching + 1-hr TTL
│
├─► Long conversations?
│   ├─► Auto-caching (default)
│   └─► Compaction + Tool clearing
│
├─► Cost critical?
│   ├─► Haiku 5.5 + Caching (99% savings)
│   ├─► Compress prompts (concise system, few-shot)
│   └─► Clear tool results, compress history
│
├─► Latency critical?
│   └─► Caching + smaller model (Haiku) + streaming
│
├─► Complex agents?
│   ├─► Subagent isolation (separate budgets)
│   ├─► Tool result clearing
│   └─► Compaction for long runs
│
└─► Compliance/Regulatory?
    └─► Explicit caching (audit trail) + 5-min TTL
```

---

## 6. Complete Token Optimization Cheat Sheet

```python
# ============================================
# TOKEN OPTIMIZATION MASTER CHECKLIST
# ============================================

# 1. PROMPT CACHING (Anthropic)
cache_config = {
    "type": "ephemeral",  # or {"type": "ephemeral", "ttl": "1h"}
    # Place on: system, tools, documents, few-shot
}

# 2. PROMPT COMPRESSION
def compress_prompt(system, tools, few_shot, query):
    # Concise system
    system = "Answer accurately. Cite sources. Be concise."
    # Minimal tools
    tools = [{"name": t.name, "desc": t.desc[:100]} for t in tools]
    # 2-3 diverse few-shots
    few_shot = few_shot[:3]
    # Static first, dynamic last
    return assemble(system, tools, few_shot, query)

# 3. RETRIEVAL COMPRESSION
def smart_retrieve(query, budget=15000):
    candidates = search(query, k=50)
    reranked = cross_encoder.rerank(query, candidates, top_k=20)
    selected = select_within_budget(reranked, budget)
    return selected

# 4. HISTORY COMPRESSION
def compress_history(messages, keep_recent=6):
    if tokens(messages) < 50000: return messages
    summary = llm.summarize(messages[:-keep_recent])
    return [SystemMessage(f"Summary: {summary}")] + messages[-keep_recent:]

# 5. TOOL RESULT CLEARING
def clear_tools(messages, keep=5):
    cleared = []
    seen = 0
    for m in reversed(messages):
        if m.type == "tool_result":
            seen += 1
            if seen > keep:
                m = replace(m, content="[Cleared]")
        cleared.append(m)
    return list(reversed(cleared))

# 6. SUBAGENT ISOLATION
def subagent(task, budget=5000):
    return create_react_agent(llm, tools, 
        config={"max_tokens": budget, "thread_id": f"sub-{task.id}"})

# 7. MONITORING
def track_metrics(response):
    u = response.usage
    return {
        "hit_rate": u.cache_read / (u.cache_read + u.cache_write + u.input),
        "cost_saved": u.cache_read * (BASE - CACHE_HIT) / 1e6,
        "latency_saved_ms": u.cache_read * 0.001  # approx
    }
```

---

## 6. Production Checklist

### 6.1 Pre-Launch

- [ ] **Prompt caching enabled** on static content (system, docs, tools, few-shot)
- [ ] **Static content first** in prompt structure
- [ ] **Cache TTL** appropriate (5-min default vs 1-hr for stable content)
- [ ] **Prompt compressed** (concise system, minimal tools, 2-3 few-shot)
- [ ] **Retrieval budgeted** with re-ranking
- [ ] **History compaction** enabled with custom prompt
- [ ] **Tool results cleared** after N turns
- [ ] **Subagent isolation** for independent tasks
- [ ] **Monitoring** for cache hit rate, latency, cost
- [ ] **Alerting** on cache miss spikes

### 6.2 Ongoing

- [ ] **Daily** cache hit rate review (target > 80%)
- [ ] **Weekly** prompt compression audit
- [ ] **Monthly** cost per task analysis
- [ ] **Quarterly** model/strategy re-evaluation

---

## 7. Quick Reference: Key Numbers

| Metric | Target | Notes |
|--------|--------|-------|
| **Min cacheable tokens** | 1,024 | Below = overhead > savings |
| **Default TTL** | 5 minutes | Auto-refresh on hit |
| **Extended TTL** | 1 hour | Additional cost |
| **Cache hit cost** | ~5% of base | e.g., $0.15 vs $3/MTok |
| **Cache write cost** | ~125% of base | e.g., $3.75 vs $3/MTok |
| **Target hit rate** | > 80% | For good ROI |
| **Min tokens for benefit** | ~2,000+ | Below = not worth it |

---

## 7. Resources & References

### Official
- **Anthropic Prompt Caching:** https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- **Anthropic Prompt Caching Cookbook:** https://github.com/anthropics/anthropic-cookbook
- **Building Effective Agents:** https://www.anthropic.com/research/building-effective-agents

### Videos
- **IBM Technology:** "What is Prompt Caching?" — https://www.youtube.com/watch?v=u57EnkQaUTY

### Related Guides (Your Library)
- **Context Engineering** — Complementary (context budget management)
- **Guardrails** — Input/output validation (adds tokens, needs budget)
- **A2A Protocol** — Multi-agent (each agent has own cache)
- **LangGraph** — Subagent isolation, compaction, tool clearing

---

## 8. Summary: Token Optimization Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TOKEN OPTIMIZATION HIERARCHY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TIER 1: PROMPT CACHING (Highest ROI)                                       │
│  ├─ 90-99% cost reduction for repeated prefixes                             │
│  ├─ Zero code changes (auto-caching)                                        │
│  └─ Essential for: docs, system prompts, few-shot, tools                    │
│                                                                             │
│  TIER 2: PROMPT COMPRESSION                                                 │
│  ├─ Concise system prompts, minimal tools, 2-3 few-shot                     │
│  ├─ Structured output schemas (Pydantic)                                    │
│  └─ 20-40% token reduction                                                  │
│                                                                             │
│  TIER 3: RETRIEVAL & HISTORY OPTIMIZATION                                   │
│  ├─ Re-ranking (cross-encoder) → precision                                  │
│  ├─ Budget-aware selection                                                  │
│  ├─ Compaction (summarize old turns)                                        │
│  ├─ Tool-result clearing (keep tool_use, drop content)                      │
│  └─ Subagent isolation (separate context budgets)                           │
│                                                                             │
│  TIER 4: MODEL SELECTION & PARALLELIZATION                                  │
│  ├─ Haiku for simple tasks (95% cheaper)                                    │
│  ├─ Parallel sectioning/voting (Anthropic patterns)                         │
│  └─ Subagent delegation                                                     │
│                                                                             │
│  TIER 5: ADVANCED TECHNIQUES                                                │
│  ├─ Explicit cache breakpoints (fine control)                               │
│  ├─ 1-hour TTL for stable content                                           │
│  ├─ Custom compaction prompts                                               │
│  └─ Structured memory (cross-session)                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```