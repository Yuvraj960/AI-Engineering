# LangGraph Lecture 2: What is Agentic AI?

## 1. Formal Definition of Agentic AI

### 1.1 Core Definition

> **Agentic AI** refers to AI systems that can **autonomously pursue goals** by making decisions, planning actions, using tools, and adapting to feedback — all while maintaining context across extended interactions.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AGENTIC AI vs WORKFLOWS (Anthropic)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WORKFLOWS (Deterministic)              │  AGENTS (Agentic)                 │
│  ─────────────────────────              │  ─────────────────                │
│  • Predefined code paths                │  • LLM dynamically directs        │
│  • Fixed sequence of steps              │    own process & tool usage       │
│  • Predictable, consistent              │  • Flexible, model-driven         │
│  • No runtime decisions                 │  • Runtime decision-making        │
│  • Example: RAG pipeline                │  • Example: Research agent        │
│                                         │                                   │
│  USE WHEN:                              │  USE WHEN:                        │
│  • Task is well-defined                 │  • Flexibility needed at scale    │
│  • Consistency > adaptability           │  • Open-ended problems            │
│  • Low error tolerance                  │  • Multi-step reasoning required  │
│                                         │                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 The Agentic Loop (Fundamental Pattern)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         THE AGENTIC LOOP                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│       ┌──────────────┐                                                      │
│       │   PERCEIVE   │  ◄── Observe environment, read state,                │
│       │  (Observe)   │       receive tool results, user input               │
│       └──────┬───────┘                                                      │
│              │                                                              │
│              ▼                                                              │
│       ┌──────────────┐                                                      │
│       │   REASON     │  ◄── Analyze, think, evaluate options                │
│       │  (Think)     │       (Chain of Thought, ReAct, Tree of Thoughts)    │
│       └──────┬───────┘                                                      │
│              │                                                              │
│              ▼                                                              │
│       ┌──────────────┐                                                      │
│       │    PLAN      │  ◄── Decide next action(s), create/update plan       │
│       │  (Decide)    │       (Task decomposition, prioritization)           │
│       └──────┬───────┘                                                      │
│              │                                                              │
│              ▼                                                              │
│       ┌──────────────┐                                                      │
│       │    ACT       │  ◄── Execute: call tools, write code, send API       │
│       │  (Execute)   │       requests, update memory                        │
│       └──────┬───────┘                                                      │
│              │                                                              │
│              ▼                                                              │
│       ┌──────────────┐                                                      │
│       │   LEARN      │  ◄── Update memory, extract lessons,                 │
│       │  (Reflect)   │       refine strategy for next iteration             │
│       └──────┬───────┘                                                      │
│              │                                                              │
│              └──────────────┐                                               │
│                             │                                               │
│                             ▼                                               │
│                      (Back to PERCEIVE)                                     │
│                                                                             │
│  THIS LOOP RUNS CONTINUOUSLY UNTIL GOAL ACHIEVED OR TERMINATION CONDITION   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Practical Example: HR Recruiter Agent

### 2.1 Scenario Walkthrough

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HR RECRUITER AGENT — FULL LIFECYCLE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GOAL: "Hire a Backend Engineer (2-4 years exp)"                            │
│                                                                             │
│  STEP 1: JD CREATION (Planning + Tools)                                     │
│  ─────────────────────────────────                                          │
│  Agent: "I'll draft a JD. First, let me access company docs to              │
│          understand tech stack for backend roles."                          │
│  → Tool: read_file(company_docs/tech_stack.md)                              │
│  → Tool: read_file(company_docs/hiring_policies.md)                         │
│  → Reasoning: "Need Python/FastAPI, PostgreSQL, AWS, 2-4 yrs exp"           │
│  → Output: Structured JD with requirements, responsibilities, benefits      │
│                                                                             │
│  STEP 2: CANDIDATE SOURCING (Tools + Autonomy)                              │
│  ──────────────────────────────────────────                                 │
│  Agent: "Now I'll search for candidates across platforms."                  │
│  → Tool: linkedin_search("Backend Engineer Python FastAPI 2-4 years")       │
│  → Tool: naukri_search("Python Django AWS backend")                         │
│  → Tool: github_search("FastAPI contributors")                              │
│  → Reasoning: "Filter for relevant skills, location, availability"          │
│  → Output: Ranked candidate list with profiles                              │
│                                                                             │
│  STEP 3: RESUME SCREENING (Reasoning + Adaptability)                        │
│  ─────────────────────────────────────────────                              │
│  Agent: "Screening 50 resumes against JD criteria."                         │
│  → For each resume: extract skills, experience, projects                    │
→ Tool: parse_resume(pdf) → LLM evaluation against criteria                   │
│  → Reasoning: "Candidate has FastAPI but no AWS — flag for interview"       │
│  → Adaptability: "Adjusting weight: portfolio > years of experience"        │
│  → Output: Shortlist with scores, strengths, concerns                       │
│                                                                             │
│  STEP 4: INTERVIEW SCHEDULING (Tools + Context Awareness)                   │
│  ──────────────────────────────────────────────────                         │
│  Agent: "Scheduling technical rounds for top 10 candidates."                │
│  → Tool: calendar_check(interviewers) → Tool: send_email(candidates)        │
│  → Context: "Candidate in EST, interviewer in IST — find overlap"           │
│  → Tool: timezone_convert → Tool: calendar_create_event                     │
│  → Output: Confirmed schedule with video links                              │
│                                                                             │
│  STEP 5: HIRING DECISION (Multi-step Reasoning + Supervisor)                │
│  ──────────────────────────────────────────────────                         │
│  Agent: "Evaluating interview feedback for final decision."                 │
│  → Aggregate: technical scores, cultural fit, compensation expectations     │
│  → Reasoning: "Candidate A strong technically but 30% over budget"          │
│  → Supervisor Review: "Escalate to hiring manager for budget exception"     │
│  → Tool: notify_hiring_manager(candidate_A, budget_concern)                 │
│  → Final: Extend offer / Reject with feedback                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Why This Is Agentic (Not Just a Workflow)

| Aspect | Workflow Version | Agentic Version |
|--------|------------------|-----------------|
| **JD Creation** | Fixed template | Reads live docs, reasons about requirements |
| **Sourcing** | Single platform, fixed query | Multi-platform, adaptive queries |
| **Screening** | Keyword matching | LLM reasoning, holistic evaluation |
| **Scheduling** | Fixed time slots | Timezone-aware, conflict resolution |
| **Decision** | Score threshold | Multi-factor reasoning + human escalation |
| **Adaptation** | None | Learns from each hire, adjusts criteria |

---

## 3. Six Key Characteristics of Agentic AI

### 3.1 Characteristic Matrix

| # | Characteristic | Definition | In HR Example | LangGraph Implementation |
|---|----------------|------------|---------------|--------------------------|
| **1** | **Autonomy** | Acts without constant human prompting | Sources candidates, screens resumes independently | Nodes execute without user input between steps |
| **2** | **Goal-Oriented** | Works toward explicit objective | "Hire backend engineer" drives all actions | State tracks goal; termination condition |
| **3** | **Planning** | Decomposes goals into steps | Creates sourcing → screening → interview plan | Plan node creates task list; conditional edges |
| **4** | **Reasoning** | Analyzes, evaluates, infers | Evaluates fit, weighs tradeoffs | LLM nodes with CoT, ReAct prompting |
| **5** | **Adaptability** | Adjusts to new info/conditions | Changes criteria, handles timezone conflicts | Dynamic routing, state-based decisions |
| **6** | **Context Awareness** | Maintains & uses context | Remembers candidate details, company policies | Shared state, memory store, checkpointer |

---

### 3.2 Deep Dive: Each Characteristic

#### 3.2.1 Autonomy
```
AUTONOMY SPECTRUM:
─────────────────────────────────────────────────────────────────────────────
L0: No Autonomy        →  Human does everything
L1: Assisted           →  AI suggests, human executes
L2: Partial Autonomy   →  AI executes sub-tasks, human approves major steps
L3: High Autonomy      →  AI executes full workflow, human oversees
L4: Full Autonomy      →  AI sets goals, executes, learns (rare, risky)

AGENTIC AI TARGET: L2-L3
• Autonomous WITHIN BOUNDARIES
• Human-in-the-loop for high-stakes decisions
• Guardrails prevent runaway actions
```

#### 3.2.2 Goal-Oriented
```python
# Goal representation in LangGraph state
class AgentState(TypedDict):
    goal: str                          # "Hire Backend Engineer"
    goal_decomposition: List[str]      # ["Create JD", "Source", "Screen", ...]
    current_subgoal: str               # "Screening resumes"
    success_criteria: Dict[str, Any]   # {"min_candidates": 5, "max_time_days": 14}
    termination_condition: Callable    # lambda s: s["hired"] or s["deadline_passed"]
```

#### 3.2.3 Planning
```
PLANNING TYPES:
─────────────────────────────────────────────────────────────────────────────
1. REACTIVE PLANNING      →  Next action based on current observation only
2. DELIBERATIVE PLANNING  →  Multi-step plan created upfront, then executed
3. CONTINUOUS PLANNING    →  Plan updated after each action (replanning)
4. HIERARCHICAL PLANNING  →  High-level goals → subgoals → primitive actions

HR AGENT USES: Continuous + Hierarchical
• High-level: "Hire engineer" → Subgoals: JD, Source, Screen, Interview, Decide
• After each interview: Replan remaining candidates, adjust criteria
```

#### 3.2.4 Reasoning
```
REASONING PATTERNS (Implemented via LLM prompting):
─────────────────────────────────────────────────────────────────────────────
• CHAIN OF THOUGHT (CoT)     →  "Think step by step..."
• REACT                      →  Thought → Action → Observation → Thought...
• TREE OF THOUGHTS (ToT)     →  Explore multiple reasoning branches
• REFLECTION                 →  Self-critique after action
• PLAN-EXECUTE               →  Separate planning and execution phases

LANGGRAPH IMPLEMENTATION:
• Each reasoning pattern = different node design
• CoT: Single LLM node with structured prompt
• ReAct: LLM node + Tool node in loop
• ToT: Parallel LLM nodes → evaluation node → selection
```

#### 3.2.5 Adaptability
```python
# Adaptability in LangGraph: Dynamic routing based on state
def adaptive_router(state: AgentState) -> str:
    """Route based on current situation."""
    
    # If screening finds few candidates → expand sourcing
    if state["screened_count"] < 5 and state["sources_exhausted"] == False:
        return "expand_sourcing"
    
    # If budget exceeded → escalate
    if state["top_candidate_salary"] > state["budget_max"]:
        return "escalate_to_manager"
    
    # If deadline near → fast-track
    if days_until_deadline(state) < 3:
        return "fast_track_interviews"
    
    return "continue_normal"
```

#### 3.2.6 Context Awareness
```
CONTEXT LAYERS IN AGENTIC AI:
─────────────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────────────────┐
│  LONG-TERM MEMORY (Cross-session)                                       │
│  • Company hiring policies                                              │
│  • Historical hiring data                                               │
│  • Learned preferences (e.g., "prefer open-source contributors")        │
├─────────────────────────────────────────────────────────────────────────┤
│  SHORT-TERM MEMORY (Thread/Session)                                     │
│  • Current candidate pool                                               │
│  • Interview feedback so far                                            │
│  • Decisions made this hiring cycle                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  IMMEDIATE CONTEXT (Current Turn)                                       │
│  • Current resume being evaluated                                       │
│  • Tool results from last action                                        │
│  • User's latest instruction                                            │
└─────────────────────────────────────────────────────────────────────────┘

LANGGRAPH: checkpointer (short-term) + store (long-term) + state (immediate)
```

---

## 4. Five Components of Agentic AI Architecture

### 4.1 Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AGENTIC AI ARCHITECTURE (5 COMPONENTS)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         SUPERVISOR                                  │   │
│   │  • High-level oversight                                             │   │
│   │  • Policy enforcement (budget, compliance, ethics)                  │   │
│   │  • Escalation handling                                              │   │
│   │  • Multi-agent coordination (if applicable)                         │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                           │
│                                 ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         BRAIN (LLM)                                 │   │
│   │  • Core reasoning engine                                            │   │
│   │  • Planning, decision-making, reflection                            │   │
│   │  • Prompted with: goal, context, tools, memory                      │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                           │
│              ┌──────────────────┼──────────────────┐                        │
│              ▼                  ▼                  ▼                        │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│   │   ORCHESTRATOR  │  │      TOOLS      │  │     MEMORY      │             │
│   │  (Nervous Sys)  │  │  (Effectors)    │  │  (Knowledge)    │             │
│   │                 │  │                 │  │                 │             │
│   │ • Routes tasks  │  │ • External APIs │  │ • Short-term    │             │
│   │ • Manages flow  │  │ • Databases     │  │   (thread state)│             │
│   │ • Handles errors│  │ • File system   │  │ • Long-term     │             │
│   │ • Coordinates   │  │ • Code exec     │  │   (store)       │             │
│   │   sub-agents    │  │ • Comm (email,  │  │ • Episodic      │             │
│   │                 │  │   Slack, etc.)  │  │   (experiences) │             │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Deep Dive

#### 4.2.1 Brain (The LLM)
```python
# Brain configuration in LangGraph
class BrainConfig:
    model: str = "claude-3-5-sonnet-20241022"
    temperature: float = 0.1  # Low for consistent reasoning
    max_tokens: int = 8192
    
    # System prompt defines agent's "personality" and capabilities
    system_prompt: str = """
    You are an expert HR Recruiter Agent. Your goal: {goal}.
    
    CAPABILITIES:
    - Reason about candidate fit holistically
    - Plan multi-step hiring workflows
    - Adapt strategy based on outcomes
    - Escalate when policy requires
    
    CONSTRAINTS:
    - Never discriminate (age, gender, nationality, etc.)
    - Respect budget limits
    - Require human approval for offers > $X
    - Log all decisions for audit
    """
```

#### 4.2.2 Orchestrator (The Nervous System / Project Manager)
```python
# Orchestrator = LangGraph Graph Structure
# It's not a separate component — it's the GRAPH ITSELF

# The orchestrator manages:
# 1. State flow between nodes
# 2. Conditional routing (decision points)
# 3. Error handling & retries
# 4. Checkpointing (persistence)
# 5. Human-in-the-loop interrupts
# 6. Subgraph invocation (for sub-agents)

# Example: HR Agent Graph Structure
graph = StateGraph(HRState)

# Nodes = Brain's reasoning steps + Tool executions
graph.add_node("create_jd", create_jd_node)          # Brain: Plan JD
graph.add_node("source_candidates", source_node)     # Tools: Search
graph.add_node("screen_resumes", screen_node)        # Brain: Evaluate
graph.add_node("schedule_interviews", schedule_node) # Tools: Calendar
graph.add_node("evaluate_feedback", evaluate_node)   # Brain: Decide
graph.add_node("human_approval", hitl_node)          # Supervisor gate

# Edges = Orchestrator's flow control
graph.add_edge(START, "create_jd")
graph.add_edge("create_jd", "source_candidates")
graph.add_conditional_edges("source_candidates", 
    lambda s: "screen" if s["candidates"] > 0 else "expand_sourcing")
graph.add_edge("screen_resumes", "schedule_interviews")
graph.add_edge("schedule_interviews", "evaluate_feedback")
graph.add_conditional_edges("evaluate_feedback",
    lambda s: "human_approval" if s["needs_approval"] else "extend_offer")
graph.add_edge("human_approval", "extend_offer")
graph.add_edge("extend_offer", END)
```

#### 4.2.3 Tools (The Effectors)
```python
# Tool definitions for HR Agent
from langchain_core.tools import tool

@tool
def linkedin_search(query: str, filters: dict) -> List[Candidate]:
    """Search LinkedIn for candidates matching criteria."""
    ...

@tool
def parse_resume(file_path: str) -> ParsedResume:
    """Extract structured data from resume PDF."""
    ...

@tool
def calendar_schedule(interviewer_ids: List[str], 
                      candidate_email: str,
                      duration_min: int = 60) -> ScheduledEvent:
    """Find mutual availability and schedule interview."""
    ...

@tool
def send_email(to: str, template: str, data: dict) -> bool:
    """Send templated email (offer, rejection, interview invite)."""
    ...

@tool
def notify_hiring_manager(candidate: Candidate, reason: str) -> None:
    """Escalate to human for approval."""
    ...

# Tools are bound to LLM in Brain nodes
llm_with_tools = llm.bind_tools([linkedin_search, parse_resume, 
                                   calendar_schedule, send_email, 
                                   notify_hiring_manager])
```

#### 4.2.4 Memory (The Knowledge Base)
```python
# Memory layers in LangGraph

# 1. SHORT-TERM: Thread state (automatic with checkpointer)
#    Contains: messages, current candidates, interview feedback, decisions
#    Persisted per thread_id, resumed automatically

# 2. LONG-TERM: Cross-session store
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# Save hiring preferences learned over time
async def save_hiring_pattern(state, store):
    key = ("hiring_patterns", state["department"])
    pattern = store.get(key) or {"successful_profiles": [], "red_flags": []}
    
    # Learn from this hire
    if state.get("hire_successful"):
        pattern["successful_profiles"].append({
            "skills": state["hired_candidate"].skills,
            "source": state["hired_candidate"].source,
            "interview_score": state["hired_candidate"].avg_score
        })
    
    store.put(key, pattern)

# Retrieve for future hiring
async def get_hiring_insights(state, store):
    pattern = store.get(("hiring_patterns", state["department"]))
    return {"hiring_insights": pattern}
```

#### 4.2.5 Supervisor (The Governance Layer)
```python
# Supervisor = Policy enforcement + Escalation
# Implemented as: Special nodes + Conditional edges + Interrupts

# Supervisor checks (run at key decision points)
def supervisor_check(state: HRState) -> HRState:
    violations = []
    
    # Budget check
    if state.get("proposed_salary", 0) > state["budget_max"]:
        violations.append("salary_exceeds_budget")
    
    # Compliance check
    if any(bias_flag in state.get("screening_notes", "") 
           for bias_flag in ["age", "gender", "nationality"]):
        violations.append("potential_bias_detected")
    
    # Policy check
    if state.get("offer_without_approval") and state["role_level"] >= "senior":
        violations.append("senior_hire_requires_approval")
    
    if violations:
        # Trigger human-in-the-loop
        return {"supervisor_violations": violations, "needs_human_review": True}
    
    return {"supervisor_violations": [], "needs_human_review": False}

# In graph: Add supervisor node before critical actions
graph.add_node("supervisor_check", supervisor_check)
graph.add_conditional_edges("evaluate_feedback",
    lambda s: "human_review" if s["needs_human_review"] else "extend_offer")
```

---

## 5. Levels of Autonomy (Formal Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LEVELS OF AGENT AUTONOMY                                 │
├────────┬──────────────────────┬─────────────────────────────────────────────┤
│ Level  │ Name                 │ Description                                 │
├────────┼──────────────────────┼─────────────────────────────────────────────┤
│ L0     │ Tool Only            │ Human drives; AI executes single functions  │
│ L1     │ Guided Workflow      │ AI follows fixed sequence; human triggers   │
│ L2     │ Supervised Agent     │ AI plans & acts; human approves key steps   │
│ L3     │ Autonomous Agent     │ AI runs independently; human monitors       │
│ L4     │ Self-Governing       │ AI sets own goals; human defines boundaries │
│ L5     │ Fully Autonomous     │ AI operates without human involvement       │
├────────┼──────────────────────┼─────────────────────────────────────────────┤
│ TARGET │ L2-L3 (Production)   │ Balance of autonomy & safety                │
└────────┴──────────────────────┴─────────────────────────────────────────────┘
```

| Level | Human Role | Agent Role | Example |
|-------|------------|------------|---------|
| **L0** | Does everything | Executes `search()` | Human types query, clicks result |
| **L1** | Triggers steps | Runs fixed pipeline | Human clicks "Run Hiring Pipeline" |
| **L2** | Approves offers | Plans, sources, screens, schedules | HR Agent (our example) |
| **L3** | Monitors dashboard | Full cycle, self-corrects | Auto-hiring for high-volume roles |
| **L4** | Sets policy | Identifies needs, initiates hiring | "We need more engineers" → Agent hires |
| **L5** | None | Runs company | Theoretical / unsafe |

> **LangGraph Sweet Spot:** L2-L3 with **Human-in-the-Loop** for L2, **Observability** for L3

---

## 6. Agentic AI vs Traditional Approaches

### 6.1 Comparison Matrix

| Dimension | Traditional Code | RAG/Workflow | Agentic AI (LangGraph) |
|-----------|------------------|--------------|------------------------|
| **Control Flow** | Hardcoded if/else | Fixed DAG | Dynamic, state-driven |
| **Decision Making** | Rules/Heuristics | Single LLM call | Multi-step reasoning |
| **Tool Use** | Direct function calls | Predefined sequence | Dynamic selection |
| **Error Handling** | Try/catch | Retry policies | Reflection + replanning |
| **State Management** | Variables/DB | Passed between steps | Shared state + checkpoints |
| **Human Interaction** | Blocking UI | Approval gates | Interrupt + resume |
| **Learning** | None | None | Memory + reflection |
| **Adaptability** | Code changes | Prompt changes | Runtime adaptation |
| **Debugging** | Stack traces | Logs | Time travel + replay |

---

## 7. When to Use Agentic AI (Decision Framework)

```
SHOULD YOU BUILD AN AGENT?
─────────────────────────────────────────────────────────────────────────────
│
├─► Is the task OPEN-ENDED? (Multiple valid approaches)
│   └─► YES → Agentic AI
│   └─► NO → Workflow / Traditional code
│
├─► Does it require MULTI-STEP REASONING?
│   └─► YES → Agentic AI
│   └─► NO → Single LLM call / RAG
│
├─► Does it need TOOL ORCHESTRATION? (Dynamic tool choice)
│   └─► YES → Agentic AI
│   └─► NO → Fixed pipeline
│
├─► Does it need LONG-RUNNING STATE? (Hours/days, resume after crash)
│   └─► YES → Agentic AI (LangGraph persistence)
│   └─► NO → Stateless function
│
├─► Does it need HUMAN OVERSIGHT at key points?
│   └─► YES → Agentic AI (HITL)
│   └─► NO → Consider full autonomy (L3+)
│
├─► Is the COST OF ERROR HIGH? (Legal, safety, financial)
│   └─► YES → Add heavy guardrails + HITL (L2)
│   └─► NO → More autonomy OK (L3)
│
└─► DEFAULT: Start with WORKFLOW, upgrade to AGENT when needed
    (Anthropic: "Find simplest solution, increase complexity only when needed")
```

---

## 8. Common Agent Design Patterns

| Pattern | Description | When to Use | LangGraph Implementation |
|---------|-------------|-------------|-------------------------|
| **ReAct** | Reason → Act → Observe loop | General purpose, tool use | LLM node + Tool node in cycle |
| **Plan-Execute** | Separate planner + executor | Complex, long tasks | Planner node → Executor subgraph |
| **Reflection** | Self-critique after action | Quality-critical tasks | Reflection node after each action |
| **ReWOO** | Reason without observation | Reduce tool calls | Planner creates full plan upfront |
| **Multi-Agent** | Specialized agents collaborate | Distinct expertise areas | Subgraphs + Supervisor node |
| **Human-in-Loop** | Pause for human approval | High-stakes decisions | `interrupt()` + `Command(resume=...)` |

---

## 9. Quick Reference: Agentic AI Concepts Cheat Sheet

```python
# ============================================
# AGENTIC AI ESSENTIALS
# ============================================

# 1. THE AGENTIC LOOP
while not goal_achieved:
    observe = perceive(state)
    thought = reason(observe, memory, goal)
    plan = update_plan(thought)
    action = decide_action(plan)
    result = execute(action, tools)
    learn = reflect(result, memory)
    state = update_state(state, result, learn)

# 2. SIX CHARACTERISTICS
AUTONOMY        # Acts within boundaries
GOAL_ORIENTED   # Driven by explicit objective
PLANNING        # Decomposes & sequences actions
REASONING       # Analyzes, evaluates, infers
ADAPTABILITY    # Adjusts to feedback/conditions
CONTEXT_AWARE   # Uses short + long-term memory

# 3. FIVE COMPONENTS
BRAIN       → LLM (reasoning engine)
ORCHESTRATOR → Graph (flow control)
TOOLS       → Functions (external actions)
MEMORY      → Checkpointer + Store (knowledge)
SUPERVISOR  → Policies + Interrupts (governance)

# 4. LANGGRAPH MAPPING
State           → Shared context (TypedDict/Pydantic)
Nodes           → Brain reasoning + Tool execution
Edges           → Orchestrator flow control
Checkpointer    → Short-term memory (thread)
Store           → Long-term memory (cross-session)
Interrupt       → Supervisor/HITL gates
Streaming       → Real-time observability

# 5. AUTONOMY LEVELS
L0: Tool only          L3: Autonomous (monitored)
L1: Guided workflow    L4: Self-governing
L2: Supervised agent   L5: Fully autonomous
TARGET: L2-L3 with HITL

# 6. WHEN AGENTIC
✓ Open-ended, multi-step, tool orchestration, long-running, HITL needed
✗ Fixed, simple, single-step, stateless, fully automated safe tasks
```

---

## 10. Key Takeaways

| Concept | Essence |
|---------|---------|
| **Agentic AI** | Autonomous goal pursuit via Perceive→Reason→Plan→Act→Learn loop |
| **6 Characteristics** | Autonomy, Goal-Oriented, Planning, Reasoning, Adaptability, Context Awareness |
| **5 Components** | Brain (LLM), Orchestrator (Graph), Tools, Memory, Supervisor |
| **HR Recruiter Example** | Demonstrates all characteristics + components in real scenario |
| **Autonomy Levels** | L0-L5; Production targets L2-L3 with HITL |
| **LangGraph Mapping** | Each component maps to LangGraph primitive |
| **Decision Framework** | Use agents for open-ended, multi-step, tool-heavy, long-running tasks |

---

## 12. Resources & References

### Videos
- **Playlist:** https://www.youtube.com/watch?v=yC36gN-rqjo&list=PLKnIA16_RmvYsvB8qkUQuJmJNuiCUJFPL

### Prerequisite Guides (Your Library)
- **Lecture 1:** `langgraph-lecture-1-foundations.md`
- **Context Engineering:** `context-engineering-independent-lecture.md`
- **Guardrails:** `guardrails-independent-lecture.md`
- **Agent Testing:** `agent-testing-independent-lecture.md`

### Theoretical Foundations
- **Anthropic:** "Building Effective Agents" — Workflows vs Agents distinction
- **ReAct Paper:** Yao et al. (2022) — Reasoning + Acting
- **Plan-Execute:** "Plan-and-Solve" / "ReWOO" papers
