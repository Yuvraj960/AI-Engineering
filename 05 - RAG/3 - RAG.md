# Chapter 3: Retrieval-Augmented Generation (RAG) — Complete Guide

## 3.1 What is RAG? The Fundamental Problem

### 3.1.1 Why LLMs Need Help

```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM LIMITATIONS                              │
├─────────────────────────────────────────────────────────────────┤
│  ❌ Knowledge Cutoff — Training data frozen at specific date   │
│  ❌ No Private Data — Can't access your docs, DB, APIs         │
│  ❌ Hallucination — Confidently generates false information    │
│  ❌ No Citations — Can't verify or attribute sources           │
└─────────────────────────────────────────────────────────────────┘
```

> **Analogy:** Asking ChatGPT "What is my mother's name?" — it cannot answer because it lacks **external knowledge access**.

### 3.1.2 RAG Solution: Retrieval + Generation

```
┌─────────────────────────────────────────────────────────────────┐
│                        RAG ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   USER QUERY                                                    │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐    ┌──────────────┐    ┌────────────────┐     │
│   │   EMBED     │───▶│  VECTOR DB   │───▶│  RETRIEVE      │     │
│   │   QUERY     │    │  (SEARCH)    │    │  TOP-K CHUNKS  │     │
│   └─────────────┘    └──────────────┘    └────────────────┘     │
│                                                      │          │
│                                                      ▼          │
│   ┌─────────────┐    ┌──────────────┐    ┌────────────────┐     │
│   │   LLM       │◀───│  PROMPT      │◀───│  CONTEXT       │     │
│   │  GENERATES  │    │  (QUERY +    │    │  (CHUNKS)      │     │
│   │  ANSWER     │    │   CHUNKS)    │    │                │     │
│   └─────────────┘    └──────────────┘    └────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two Key Components:**
1. **Retrieval Model** — Fetches relevant information from external knowledge source
2. **Language Model** — Generates responses based on retrieved knowledge

---

## 3.2 The Three-Phase RAG Pipeline

### Phase 1: INDEXING (Offline, One-time)

```
DOCUMENTS ──▶ CHUNKING ──▶ EMBEDDING ──▶ VECTOR STORE
```

```python
# From HuggingFace blog - Simple implementation
import ollama

EMBEDDING_MODEL = 'hf.co/CompendiumLabs/bge-base-en-v1.5-gguf'
LANGUAGE_MODEL = 'hf.co/bartowski/Llama-3.2-1B-Instruct-GGUF'

VECTOR_DB = []  # List of (chunk, embedding) tuples

def add_chunk_to_database(chunk):
    embedding = ollama.embed(model=EMBEDDING_MODEL, input=chunk)['embeddings'][0]
    VECTOR_DB.append((chunk, embedding))

# Load dataset (each line = chunk for simplicity)
with open('cat-facts.txt', 'r') as file:
    dataset = file.readlines()

for i, chunk in enumerate(dataset):
    add_chunk_to_database(chunk)
    print(f'Added chunk {i+1}/{len(dataset)}')
```

**Key Decisions in Indexing:**

| Decision | Options | Trade-offs |
|----------|---------|------------|
| **Chunk Size** | 100-4000 chars | Small = precise, Large = context |
| **Overlap** | 0-20% of chunk_size | Prevents context loss at boundaries |
| **Splitter** | Recursive, Semantic, Markdown, Code | Structure preservation vs speed |
| **Embedding Model** | bge, e5, text-embedding-3, all-MiniLM | Quality vs speed vs cost |

---

### Phase 2: RETRIEVAL (Online, Per Query)

```python
# Cosine similarity from scratch (HuggingFace blog)
def cosine_similarity(a, b):
    dot_product = sum(x * y for x, y in zip(a, b))
    norm_a = sum(x ** 2 for x in a) ** 0.5
    norm_b = sum(x ** 2 for x in b) ** 0.5
    return dot_product / (norm_a * norm_b)

def retrieve(query, top_n=3):
    query_embedding = ollama.embed(model=EMBEDDING_MODEL, input=query)['embeddings'][0]
    similarities = []
    for chunk, embedding in VECTOR_DB:
        similarity = cosine_similarity(query_embedding, embedding)
        similarities.append((chunk, similarity))
    similarities.sort(key=lambda x: x[1], reverse=True)
    return similarities[:top_n]
```

**Advanced Retrieval Strategies (from freeCodeCamp course):**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Similarity Search** | Cosine/Inner product | Basic semantic search |
| **MMR (Max Marginal Relevance)** | Balance relevance + diversity | Reduce redundancy |
| **Multi-Query** | Generate multiple query variations | Handle query ambiguity |
| **RAG Fusion** | Combine multiple retrievers (RRF) | Best of dense + sparse |
| **HyDE** | Generate hypothetical doc → search | Bridge query-doc gap |
| **Step-Back** | Abstract query → retrieve → answer | Complex reasoning |
| **Query Decomposition** | Break complex query → sub-queries | Multi-hop questions |
| **Self-Query** | NL → structured filter (metadata) | Filtered retrieval |

---

### Phase 3: GENERATION (Online, Per Query)

```python
# From HuggingFace blog
input_query = input('Ask me a question: ')
retrieved_knowledge = retrieve(input_query)

print('Retrieved knowledge:')
for chunk, similarity in retrieved_knowledge:
    print(f' - (similarity: {similarity:.2f}) {chunk}')

instruction_prompt = f'''You are a helpful chatbot.
Use only the following pieces of context to answer the question. Don't make up any new information:
{chr(10).join([f' - {chunk}' for chunk, similarity in retrieved_knowledge])}
'''

stream = ollama.chat(
    model=LANGUAGE_MODEL,
    messages=[
        {'role': 'system', 'content': instruction_prompt},
        {'role': 'user', 'content': input_query},
    ],
    stream=True,
)

print('Chatbot response:')
for chunk in stream:
    print(chunk['message']['content'], end='', flush=True)
```

---

## 3.3 LangChain Implementation (Modern LCEL)

### 3.3.1 Standard RAG Chain

```python
from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_core.output_parsers import StrOutputParser

# 1. LOAD & SPLIT
loader = DirectoryLoader("data/", glob="**/*.pdf", loader_cls=PyPDFLoader)
docs = loader.load()

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
chunks = splitter.split_documents(docs)

# 2. EMBED & STORE
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(chunks, embeddings, persist_directory="./chroma_db")

# 3. RETRIEVER WITH ADVANCED SEARCH
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 6, "fetch_k": 20, "lambda_mult": 0.5}
)

# 4. RERANKING (Cross-Encoder)
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

reranker = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
compressor = CrossEncoderReranker(model=reranker, top_n=4)

compressed_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever
)

# 4. FORMAT DOCUMENTS
def format_docs(docs):
    return "\n\n".join(f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}" for doc in docs)

# 5. PROMPT TEMPLATE
RAG_PROMPT = ChatPromptTemplate.from_template("""
Answer the question using ONLY the context below.
If the answer is not in the context, say "I don't know."
Cite sources using [1], [2] notation.

Context:
{context}

Question: {question}
""")

# 6. LCEL CHAIN
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

rag_chain = (
    RunnableParallel(
        context=compressed_retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | RAG_PROMPT
    | llm
    | StrOutputParser()
)

# 7. USE
result = rag_chain.invoke("What are the key findings?")
print(result)
```

---

### 3.3.2 Conversational RAG with History

```python
from langchain_core.runnables import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

conversational_rag = RunnableWithMessageHistory(
    rag_chain,
    lambda session_id: ChatMessageHistory(),
    input_messages_key="question",
    history_messages_key="history"
)

config = {"configurable": {"session_id": "user_123"}}
result = conversational_rag.invoke({"question": "What is RAG?"}, config=config)
result = conversational_rag.invoke({"question": "How does retrieval work?"}, config=config)
# Remembers previous context!
```

---

## 3.4 Advanced RAG Techniques (from freeCodeCamp Course)

### 3.4.1 Query Translation Strategies

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

# Multi-Query: Generate alternative queries
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
multi_query_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=llm
)
# Generates: "What is RAG?", "Explain RAG", "RAG definition" → searches each

# RAG Fusion (Reciprocal Rank Fusion)
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(chunks)
bm25.k = 10
dense = vectorstore.as_retriever(search_kwargs={"k": 10})

fusion_retriever = EnsembleRetriever(
    retrievers=[dense, bm25],
    weights=[0.6, 0.4]
)
```

### 3.4.2 HyDE (Hypothetical Document Embeddings)

```python
# Generate hypothetical answer → embed → search
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

hyde_prompt = ChatPromptTemplate.from_template("""
Write a passage that answers the question. Be detailed and specific.
Question: {question}
Passage:
""")

hyde_chain = hyde_prompt | llm | StrOutputParser()

# Use hypothetical passage for search
def hyde_search(query):
    hypothetical = hyde_chain.invoke({"question": query})
    return vectorstore.similarity_search(hypothetical, k=5)
```

### 3.4.3 Step-Back Prompting

```python
step_back_prompt = ChatPromptTemplate.from_template("""
You are an expert at abstracting questions. Generate a step-back question
that is more general but helps answer the original question.

Original: {question}
Step-back:
""")

step_back_chain = step_back_prompt | llm | StrOutputParser()

def step_back_retrieve(query):
    # 1. Generate step-back question
    general_query = step_back_chain.invoke({"question": query})
    # 2. Retrieve for both
    specific_docs = vectorstore.similarity_search(query, k=3)
    general_docs = vectorstore.similarity_search(general_query, k=3)
    # 3. Combine
    return specific_docs + general_docs
```

### 3.4.4 CRAG (Corrective RAG) & Adaptive RAG

```python
# CRAG: Check retrieval quality, rewrite query if poor
from langchain_core.prompts import ChatPromptTemplate

grader_prompt = ChatPromptTemplate.from_template("""
You are a grader assessing relevance of a retrieved document to a question.
Question: {question}
Document: {document}
Grade (yes/no):
""")

grader_chain = grader_prompt | llm | StrOutputParser()

def crag_retrieve(query):
    docs = vectorstore.similarity_search(query, k=5)
    relevant_docs = []
    for doc in docs:
        grade = grader_chain.invoke({"question": query, "document": doc.page_content})
        if "yes" in grade.lower():
            relevant_docs.append(doc)
    
    if len(relevant_docs) < 2:  # Low quality → rewrite query
        rewrite_prompt = ChatPromptTemplate.from_template("""
        Rewrite the question to be more specific for better retrieval.
        Original: {question}
        Rewritten:
        """)
        rewritten = (rewrite_prompt | llm | StrOutputParser()).invoke({"question": query})
        docs = vectorstore.similarity_search(rewritten, k=5)
    
    return relevant_docs or docs
```

---

## 3.5 RAG Evaluation (RAGAS)

```python
# Install: pip install ragas
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
    context_relevancy
)
from datasets import Dataset

# Prepare evaluation dataset
eval_data = {
    "question": ["What is RAG?", "How does retrieval work?"],
    "answer": ["RAG combines retrieval...", "Retrieval finds relevant docs..."],
    "contexts": [["RAG combines...", "Retrieval..."], ["Retrieval finds...", "Generation..."]],
    "ground_truth": ["RAG is a framework...", "Retrieval searches..."]
}

dataset = Dataset.from_dict(eval_data)

# Evaluate
results = evaluate(
    dataset,
    metrics=[
        faithfulness,           # Answer supported by context?
        answer_relevancy,       # Answer relevant to question?
        context_precision,      # Retrieved context precise?
        context_recall,         # Retrieved all needed context?
        context_relevancy       # Context relevant to question?
    ]
)

print(results)
# {'faithfulness': 0.85, 'answer_relevancy': 0.92, ...}
```

---

## 3.6 RAG Architectural Patterns

### 3.5.1 Modular RAG (Linear, Conditional, Branching, Looping)

```
LINEAR:          Query → Retrieve → Generate
CONDITIONAL:     Query → Classify → [Retrieve A | Retrieve B] → Generate
BRANCHING:       Query → Route → [Path 1 | Path 2 | Path 3] → Merge → Generate
LOOPING:         Query → Retrieve → Evaluate → [Loop/Exit] → Generate
```

### 3.5.2 Agentic RAG (LangGraph)

```python
from langgraph.graph import StateGraph, MessagesState
from langgraph.prebuilt import ToolNode, create_react_agent

# Tools for agentic RAG
from langchain_core.tools import tool

@tool
def search_knowledge_base(query: str) -> str:
    """Search the knowledge base for relevant information."""
    docs = vectorstore.similarity_search(query, k=4)
    return "\n\n".join(doc.page_content for doc in docs)

@tool
def search_web(query: str) -> str:
    """Search the web for up-to-date information."""
    # Implementation
    return results

# Agentic RAG with multiple tools
agentic_rag = create_react_agent(
    model=llm,
    tools=[search_knowledge_base, search_web],
    prompt="You are a research assistant. Use tools to find accurate answers."
)

# Usage
result = agentic_rag.invoke({"messages": [HumanMessage("Latest AI research on transformers?")]})
```

---

## 3.6 Embedding Models Deep Dive (DeepLearning.AI Courses)

### 3.6.1 Embedding Properties

| Property | Description | Application |
|----------|-------------|-------------|
| **Semantic Similarity** | Cosine similarity ≈ meaning similarity | Search, clustering |
| **Arithmetic** | king - man + woman ≈ queen | Analogies, bias detection |
| **Clustering** | Similar sentences cluster together | Topic modeling |
| **Outlier Detection** | Far from cluster = anomaly | Quality control |

### 3.6.2 Embedding Model Selection

| Model | Dimensions | Speed | Quality | Best For |
|-------|------------|-------|---------|----------|
| `text-embedding-3-large` | 3072 | Medium | Highest | Production RAG |
| `text-embedding-3-small` | 1536 | Fast | High | Cost-sensitive |
| `text-embedding-ada-002` | 1536 | Fast | Good | Legacy |
| `bge-large-en-v1.5` | 1024 | Medium | SOTA Open | Local/Private |
| `all-MiniLM-L6-v2` | 384 | Very Fast | Good | Real-time/Edge |
| `nomic-embed-text-v1.5` | 768 | Fast | Excellent | Long context (8k) |

---

## 3.7 Vector Database Selection

| Database | Type | Filtering | Scale | Best For |
|----------|------|-----------|-------|----------|
| **FAISS** | Local (RAM) | ❌ | ~1M | Prototyping |
| **Chroma** | Local/Server | ✅ Metadata | ~10M | Dev/Prod |
| **Pinecone** | Managed Cloud | ✅ | Unlimited | Zero-ops Prod |
| **Qdrant** | Cloud/Self-host | ✅ Payload | Large | Rust perf |
| **Weaviate** | Cloud/Self-host | ✅ Graph | Large | Hybrid search |
| **PGVector** | PostgreSQL | ✅ SQL | Large | Existing Postgres |
| **Milvus** | Distributed | ✅ | Billions | Massive scale |

---

## 3.8 RAG Production Checklist

### Data Pipeline
- [ ] **Document loading** — Handle all formats (PDF, HTML, MD, Code, etc.)
- [ ] **Chunking strategy** — RecursiveCharacterTextSplitter (1000/100) default
- [ ] **Metadata enrichment** — source, page, date, author, category
- [ ] **Embedding batching** — `embed_documents()` for efficiency
- [ ] **Incremental updates** — Add/delete without full rebuild

### Retrieval Quality
- [ ] **Hybrid search** — Dense + BM25 (EnsembleRetriever)
- [ ] **Reranking** — Cross-encoder (top-20 → top-5)
- [ ] **Query expansion** — Multi-query, HyDE, Step-back
- [ ] **Diversity** — MMR (lambda_mult=0.5)
- [ ] **Filtering** — Metadata filters at vector store level

### Generation Quality
- [ ] **Citation prompts** — "[1] Source: file.pdf, page 3"
- [ ] **Structured output** — Pydantic for consistent format
- [ ] **Hallucination guard** — "If not in context, say I don't know"
- [ ] **Token budgeting** — Fit context + query + answer in window

### Reliability & Observability
- [ ] **LangSmith tracing** — Full RAG chain visibility
- [ ] **RAGAS evaluation** — Faithfulness, relevancy, precision, recall
- [ ] **Latency monitoring** — p50, p95, p99 per component
- [ ] **Fallback retriever** — BM25 if dense fails
- [ ] **Circuit breakers** — Embedding/LLM API failures

### Security & Compliance
- [ ] **PII detection** — Filter sensitive data pre-embedding
- [ ] **Access control** — Metadata-based document permissions
- [ ] **Audit logging** — Query + retrieved docs + answer

---

## 3.9 Decision Flowchart

```
Need RAG?
│
├─► Simple Q&A over docs?
│   └─► Standard RAG
│
├─► Complex/ambiguous queries?
│   ├─► Multi-Query + RAG Fusion
│   └─► Query Decomposition
│
├─► Need up-to-date info?
│   └─► Agentic RAG (Web search tool)
│
├─► Low retrieval quality?
│   ├─► HyDE + Reranking
│   └─► CRAG (adaptive rewrite)
│
├─► Metadata filtering needed?
│   └─► SelfQueryRetriever + Chroma/Pinecone
│
├─► Conversational?
│   └─► RunnableWithMessageHistory + Conversational RAG
│
├─► High precision required?
│   └─► Cross-Encoder Reranker + ContextualCompression
│
└─► Massive scale / Multi-tenant?
    └─► Pinecone/Weaviate + Modular RAG + LangGraph
```

---

## 3.10 Complete RAG Cheat Sheet

```python
# THE PRODUCTION RAG STACK

# 1. LOAD & SPLIT
from langchain_community.document_loaders import DirectoryLoader, PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

loader = DirectoryLoader("data/", glob="**/*.pdf", loader_cls=PyPDFLoader)
docs = loader.load()

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""]
)
chunks = splitter.split_documents(docs)

# 2. EMBED & STORE (Chroma + filtering)
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(
    chunks, embeddings, 
    persist_directory="./chroma_db",
    collection_name="rag_docs"
)

# 3. ADVANCED RETRIEVAL PIPELINE
from langchain.retrievers import ContextualCompressionRetriever, MultiQueryRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever

# Base retrievers
dense = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 10, "fetch_k": 30})
bm25 = BM25Retriever.from_documents(chunks)
bm25.k = 10

# Hybrid fusion
hybrid = EnsembleRetriever(retrievers=[dense, bm25], weights=[0.6, 0.4])

# Multi-query expansion
multi = MultiQueryRetriever.from_llm(
    retriever=hybrid,
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0)
)

# Cross-encoder reranking
reranker = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
compressor = CrossEncoderReranker(model=reranker, top_n=5)

retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=multi
)

# 4. GENERATION WITH CITATIONS
def format_docs(docs):
    return "\n\n".join(
        f"[Source {i+1}: {doc.metadata.get('source', '?')}, "
        f"page {doc.metadata.get('page', '?')}]\n{doc.page_content}"
        for i, doc in enumerate(docs)
    )

RAG_PROMPT = ChatPromptTemplate.from_template("""
Answer using ONLY the context below. Cite sources with [1], [2].

Context: {context}

Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | RAG_PROMPT
    | llm
    | StrOutputParser()
)

# 5. CONVERSATIONAL
from langchain_core.runnables import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

conversational = RunnableWithMessageHistory(
    rag_chain,
    lambda sid: ChatMessageHistory(),
    input_messages_key="question",
    history_messages_key="history"
)

# USE
result = conversational.invoke(
    {"question": "What is the main finding?"},
    config={"configurable": {"session_id": "user_123"}}
)
```

---

## 3.11 Resources & References

### Core Papers
- **Original RAG:** Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020) — https://arxiv.org/abs/2005.11401
- **RAG Fusion:** "RAG-Fusion: Improving RAG with Reciprocal Rank Fusion" — https://arxiv.org/html/2408.04948v1
- **Modular RAG:** "Modular RAG: Transforming RAG into a Flexible Framework" — https://arxiv.org/html/2407.21059v1
- **CRAG:** "Corrective Retrieval Augmented Generation" — https://arxiv.org/abs/2401.15884
- **Self-RAG:** "Self-RAG: Learning to Retrieve, Generate, and Critique" — https://arxiv.org/abs/2310.11511

### Courses & Tutorials
- **HuggingFace Blog:** https://huggingface.co/blog/ngxson/make-your-own-rag
- **freeCodeCamp (Lance Martin):** https://www.freecodecamp.org/news/mastering-rag-from-scratch/
- **DeepLearning.AI Vector DBs:** https://learn.deeplearning.ai/courses/vector-databases-embeddings-applications/
- **DeepLearning.AI Vertex AI Embeddings:** https://learn.deeplearning.ai/courses/google-cloud-vertex-ai/
- **CampusX RAG Video:** https://youtu.be/X0btK9X0Xnk

### Evaluation
- **RAGAS:** https://github.com/explodinggradients/ragas
- **RAG Evaluation Guide:** https://www.pinecone.io/learn/series/rag/rerankers/
