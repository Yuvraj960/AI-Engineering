# LAYER 0 — Software Engineering Foundation

This is the layer people dangerously underestimate.

Before AI engineering, you must already be a good software engineer.

### Programming

Primary:

**Python**

Secondary:

* TypeScript/JavaScript
* Java or C++

Learn:

* functions
* classes
* OOP
* typing
* dataclasses
* decorators
* generators
* iterators
* context managers
* async/await
* multiprocessing
* threading
* error handling
* package management
* virtual environments
* dependency management
* logging
* configuration
* testing

Then:

```text
Python
→ NumPy
→ Pandas
→ PyTorch
→ FastAPI
→ Pydantic
→ pytest
```

### Software engineering

Learn properly:

* Git
* GitHub
* Git branching
* pull requests
* code review
* REST
* APIs
* authentication
* authorization
* SQL
* PostgreSQL
* Redis
* caching
* queues
* WebSockets
* Docker
* Linux
* CI/CD

Then system design:

* load balancing
* horizontal scaling
* distributed systems
* event-driven architecture
* message brokers
* databases
* object storage
* observability

---

# LAYER 1 — Mathematics + Machine Learning

## Mathematics

### Linear Algebra

Know:

* vectors
* matrices
* matrix multiplication
* dot product
* norms
* eigenvalues/eigenvectors
* projections
* orthogonality
* SVD
* tensor operations

### Calculus

Know:

* derivatives
* partial derivatives
* gradients
* chain rule
* Jacobian
* optimization

### Probability

Know:

* random variables
* distributions
* expectation
* variance
* covariance
* conditional probability
* Bayes theorem
* likelihood
* maximum likelihood
* entropy
* cross entropy

### Statistics

Know:

* sampling
* hypothesis testing
* confidence intervals
* correlation
* regression
* bias/variance
* statistical significance
* A/B testing

---

# LAYER 2 — Classical Machine Learning

### Supervised learning

* linear regression
* logistic regression
* Naive Bayes
* decision trees
* random forests
* SVM
* gradient boosting
* XGBoost

### Unsupervised

* K-means
* GMM
* PCA
* dimensionality reduction

### Model evaluation

* accuracy
* precision
* recall
* F1
* ROC-AUC
* PR-AUC
* confusion matrix
* calibration
* cross-validation

### ML engineering

* preprocessing
* feature engineering
* data leakage
* class imbalance
* hyperparameter tuning
* model selection
* error analysis

Stanford's CS229 continues to provide almost exactly this foundation. ([CS229 Machine Learning][12])

---

# LAYER 3 — Deep Learning

### Neural networks

* perceptron
* MLP
* activation functions
* loss functions
* forward propagation
* backpropagation
* gradient descent
* SGD
* Adam
* regularization
* dropout
* batch normalization

### Architectures

Understand:

* CNN
* RNN
* LSTM
* GRU
* encoder-decoder

Then arrive at:

# Transformers

This is the gateway into modern AI.

We need to understand:

* tokenization
* embeddings
* positional encoding
* self-attention
* multi-head attention
* Q/K/V
* encoder
* decoder
* causal masking
* cross-attention
* autoregressive generation

---

# LAYER 4 — LLM Engineering

Now the curriculum becomes much more specifically AI Engineering.

## LLM architecture

* tokenizer
* vocabulary
* embeddings
* transformer blocks
* attention
* FFN
* normalization
* residual connections
* context window
* logits
* sampling

### Generation

Understand:

* temperature
* top-k
* top-p
* repetition penalties
* greedy decoding
* beam search
* stop tokens

### LLM lifecycle

```text
Dataset
   ↓
Pretraining
   ↓
Instruction tuning
   ↓
Preference optimization
   ↓
Evaluation
   ↓
Inference
   ↓
Application
```

Study:

* pretraining
* instruction tuning
* SFT
* RLHF
* DPO
* PEFT
* LoRA
* QLoRA

### Hugging Face ecosystem

You should know:

* Transformers
* Datasets
* Tokenizers
* Hub
* PEFT
* TRL
* Accelerate

---

# LAYER 5 — LLM Application Engineering

This is where **AI Engineering really begins**.

Learn model APIs:

* OpenAI
* Anthropic
* Google
* open-source models
* local models

Don't become vendor-dependent.

Your application architecture should allow:

```text
Application
     ↓
Model abstraction
     ↓
OpenAI / Anthropic / Gemini / local model
```

Learn:

* structured output
* JSON schemas
* function/tool calling
* streaming
* retries
* rate limiting
* fallback models
* timeouts
* token budgeting
* context management
* prompt versioning

### Prompt engineering

Learn:

* role/system instructions
* few-shot prompting
* decomposition
* structured prompting
* constraints
* output schemas
* prompt templates
* prompt testing
* prompt injection defenses

The goal is not to become a **Prompt Engineer**.
The goal is to become an engineer who understands how prompts influence a system.

---

# LAYER 6 — RAG

This is absolutely mandatory.

You should eventually be able to implement RAG **without LangChain first**.

Understand:

```text
Documents
    ↓
Parsing
    ↓
Chunking
    ↓
Embedding
    ↓
Vector index
    ↓
Retrieval
    ↓
Reranking
    ↓
Context construction
    ↓
LLM
    ↓
Answer
```

Learn:

### Document ingestion

* PDF
* HTML
* Markdown
* DOCX
* CSV
* JSON
* database sources

### Chunking

* fixed-size
* recursive
* semantic
* sentence
* hierarchical

### Embeddings

Understand:

* embedding spaces
* cosine similarity
* dot product
* dimensionality
* embedding model selection

### Retrieval

* vector search
* keyword search
* BM25
* hybrid search
* metadata filtering

### Advanced retrieval

* reranking
* query rewriting
* multi-query retrieval
* HyDE
* parent-child retrieval
* sentence-window retrieval
* auto-merging
* contextual retrieval

DeepLearning.AI's current RAG curriculum explicitly covers semantic search, BM25, Reciprocal Rank Fusion, vector databases and production-oriented RAG implementation, while its advanced RAG course emphasizes sentence-window retrieval, auto-merging and evaluation via context relevance, groundedness and answer relevance. ([DeepLearning.ai][13])

### Vector databases

Know at least two:

* pgvector
* Qdrant
* Weaviate
* Pinecone
* Milvus

I would personally learn:

**PostgreSQL + pgvector + Qdrant**

because it gives you both a conventional relational foundation and dedicated vector infrastructure.

---

# LAYER 7 — AI Agents

This is becoming one of the defining areas of AI Engineering.

But here is the crucial distinction:

**Do not learn “LangGraph” first.**

Learn agent mechanics first.

Andrew Ng's approach is particularly valuable here: first understand agentic design patterns in raw Python and only then use frameworks. ([DeepLearning.ai][8])

Understand:

```text
LLM
 ↓
Reason/decide
 ↓
Tool
 ↓
Observation
 ↓
Reason
 ↓
Tool
 ↓
Result
```

### Agent fundamentals

Learn:

* tools
* tool schemas
* function calling
* state
* memory
* planning
* routing
* reflection
* observation
* retries
* error handling
* human-in-the-loop
* autonomy levels

### Agent patterns

Master:

#### 1. Tool use

LLM → API/database/tool

#### 2. Routing

```text
User request
      ↓
Classifier/router
   ↙     ↓      ↘
SQL   RAG    Web Search
```

#### 3. Reflection

```text
Generate
   ↓
Critique
   ↓
Improve
```

#### 4. Planning

```text
Goal
 ↓
Plan
 ↓
Execute step 1
 ↓
Execute step 2
 ↓
Verify
```

#### 5. Multi-agent

```text
Manager
 ├── Researcher
 ├── Analyst
 ├── Coder
 └── Reviewer
```

Andrew Ng's current Agentic AI curriculum explicitly covers reflection, tool use, planning and multi-agent workflows, plus external tools, evaluations and production optimization. ([DeepLearning.ai][8])

Hugging Face's current curriculum independently converges on the same concepts and then maps them onto smolagents, LlamaIndex and LangGraph. ([Hugging Face][10])

That convergence is significant.

---

# LAYER 8 — MCP + AI Context Infrastructure

This is one of the newer pieces you should learn.

MCP = **Model Context Protocol**.

You should understand:

* MCP client
* MCP server
* tools
* resources
* prompts
* transports
* authentication
* permissions
* server lifecycle

Build MCP servers exposing:

```text
PostgreSQL
GitHub
filesystem
REST APIs
internal company APIs
```

This isn't theoretical anymore.

DXC's current GenAI Engineer role specifically asks engineers to build MCP servers in Python/TypeScript to expose internal databases, APIs and filesystems to agents. ([dxc][3])

DataCamp also added MCP to its AI Engineer for Developers curriculum in its July 2026 update. ([DataCamp][11])

So I consider MCP a **core modern AI-engineering topic**, not a nice-to-have.

---

# LAYER 9 — Evaluation, LLMOps and Reliability

This is where many “AI developers” fail.

They build:

> Chatbot works!

and stop.

Industry asks:

> How do you know it works?

This is a huge difference.

Learn:

### Evaluation

Traditional:

* exact match
* precision/recall
* accuracy

LLM-specific:

* faithfulness
* groundedness
* relevance
* coherence
* helpfulness
* toxicity
* hallucination
* tool-call accuracy

Agent-specific:

* task completion
* trajectory correctness
* tool selection
* execution success
* recovery
* latency
* cost

### Build an evaluation dataset

For example:

```text
100 representative queries
+
expected answers / expected behavior
+
ground-truth documents
+
failure labels
```

Then measure:

```text
RAG Recall
RAG Precision
Answer Relevance
Faithfulness
Task Success
Latency
Token Cost
```

This is explicitly reflected in current AI-engineering roles and educational material. DXC calls out correctness, latency, token optimization, hallucinations and bias. OpenAI's agent roles emphasize evaluations, reliability, observability and cost/latency efficiency. ([dxc][3])

### Observability

Learn:

* tracing
* spans
* logs
* metrics
* token usage
* latency
* model calls
* tool calls
* prompt versions
* failure traces

Learn:

* OpenTelemetry
* LangSmith
* Arize/Phoenix
* W&B
* MLflow

You do not need every observability product.

Understand the concepts.

---

# LAYER 10 — Production AI Infrastructure

This is what can turn your profile from:

**“GenAI developer”**

into:

**“AI Engineer.”**

Learn:

### Backend

* FastAPI
* Pydantic
* async Python
* PostgreSQL
* Redis
* Celery/RQ
* Kafka

### Deployment

* Docker
* Kubernetes
* NGINX
* CI/CD
* GitHub Actions

### Cloud

Learn at least one deeply:

**AWS**

and understand equivalent services in Azure/GCP.

For example:

```text
API
↓
Load Balancer
↓
FastAPI
↓
Agent service
↓
Redis
↓
PostgreSQL
↓
Vector DB
↓
LLM API
```

### Model serving

Understand:

* batching
* caching
* quantization
* GPU memory
* inference latency
* throughput
* concurrency
* vLLM
* TGI
* llama.cpp
* Ollama

### Scaling

You should understand:

```text
1 user
→ 100 users
→ 10,000 users
→ 1,000,000 requests
```

and what breaks at each level.

---

# AI Security is now part of the curriculum

This is becoming unavoidable.

Study:

### Prompt injection

```text
User input
      ↓
Malicious instructions
      ↓
Agent
      ↓
Tool misuse
```

### Threats

* prompt injection
* indirect prompt injection
* data exfiltration
* tool abuse
* excessive agency
* insecure tool permissions
* sensitive data disclosure
* model supply-chain attacks
* jailbreaks

### Security architecture

Learn:

* sandboxing
* authentication
* authorization
* least privilege
* secrets management
* tenant isolation
* audit logs
* human approvals
* secure execution

OpenAI's current agent infrastructure roles explicitly emphasize secure execution environments, identity, permissions, observability and safe production operation. ([OpenAI][2])

---