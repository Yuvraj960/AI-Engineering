# LangSmith — Observability, Evaluation & Debugging for GenAI Applications

## 1. Why LangSmith? The Observability Problem

### 1.1 The "Black Box" Crisis in LLM Applications

> **Real-World Scenario:** A startup builds an AI cover-letter generator. Users love it. Suddenly latency spikes from 2 min → 10 min. Users churn. Revenue drops.

**The Problem:** You only see **Input → Output + Total Latency**. No visibility into:
- Which LLM call is slow? (JD parsing? Skill matching? Cover letter generation? Proofreading?)
- Which component fails? (Retriever? LLM? Parser? External API?)
- Why costs spike? (Token explosion? Retry loops? Agent loops?)
- Why quality drops? (Hallucination? Bad retrieval? Bad prompt?)

```
┌─────────────────────────────────────────────────────────────────┐
│                  WITHOUT LANGSMITH                              │
├─────────────────────────────────────────────────────────────────┤
│  USER INPUT                    │        FINAL OUTPUT            │
│  "Generate cover letter"       │        "Here's your letter..." │
│                                │                                │
│  ❌ NO visibility into:        │  ❌ NO per-component metrics  │
│  - Individual LLM calls        │  - Latency per component       │
│  - Tool invocations            │  - Tokens per call             │
│  - Retrieval steps             │  - Cost per call               │
│  - Agent reasoning steps       │  - Error details               │
│  - Retry loops                 │  - Input/Output per step       │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 The Cost & Quality Crisis

| Scenario | Root Cause | Without LangSmith | With LangSmith |
|----------|------------|-------------------|----------------|
| **Cost spikes** (50¢ → ₹2/report) | Agent prompt change → infinite loop | ❌ Blind | ✅ Trace shows repeated retriever calls |
| **Latency spikes** (2 min → 10 min) | Code scans entire Google Drive vs folder | ❌ Blind | ✅ Trace shows Drive API duration |
| **Hallucination** | Retriever returns wrong chunks | ❌ Blind | ✅ Trace shows retrieved chunks vs query |
| **Quality drops** | Prompt change breaks logic | ❌ Blind | ✅ Compare traces before/after |

---

## 2. Core Concepts

### 2.1 The Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORGANIZATION                                 │
│  (Team/Company level - billing, members, settings)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        PROJECT                                  │
│  (Logical grouping - "rag-demo", "agent-prod", "eval-test")     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        TRACE                                    │
│  (Single execution - one user request, one agent run)           │
│  Contains: metadata, tags, latency, tokens, cost, status        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          RUN                                    │
│  (Individual component execution within a trace)                │
│  Examples: prompt_template, chat_openai, retriever, tool_call   │
│  Each run has: input, output, latency, tokens, error, metadata  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 What Gets Traced Automatically

| Component | What's Captured |
|-----------|-----------------|
| **LLM Calls** | Model, prompt, completion, tokens (in/out), latency, cost |
| **Tool Calls** | Tool name, arguments, result, latency, errors |
| **Retrievers** | Query, retrieved docs (with scores), latency |
| **Chains/Runnables** | Input, output, each step's I/O, latency |
| **Agents** | Full reasoning loop: thought → action → observation |
| **Embeddings** | Text → vector, latency, token count |

---

## 3. Setup & Configuration

### 3.1 Environment Variables (Minimal Setup)

```bash
# .env file
OPENAI_API_KEY=sk-proj-xxxxx
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
LANGCHAIN_API_KEY=lsv2_pt_xxxxx
LANGCHAIN_PROJECT=my-project-name
```

### 3.2 Programmatic Configuration (Advanced)

```python
import os
from langchain_core.runnables import RunnableConfig

# Per-invocation config
config = RunnableConfig(
    tags=["production", "rag", "v2.1"],
    metadata={
        "user_id": "user_123",
        "session_id": "sess_456",
        "model": "gpt-4o-mini",
        "temperature": 0,
        "version": "2.1.0"
    },
    max_concurrency=10,
    run_name="rag_query_handler",
    callbacks=[LangChainTracer()]  # Automatic if env vars set
)

# Usage
result = chain.invoke({"question": "What is RAG?"}, config=config)
```

### 3.3 SDK Client (For Custom Operations)

```python
from langsmith import Client

client = Client(
    api_key="lsv2_pt_xxxxx",
    api_url="https://api.smith.langchain.com"
)

# Create project
client.create_project(project_name="my-rag-app", description="Production RAG")

# Log custom run
client.create_run(
    name="custom_operation",
    run_type="chain",
    inputs={"query": "user question"},
    outputs={"answer": "llm response"},
    project_name="my-rag-app"
)
```

---

## 4. Tracing Deep Dive

### 4.1 Automatic Tracing (Zero Code Changes)

```python
# Just set env vars - everything auto-traced!
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Answer: {question}")
llm = ChatOpenAI(model="gpt-4o-mini")
parser = StrOutputParser()

chain = prompt | llm | parser

# Automatically traced!
result = chain.invoke({"question": "What is LangSmith?"})
# Check LangSmith UI → Project → Traces
```

### 4.2 What You See in a Trace

```
Trace: "RunnableSequence" (run_name)
├── Run 1: ChatPromptTemplate
│   Input:  {"question": "What is LangSmith?"}
│   Output: "Answer: What is LangSmith?" (PromptValue)
│   Latency: 2ms
│
├── Run 2: ChatOpenAI (gpt-4o-mini)
│   Input:  PromptValue with system + human messages
│   Output: AIMessage(content="LangSmith is a platform...")
│   Latency: 1,247ms
│   Tokens:  45 in / 156 out
│   Cost:    $0.0003
│
├── Run 3: StrOutputParser
│   Input:  AIMessage
│   Output: "LangSmith is a platform..."
│   Latency: 1ms
│
Total Latency: 1,250ms | Total Tokens: 201 | Total Cost: $0.0003
```

### 4.3 Customizing Traces

```python
from langchain_core.runnables import RunnableConfig

# Per-run customization
config = RunnableConfig(
    run_name="rag_query_v2",           # Custom trace name
    tags=["rag", "production", "v2"],  # Filterable tags
    metadata={
        "user_id": "user_123",
        "model": "gpt-4o-mini",
        "temperature": 0,
        "retriever_k": 5
    }
)

# In chain
result = chain.invoke({"question": "What is RAG?"}, config=config)
```

### 4.4 RAG Application Tracing (Complete Example)

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_core.output_parsers import StrOutputParser

# Setup (auto-traced)
loader = PyPDFLoader("doc.pdf")
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
chunks = splitter.split_documents(docs)

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# RAG Chain (every component traced)
def format_docs(docs):
    return "\n\n".join(f"[Source: {d.metadata.get('source','?')}]\n{d.page_content}" for d in docs)

prompt = ChatPromptTemplate.from_template("""
Answer using ONLY context. Cite sources [1], [2].

Context: {context}
Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | prompt
    | llm
    | StrOutputParser()
)

# Invoke with tracing config
config = RunnableConfig(
    run_name="rag_query",
    tags=["rag", "pdf-qa"],
    metadata={"doc": "annual_report.pdf"}
)

result = rag_chain.invoke({"question": "What were Q3 revenues?"}, config=config)
```

**What You'll See in LangSmith:**
- Trace shows: retriever (query + 4 docs) → prompt → LLM → parser
- Each doc with metadata visible
- Token counts per component
- Latency breakdown: retriever 45ms, LLM 890ms, parser 1ms

---

## 5. Evaluation & Testing (LangSmith's Killer Feature)

### 5.1 Dataset Creation

```python
from langsmith import Client

client = Client()

# Create dataset
dataset = client.create_dataset(
    dataset_name="rag-qa-eval",
    description="RAG Q&A evaluation set",
    data_type="kv"  # key-value (question -> answer)
)

# Add examples
examples = [
    {"inputs": {"question": "What is RAG?"}, "outputs": {"answer": "RAG combines retrieval..."}},
    {"inputs": {"question": "How does retrieval work?"}, "outputs": {"answer": "Retrieval finds..."}},
    {"inputs": {"question": "What is LCEL?"}, "outputs": {"answer": "LCEL is LangChain Expression Language..."}},
]

for ex in examples:
    client.create_example(
        inputs=ex["inputs"],
        outputs=ex["outputs"],
        dataset_id=dataset.id
    )
```

### 5.2 Evaluator Types

| Evaluator Type | Use Case | Example |
|----------------|----------|---------|
| **Correctness** | QA with ground truth | "Is answer factually correct?" |
| **Criteria** | Custom rubrics | "Is tone professional? Is answer concise?" |
| **String Distance** | Exact match fuzzy | Levenshtein, Jaro-Winkler |
| **Embedding Distance** | Semantic similarity | Cosine similarity of embeddings |
| **Custom LLM Judge** | Complex reasoning | "Rate 1-5: Does answer cite sources?" |
| **Regex/Exact Match** | Structured output | JSON schema validation |

### 5.3 Running Evaluations

```python
from langsmith.evaluation import evaluate
from langsmith.schemas import Run, Example
from langchain_openai import ChatOpenAI

# Define target (your chain)
def predict(example: dict):
    return rag_chain.invoke({"question": example["question"]})

# Run evaluation
results = evaluate(
    predict,
    data="rag-qa-eval",  # Dataset name
    evaluators=[
        "correctness",           # Built-in: compare to ground truth
        "criteria:conciseness",  # Custom criteria
        "criteria:hallucination", # Custom: "no made-up facts"
        "embedding_distance",    # Semantic similarity
    ],
    experiment_prefix="rag-eval-v1",
    metadata={"version": "2.1", "model": "gpt-4o-mini"},
    max_concurrency=5,
)

# View results in LangSmith UI → Experiments
```

### 5.4 Custom Evaluators

```python
from langsmith.evaluation import Evaluator
from langsmith.schemas import Run, Example

# Custom: Citation checker
def has_citations(run: Run, example: Example) -> dict:
    answer = run.outputs.get("answer", "")
    citations = answer.count("[")
    score = min(citations / 2, 1.0)  # Expect at least 2 citations
    return {"key": "citation_score", "score": score}

# Custom: Faithfulness (answer grounded in context)
def faithfulness_check(run: Run, example: Example) -> dict:
    # Use LLM-as-judge
    judge = ChatOpenAI(model="gpt-4o", temperature=0)
    prompt = f"""
    Context: {example.inputs.get('context', '')}
    Answer: {run.outputs.get('answer', '')}
    
    Is the answer fully supported by the context? (yes/no)
    """
    result = judge.invoke(prompt)
    return {"key": "faithful", "score": 1.0 if "yes" in result.content.lower() else 0.0}

# Run with custom evaluators
results = evaluate(
    predict,
    data="rag-qa-eval",
    evaluators=[has_citations, faithfulness_check, "correctness"],
)
```

### 5.5 Evaluation Cost Management (Dave Ebbelaar's Warning)

> **Real Experience:** 4-record dataset → 2,200 tokens → $1.30 for one evaluation run.

| Strategy | Impact |
|----------|--------|
| **Subsample** | Evaluate 10-20% of production traffic |
| **Batch offline** | Run nightly, not per-request |
| **Cheap evaluators first** | String distance < LLM judge |
| **Cache results** | Avoid re-evaluating same inputs |
| **Monitor dashboard** | Set cost alerts |

---

## 6. LangGraph + LangSmith Integration

### 6.1 Automatic Tracing

```python
from langgraph.graph import StateGraph, MessagesState
from langgraph.prebuilt import ToolNode, create_react_agent
from langchain_openai import ChatOpenAI

# Just set env vars - LangGraph auto-traces!
llm = ChatOpenAI(model="gpt-4o-mini")
tools = [search_web, get_weather, calculate]

agent = create_react_agent(llm, tools)

# Every graph execution = one trace
result = agent.invoke({"messages": [HumanMessage("Weather in Tokyo?")]})
# LangSmith shows: agent loop → tool calls → tool results → LLM → done
```

### 6.2 What You See in Agent Traces

```
Trace: "Agent Execution"
├── Run: Agent Node (LLM reasoning)
│   Input: Messages + tools
│   Output: AIMessage with tool_calls
│
├── Run: ToolNode (Parallel execution)
│   ├── Run: search_web("Tokyo weather")
│   │   Input: {"query": "Tokyo weather"}
│   │   Output: "Tokyo: 22°C, Sunny"
│   │   Latency: 1,200ms
│   ├── Run: get_weather("Tokyo")
│   │   Input: {"city": "Tokyo"}
│   │   Output: {"temp": 22, "condition": "Sunny"}
│   │   Latency: 800ms
│
├── Run: Agent Node (LLM with tool results)
│   Input: Messages + tool results
│   Output: Final AIMessage
│
Total: 3.2s | 5 LLM calls | 12 tool calls
```

---

## 7. Advanced Features

### 6.1 Prompt Playground

- Test prompts interactively in UI
- Version control prompts
- Compare outputs across models
- Export as code

### 6.2 Annotations & Feedback

```python
# Annotate runs for training data
client.create_feedback(
    run_id="run_abc123",
    key="quality",
    score=1.0,
    comment="Perfect answer with citations"
)

# Collect user feedback (thumbs up/down)
client.create_feedback(
    run_id="run_abc123",
    key="user_satisfaction",
    score=0.0,
    comment="User reported hallucination"
)
```

### 6.3 Monitoring Dashboards

| Metric | Alert Threshold |
|--------|-----------------|
| **Latency p95** | > 10s |
| **Error rate** | > 5% |
| **Cost per 1k requests** | > $5 |
| **Token usage** | > 100k/day |
| **Hallucination rate** | > 2% (via evaluator) |

### 6.4 Automations (Rules & Webhooks)

```yaml
# Example: Alert on high hallucination
rule:
  name: "Hallucination Alert"
  condition: "feedback.hallucination_score > 0.5"
  actions:
    - webhook: "https://hooks.slack.com/xxx"
    - email: "team@company.com"
```

---

## 7. LangSmith vs Traditional ML Observability

| Aspect | Traditional ML (MLflow) | LangSmith (LLM-Native) |
|--------|------------------------|------------------------|
| **Trace Structure** | Flat metrics | Hierarchical (trace → runs) |
| **Data Types** | Numbers, params | Prompts, completions, tools, docs |
| **Evaluation** | Metrics (accuracy, F1) | LLM-as-judge, criteria, semantic |
| **Debugging** | Metrics charts | Full I/O replay per component |
| **Prompt Versioning** | Manual | Built-in playground + versioning |
| **Cost Tracking** | Manual | Automatic (tokens × model pricing) |

---

## 7. Production Patterns

### 7.1 A/B Testing Prompts

```python
# Version prompts in code
PROMPT_V1 = "Answer concisely: {question}"
PROMPT_V2 = "Answer with citations: {question}"

# Tag runs with version
config_v1 = RunnableConfig(tags=["prompt:v1"])
config_v2 = RunnableConfig(tags=["prompt:v2"])

# Compare in LangSmith: filter by tag, compare latency/cost/quality
```

### 7.2 Canary Deployments

```python
# Route 10% traffic to new model
import random

def get_chain():
    if random.random() < 0.1:
        return chain_v2  # New prompt/model
    return chain_v1  # Current production

# Tag accordingly
config = RunnableConfig(tags=["canary" if is_v2 else "stable"])
```

### 7.3 Regression Detection

```python
# Nightly evaluation job
def nightly_eval():
    results = evaluate(
        predict=current_chain,
        data="regression-test-set",
        evaluators=["correctness", "faithfulness", "latency"],
        experiment_prefix="nightly-regression"
    )
    
    # Alert if regression
    if results["faithfulness"] < 0.85:
        send_alert("Faithfulness dropped below 85%!")
```

---

## 8. Cost & Performance Optimization

### 8.1 Token & Cost Tracking

```python
# Per-trace cost breakdown visible in UI
# LLM Run: 45 in / 156 out tokens = $0.0003
# Embedding Run: 1000 tokens = $0.0001
# Total trace cost: $0.0004

# Dashboard queries:
# - Cost per project/day
# - Cost per user session
# - Cost per feature (tag)
```

### 8.2 Latency Optimization

```python
# Trace shows: retriever 45ms, LLM 890ms, parser 1ms
# Optimization targets:
# 1. LLM: Use gpt-4o-mini, reduce max_tokens, enable streaming
# 2. Retriever: Add caching, reduce k, use MMR
# 3. Parallel: Use RunnableParallel for independent ops
```

---

## 8. Decision Flowchart

```
Need observability for LLM app?
│
├─► Just starting?
│   └─► Set env vars → Auto-trace everything
│
├─► Need to debug specific issue?
│   ├─► Latency → Check per-run latency breakdown
│   ├─► Cost → Check token/cost per component
│   ├─► Quality → Check retrieved docs, LLM reasoning
│   └─► Errors → Check run status + error details
│
├─► Need evaluation?
│   ├─► Ground truth exists → Correctness evaluator
│   ├─► No ground truth → Criteria / LLM judge
│   ├─► Semantic similarity → Embedding distance
│   └─► Custom rules → Custom evaluator
│
├─► Production monitoring?
│   ├─► Dashboards for latency/cost/error
│   ├─► Alerts on thresholds
│   ├─► Feedback collection
│   └─► Automated evaluations (nightly)
│
└─► Agent debugging?
    └─► Full reasoning loop visibility
        → Thought → Action → Observation → ...
```

---

## 9. Quick Reference

### Environment Variables
| Variable | Required | Description |
|----------|----------|-------------|
| `LANGCHAIN_TRACING_V2` | Yes | Enable tracing (`true`) |
| `LANGCHAIN_API_KEY` | Yes | Your LangSmith API key |
| `LANGCHAIN_PROJECT` | No | Project name (default: "default") |
| `LANGCHAIN_ENDPOINT` | No | API endpoint (default: cloud) |
| `LANGCHAIN_TAGS` | No | Default tags for all runs |

### Key Classes & Methods
| Class/Method | Purpose |
|--------------|---------|
| `RunnableConfig` | Per-invocation config (tags, metadata, run_name) |
| `langsmith.Client` | SDK for datasets, experiments, feedback |
| `langsmith.evaluate()` | Run evaluations on datasets |
| `RunnableWithMessageHistory` | Auto-traced conversational chains |
| `create_react_agent` | Auto-traced LangGraph agents |

### UI Navigation
```
Projects → [project] → Traces → [trace] → Runs → [run] → Details
                    ↓
              Experiments → [experiment] → Results
                    ↓
              Datasets → [dataset] → Examples
                    ↓
              Prompt Hub → [prompt] → Versions
```

---

## 10. LangSmith in Your GenAI Stack

| Layer | Tool | LangSmith Role |
|-------|------|----------------|
| **Framework** | LangChain / LangGraph | Auto-tracing |
| **LLM** | OpenAI / Anthropic / Local | Token/cost/latency tracking |
| **Vector DB** | Chroma / Pinecone / FAISS | Retrieval tracing |
| **Observability** | **LangSmith** | **Traces, evals, monitoring** |
| **Deployment** | LangServe / FastAPI / Vercel | Production monitoring |
| **CI/CD** | GitHub Actions / GitLab | Automated evaluations |

---

## 11. Resources & Next Steps

### Official Resources
- **Docs:** https://docs.langchain.com/langsmith/observability
- **Platform:** https://smith.langchain.com
- **Cookbook:** https://github.com/langchain-ai/langsmith-cookbook
- **SDK:** `pip install langsmith`

### Learning Path
1. **Setup** → Environment variables → First trace
2. **Trace** → RAG chain → Inspect runs
3. **Evaluate** → Create dataset → Run evaluators
4. **Monitor** → Dashboard → Alerts
5. **Optimize** → Compare prompts → Cost/latency/quality

