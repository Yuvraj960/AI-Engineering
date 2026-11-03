# Chapter 2: Vector Stores & Retrievers — The Search Engine of RAG

## 2.1 The RAG Pipeline — Where Vector Stores & Retrievers Fit

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐    ┌──────────────┐    ┌────────────────┐
│  Document   │───▶│   Text       │───▶│  Embedding  │───▶│   Vector       │───▶│   Retriever  │───▶│   LLM +        │
│  Loaders    │    │  Splitters   │    │  Models     │    │   Store        │    │   (Search)   │    │   Prompt       │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘    └──────────────┘    └────────────────┘
```

**Vector Store** = Database optimized for **vector similarity search** (stores embeddings + metadata)
**Retriever** = Interface that **fetches relevant documents** given a query (abstraction over search logic)

---

## 2.2 Why Vector Stores? (The Movie Recommendation Analogy)

### The Problem: Traditional Databases Can't Do Semantic Search

> **Example:** Building an IMDb-like movie catalog

```sql
-- Traditional SQL: Exact match only
SELECT * FROM movies WHERE genre = 'Action' AND year > 2020;
-- Can't do: "Movies like Spider-Man" or "Sci-fi with emotional depth"
```

**Vector Stores solve this:**
- Store **embeddings** (vectors) instead of raw text
- Enable **semantic similarity search** (cosine similarity, dot product)
- "Movies like Spider-Man" → Embed query → Find nearest vectors

### Core Concept: Embeddings as Coordinates

```
Text: "Spider-Man swings through New York"
         │
         ▼ Embedding Model (text-embedding-3-large)
         │
         ▼
Vector: [0.12, -0.45, 0.78, 0.23, -0.11, ...]  (3072 dimensions)
         │
         ▼ Stored in Vector DB with metadata
         │
         ▼
Query: "Web-slinger NYC adventures"
         │
         ▼ Same Embedding Model
         │
         ▼
Query Vector: [0.11, -0.44, 0.79, 0.22, -0.10, ...]
         │
         ▼ COSINE SIMILARITY ≈ 0.98 → MATCH!
```

---

## 2.3 Vector Store Types & Comparison

| Vector Store | Type | Best For | Persistence | Scale | LangChain Package |
|--------------|------|----------|-------------|-------|-------------------|
| **FAISS** | In-memory (local) | Prototyping, small-medium data, offline | File-based (pickle) | ~1M vectors | `langchain-community` |
| **Chroma** | Local file / Server | Development, production, filtering | SQLite/Parquet | ~10M vectors | `langchain-chroma` |
| **Pinecone** | Managed cloud | Production, high scale, managed | Cloud | Unlimited | `langchain-pinecone` |
| **Weaviate** | Cloud / Self-hosted | Graph + vector, hybrid search | Cloud/Docker | Large | `langchain-weaviate` |
| **Qdrant** | Cloud / Self-hosted | Filtering, payload, Rust performance | Cloud/Docker | Large | `langchain-qdrant` |
| **Milvus** | Distributed | Massive scale, Kubernetes | Cluster | Billions | `langchain-milvus` |
| **PGVector** | PostgreSQL extension | SQL + vector, existing Postgres | PostgreSQL | Large | `langchain-postgres` |
| **Elasticsearch** | Search engine | Hybrid (BM25 + vector), logs | Cluster | Large | `langchain-elasticsearch` |

### Quick Selection Guide

| Scenario | Recommended |
|----------|-------------|
| **Local dev, < 100K docs** | FAISS or Chroma |
| **Production with metadata filtering** | Chroma, Qdrant, or PGVector |
| **Managed, zero-ops, auto-scaling** | Pinecone |
| **Existing PostgreSQL** | PGVector |
| **Hybrid BM25 + Vector** | Elasticsearch, Weaviate |
| **Billion-scale, custom infra** | Milvus |

---

## 2.4 Core Vector Store Operations

All vector stores implement the same interface:

```python
from langchain_core.vectorstores import VectorStore

# Create from documents (loads → splits → embeds → stores)
vectorstore = VectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    # store-specific kwargs
)

# Add documents later
vectorstore.add_documents(new_chunks)

# Similarity search (returns List[Document] with scores)
results = vectorstore.similarity_search(query, k=4)
# [(doc1, 0.85), (doc2, 0.72), ...]

# Similarity search with scores
results = vectorstore.similarity_search_with_score(query, k=4)

# Max Marginal Relevance (diverse results)
results = vectorstore.max_marginal_relevance_search(query, k=4, fetch_k=20)

# As retriever (for LCEL chains)
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# Persistence
vectorstore.save_local("path/")  # FAISS
vectorstore.persist()            # Chroma
```

---

## 2.5 Major Vector Stores — Code Examples

### 2.5.1 FAISS — Facebook AI Similarity Search (Local, Fast)

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 1. Load & Split
loader = PyPDFLoader("document.pdf")
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
chunks = splitter.split_documents(docs)

# 2. Create Embeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 3. Create Vector Store
vectorstore = FAISS.from_documents(chunks, embeddings)

# 4. Search
results = vectorstore.similarity_search("What is the main topic?", k=3)
for doc, score in results:
    print(f"Score: {score:.3f} | Content: {doc.page_content[:100]}...")

# 5. Persist to Disk
vectorstore.save_local("faiss_index")

# 6. Load from Disk (CRITICAL: allow_dangerous_deserialization=True)
loaded_vs = FAISS.load_local(
    "faiss_index", 
    embeddings, 
    allow_dangerous_deserialization=True
)
```

**FAISS Notes:**
- ⚡ Extremely fast (C++ backend, CPU optimized)
- 💾 Saves as `.faiss` (index) + `.pkl` (docstore)
- ⚠️ `allow_dangerous_deserialization=True` required for loading (pickle security)
- 🔧 Supports `IndexFlatIP` (inner product) or `IndexFlatL2` (Euclidean)

---

### 2.5.2 Chroma — Persistent, Filterable, Production-Ready

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Create / Connect (persists automatically to ./chroma_db)
vectorstore = Chroma(
    collection_name="my_docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

# Add documents
vectorstore.add_documents(chunks)

# Search with metadata filtering!
results = vectorstore.similarity_search(
    "machine learning",
    k=5,
    filter={"source": "research_paper.pdf"}  # Chroma native filtering
)

# Advanced: Where clauses
from langchain_chroma import Chroma
results = vectorstore.similarity_search(
    query="transformer architecture",
    k=4,
    filter={
        "$and": [
            {"year": {"$gte": 2020}},
            {"category": {"$in": ["NLP", "CV"]}}
        ]
    }
)

# Persistence is automatic, but can force:
vectorstore.persist()
```

**Chroma Advantages:**
- ✅ Native metadata filtering (`filter` parameter)
- ✅ Automatic persistence (SQLite backend)
- ✅ Collections for multi-tenancy
- ✅ HTTP server mode for shared access

---

### 2.5.3 Pinecone — Managed Cloud (Production Scale)

```python
# pip install langchain-pinecone pinecone
from langchain_pinecone import PineconeVectorStore
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="your-api-key")

# Create index (one-time)
index_name = "my-rag-index"
if index_name not in pc.list_indexes().names():
    pc.create_index(
        name=index_name,
        dimension=1536,  # Must match embedding dimension!
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1")
    )

# Connect
index = pc.Index(index_name)
vectorstore = PineconeVectorStore(
    index=index,
    embedding=embeddings,
    text_key="text"  # Field name for document content
)

# Add (batch for efficiency)
vectorstore.add_documents(chunks, batch_size=100)

# Search with filter
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"category": "technical", "year": {"$gte": 2023}}
)

# Pinecone also supports sparse vectors for hybrid search
```

---

### 2.5.4 PGVector — PostgreSQL + Vectors

```python
# pip install langchain-postgres pgvector
from langchain_postgres import PGVector
from sqlalchemy import create_engine

CONNECTION_STRING = "postgresql+psycopg://user:pass@localhost:5432/db"

# Create vectorstore (auto-creates table)
vectorstore = PGVector(
    embeddings=embeddings,
    collection_name="my_docs",
    connection=CONNECTION_STRING,
    use_jsonb=True,  # Store metadata as JSONB
)

# Add documents
vectorstore.add_documents(chunks)

# Search with full SQL power!
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"source": "report.pdf"}  # JSONB querying
)

# Hybrid: Combine with SQL
from sqlalchemy import text
with engine.connect() as conn:
    result = conn.execute(text("""
        SELECT content, metadata, 1 - (embedding <=> :query_vec) as similarity
        FROM langchain_pg_embedding
        WHERE metadata->>'source' = 'report.pdf'
        ORDER BY embedding <=> :query_vec
        LIMIT 5
    """), {"query_vec": embeddings.embed_query("query")})
```

---

## 2.6 Retrievers — The Search Abstraction

> **Key Insight:** VectorStore = Storage. Retriever = Search Interface.

```python
# VectorStore has .as_retriever() method
retriever = vectorstore.as_retriever(
    search_type="similarity",        # "similarity", "mmr", "similarity_score_threshold"
    search_kwargs={"k": 4}           # k, fetch_k, lambda_mult, score_threshold
)

# Retriever is a Runnable!
result = retriever.invoke("What is RAG?")
# Returns List[Document]

# Async
result = await retriever.ainvoke("What is RAG?")

# Batch
results = retriever.batch(["query1", "query2"])

# Streaming (if supported)
async for doc in retriever.astream("query"):
    print(doc.page_content)
```

---

## 2.7 Retriever Types — Deep Dive

### 2.7.1 VectorStoreRetriever — The Standard

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

vectorstore = Chroma(embedding_function=embeddings, persist_directory="./db")

# Basic similarity (default)
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4}
)

# With score threshold (filter low relevance)
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={"k": 10, "score_threshold": 0.7}
)
```

---

### 2.7.2 Maximum Marginal Relevance (MMR) — Diverse Results

> **Problem:** Similarity search returns redundant near-duplicates.
> **Solution:** MMR balances relevance vs diversity.

```python
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 4,              # Final results
        "fetch_k": 20,       # Candidate pool (larger = better diversity)
        "lambda_mult": 0.5   # 0 = max diversity, 1 = max relevance
    }
)

# How MMR works:
# Score = lambda * relevance - (1 - lambda) * max_similarity_to_selected
# Iteratively picks docs that are relevant BUT different from already selected
```

**When to Use MMR:**
- ✅ Document summarization (need diverse coverage)
- ✅ Long documents with repetitive sections
- ✅ When user wants "broad overview"

---

### 2.7.3 MultiQueryRetriever — Query Expansion

> **Problem:** Single query may miss relevant docs due to phrasing.
> **Solution:** Generate multiple query variations, search each, combine.

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=llm,
    prompt=None  # Uses default: "Generate 3 alternative queries..."
)

# Or with custom prompt
from langchain_core.prompts import PromptTemplate
custom_prompt = PromptTemplate(
    input_variables=["question"],
    template="Generate 5 alternative phrasings for this question to improve search:\n{question}\n\nAlternatives:"
)

retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=llm,
    prompt=custom_prompt
)

# Result: Queries each variant, unions results, deduplicates
result = retriever.invoke("How does attention work in transformers?")
# Internally searches: "attention mechanism transformers", "self-attention explanation", etc.
```

---

### 2.7.4 ContextualCompressionRetriever — Filter & Compress

> **Problem:** Retrieved chunks may be long with irrelevant parts.
> **Solution:** Use LLM to extract only relevant segments.

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# LLM-based compressor: "Given query and doc, extract only relevant sentences"
compressor = LLMChainExtractor.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10})
)

# Result: Shorter, more focused documents
result = compression_retriever.invoke("What is the capital of France?")
# Returns compressed chunks, not full original docs
```

**Alternative Compressors:**
```python
# 1. Embeddings-based (faster, no LLM call)
from langchain.retrievers.document_compressors import EmbeddingsFilter
compressor = EmbeddingsFilter(
    embeddings=embeddings,
    similarity_threshold=0.76
)

# 2. Combined pipeline
from langchain.retrievers.document_compressors import DocumentCompressorPipeline
from langchain.text_splitter import CharacterTextSplitter

pipeline = DocumentCompressorPipeline(
    transformers=[
        CharacterTextSplitter(chunk_size=300, chunk_overlap=0),
        EmbeddingsFilter(embeddings=embeddings, similarity_threshold=0.76)
    ]
)
compressor = pipeline
```

---

### 2.7.5 Other Retrievers

| Retriever | Use Case | Package |
|-----------|----------|---------|
| **WikipediaRetriever** | Search Wikipedia | `langchain_community.retrievers` |
| **PubMedRetriever** | Biomedical literature | `langchain_community.retrievers` |
| **ArxivRetriever** | Physics/CS papers | `langchain_community.retrievers` |
| **GoogleSearchAPIRetriever** | Web search | `langchain_google_community` |
| **BingSearchRetriever** | Web search | `langchain_community.retrievers` |
| **SelfQueryRetriever** | Natural language → structured filter | `langchain.retrievers` |
| **EnsembleRetriever** | Combine multiple retrievers (RRF) | `langchain.retrievers` |
| **ParentDocumentRetriever** | Small chunks → parent doc mapping | `langchain.retrievers` |
| **MergerRetriever** | Merge multiple retrievers | `langchain.retrievers` |

---

### 2.7.6 SelfQueryRetriever — NL to Structured Filter

```python
from langchain.retrievers.self_query import SelfQueryRetriever
from langchain_openai import ChatOpenAI
from langchain_chroma import Chroma
from langchain_core.prompts import PromptTemplate

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Define metadata schema
metadata_field_info = [
    {"name": "source", "description": "Source file", "type": "string"},
    {"name": "page", "description": "Page number", "type": "integer"},
    {"name": "year", "description": "Publication year", "type": "integer"},
    {"name": "category", "description": "Topic category", "type": "string"},
]

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents="Research papers on machine learning",
    metadata_field_info=metadata_field_info,
    verbose=True
)

# User asks in natural language:
result = retriever.invoke("Papers about transformers from 2023")
# LLM converts to: filter={"year": 2023, "category": "transformers"}
```

---

### 2.7.7 EnsembleRetriever — Reciprocal Rank Fusion

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_chroma import Chroma

# Dense (semantic) retriever
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# Sparse (keyword) retriever
bm25_retriever = BM25Retriever.from_documents(chunks)
bm25_retriever.k = 10

# Combine with weights (RRF fusion)
ensemble_retriever = EnsembleRetriever(
    retrievers=[dense_retriever, bm25_retriever],
    weights=[0.7, 0.3]  # Dense more important
)

result = ensemble_retriever.invoke("transformer attention mechanism")
# Combines semantic + keyword matching
```

---

## 2.8 Advanced Retrieval Patterns

### 2.8.1 Parent Document Retriever (Small Chunk → Large Context)

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain.storage import InMemoryStore

# Small chunks for retrieval
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)
# Large chunks for context
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

vectorstore = Chroma(embedding_function=embeddings)
store = InMemoryStore()  # Or Redis, FileSystemStore

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(docs)

# Retrieves small chunks, but RETURNS parent documents!
result = retriever.invoke("query")
# Returns 2000-char parent docs that contain the matching 400-char chunks
```

---

### 2.8.2 Hybrid Search (Dense + Sparse)

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_chroma import Chroma

# Dense (vector) - understands meaning
dense = Chroma(...).as_retriever(search_kwargs={"k": 10})

# Sparse (BM25) - exact keyword matching
bm25 = BM25Retriever.from_documents(chunks)
bm25.k = 10

# Combine
hybrid = EnsembleRetriever(
    retrievers=[dense, bm25],
    weights=[0.6, 0.4]
)

# Best of both worlds:
# - "transformer architecture" → dense wins
# - "BERT-base-uncased" → BM25 wins (exact term)
```

---

### 2.8.3 Reranking (Cross-Encoder)

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# Cross-encoder: query + doc → relevance score (more accurate than bi-encoder)
reranker = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")

compressor = CrossEncoderReranker(model=reranker, top_n=5)

reranked_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)

# 1. Vector search gets 20 candidates (fast)
# 2. Cross-encoder reranks top 20 → returns best 5 (accurate)
result = reranked_retriever.invoke("query")
```

---

## 2.9 Integration with LCEL

### 2.9.1 Standard RAG Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

RAG_PROMPT = ChatPromptTemplate.from_template("""
Answer the question using only the context below.
If the answer is not in the context, say "I don't know."

Context: {context}

Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini")

rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | RAG_PROMPT
    | llm
    | StrOutputParser()
)

# Invoke
result = rag_chain.invoke("What is the capital of France?")

# Stream
for chunk in rag_chain.stream("What is RAG?"):
    print(chunk, end="", flush=True)
```

---

### 2.9.2 RAG with Citations

```python
def format_docs_with_sources(docs):
    return "\n\n".join(
        f"[Source {i+1}: {doc.metadata.get('source', 'unknown')}, "
        f"page {doc.metadata.get('page', 'N/A')}]\n{doc.page_content}"
        for i, doc in enumerate(docs)
    )

CITATION_PROMPT = ChatPromptTemplate.from_template("""
Answer using ONLY the context below. Cite sources using [1], [2] notation.

Context:
{context}

Question: {question}
""")

citation_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs_with_sources),
        question=RunnablePassthrough()
    )
    | CITATION_PROMPT
    | llm
    | StrOutputParser()
)

# Output: "The capital is Paris [1]. [1] Source: france.pdf, page 3..."
```

---

### 2.9.3 Conversational RAG (with History)

```python
from langchain_core.runnables import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

# Wrap RAG chain with history
conversational_rag = RunnableWithMessageHistory(
    rag_chain,
    lambda session_id: ChatMessageHistory(),
    input_messages_key="question",
    history_messages_key="history"
)

# Use
config = {"configurable": {"session_id": "user_123"}}
result = conversational_rag.invoke({"question": "What is RAG?"}, config=config)
result = conversational_rag.invoke({"question": "How does it differ from fine-tuning?"}, config=config)
# Second question remembers first!
```

---

## 2.10 Persistence & Production Patterns

### 2.10.1 FAISS Persistence

```python
# Save
vectorstore.save_local("faiss_index")

# Load (MUST set allow_dangerous_deserialization=True)
vectorstore = FAISS.load_local(
    "faiss_index",
    embeddings,
    allow_dangerous_deserialization=True  # ⚠️ Security: only load trusted files
)

# For production: serialize safely
import pickle
# Save documents separately
with open("docs.pkl", "wb") as f:
    pickle.dump(docs, f)
# Save index only
vectorstore.index.save("faiss_index.faiss")
```

---

### 2.10.2 Chroma Persistence (Automatic)

```python
# Just specify persist_directory - auto-saves!
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings
)

# Force flush
vectorstore.persist()

# Reset collection
vectorstore.delete_collection()
```

---

### 2.10.3 Production Checklist

| Area | Checklist |
|------|-----------|
| **Indexing** | [ ] Batch embeddings (`embeddings.embed_documents()`) |
| | [ ] Use `add_documents` in batches |
| | [ ] Monitor index build time |
| **Search** | [ ] Choose right `k` (4-10 typical) |
| | [ ] Use MMR for diverse results |
| | [ ] Add score threshold for quality |
| **Filtering** | [ ] Index metadata for common filters |
| | [ ] Use native vector store filtering |
| **Reranking** | [ ] Add cross-encoder for precision |
| | [ ] Cache cross-encoder results |
| **Scale** | [ ] Monitor query latency (p50, p99) |
| | [ ] Load test with concurrent queries |
| | [ ] Consider read replicas (Pinecone, Weaviate) |
| **Reliability** | [ ] Health check endpoint |
| | [ ] Fallback retriever (BM25) |
| | [ ] Circuit breaker for embedding API |

---

## 2.11 Content Summary

### Vector Stores — Key Points
| Topic | Covered |
|-------|---------|
| **Why Vector Stores** | Movie recommendation analogy — semantic vs exact search |
| **Types** | FAISS, Chroma, Pinecone, etc. |
| **FAISS Demo** | `FAISS.from_documents()`, `save_local()`, `load_local()` |
| **Code Flow** | Load → Split → Embed → Store → Search |
| **Real-world** | Movie recommender system |

### Retrievers — Key Points
| Retriever | Covered |
|-----------|---------|
| **VectorStoreRetriever** | Basic similarity, score threshold |
| **MMR** | Maximum Marginal Relevance for diversity |
| **MultiQueryRetriever** | Query expansion via LLM |
| **ContextualCompressionRetriever** | LLM-based extraction / EmbeddingsFilter |
| **WikipediaRetriever** | External knowledge |
| **More** | SelfQuery, Ensemble, ParentDocument mentioned |

---

## 2.12 Decision Flowchart

```
Need to store & search embeddings?
│
├─► Local dev / prototype?
│   └─► FAISS (fastest, simplest)
│
├─► Production, need filtering?
│   ├─► Self-hosted → Chroma, Qdrant
│   └─► Managed → Pinecone
│
├─► Existing PostgreSQL?
│   └─► PGVector
│
├─► Hybrid BM25 + Vector?
│   └─► Elasticsearch, Weaviate
│
└─► Massive scale / custom infra?
    └─► Milvus

Then choose Retriever:
│
├─► Simple similarity? → VectorStoreRetriever (similarity)
├─► Diverse results? → MMR (lambda_mult=0.5)
├─► Query phrasing varies? → MultiQueryRetriever
├─► Long chunks, need precision? → ContextualCompressionRetriever
├─► Natural language filters? → SelfQueryRetriever
├─► Best of semantic + keyword? → EnsembleRetriever (BM25 + Dense)
├─► Small chunks need big context? → ParentDocumentRetriever
└─► Highest precision? → CrossEncoderReranker + ContextualCompression
```

---

## 2.13 Quick Reference: Key Classes

### Vector Stores
| Class | Package | Type |
|-------|---------|------|
| `FAISS` | `langchain_community.vectorstores` | In-memory local |
| `Chroma` | `langchain_chroma` | Local persistent |
| `PineconeVectorStore` | `langchain_pinecone` | Managed cloud |
| `PGVector` | `langchain_postgres` | PostgreSQL |
| `WeaviateVectorStore` | `langchain_weaviate` | Cloud/self-hosted |
| `QdrantVectorStore` | `langchain_qdrant` | Cloud/self-hosted |
| `Milvus` | `langchain_milvus` | Distributed |

### Retrievers
| Class | Package | Search Type |
|-------|---------|-------------|
| `VectorStoreRetriever` | `langchain_core.vectorstores` | similarity, mmr, score_threshold |
| `MultiQueryRetriever` | `langchain.retrievers.multi_query` | Query expansion |
| `ContextualCompressionRetriever` | `langchain.retrievers` | Compression/reranking |
| `SelfQueryRetriever` | `langchain.retrievers.self_query` | NL → filter |
| `EnsembleRetriever` | `langchain.retrievers` | RRF fusion |
| `ParentDocumentRetriever` | `langchain.retrievers` | Child→Parent |
| `BM25Retriever` | `langchain_community.retrievers` | Sparse/keyword |

### Compressors
| Class | Package | Method |
|-------|---------|--------|
| `LLMChainExtractor` | `langchain.retrievers.document_compressors` | LLM extraction |
| `EmbeddingsFilter` | `langchain.retrievers.document_compressors` | Embedding similarity |
| `CrossEncoderReranker` | `langchain.retrievers.document_compressors` | Cross-encoder |
| `DocumentCompressorPipeline` | `langchain.retrievers.document_compressors` | Multi-stage |

---

## 2.14 Summary: Vector Store + Retriever Cheat Sheet

```python
# THE STANDARD PRODUCTION RAG SETUP

# 1. EMBEDDINGS
from langchain_openai import OpenAIEmbeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 2. VECTOR STORE (Chroma - good balance)
from langchain_chroma import Chroma
vectorstore = Chroma(
    collection_name="rag_docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

# 3. ADD DOCUMENTS
# vectorstore.add_documents(chunks)

# 4. RETRIEVER WITH MMR + RERANKING
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

base_retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 10, "fetch_k": 30, "lambda_mult": 0.5}
)

reranker = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
compressor = CrossEncoderReranker(model=reranker, top_n=5)

retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever
)

# 5. LCEL RAG CHAIN
rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | ChatPromptTemplate.from_template(RAG_PROMPT)
    | ChatOpenAI(model="gpt-4o-mini")
    | StrOutputParser()
)

# 6. USE
result = rag_chain.invoke("Your question here")
```

---

## 2.15 What's Next

| Chapter | Focus | Builds On |
|---------|-------|-----------|
| **RAG Deep Dive** | Full RAG architectures, evaluation, streaming, citation, multi-hop | Retrievers + LCEL + Prompts |

---

## 2.16 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **Vector Stores:** https://python.langchain.com/docs/concepts/vectorstores/
- **Retrievers:** https://python.langchain.com/docs/concepts/retrievers/
- **FAISS:** https://python.langchain.com/docs/integrations/vectorstores/faiss/
- **Chroma:** https://python.langchain.com/docs/integrations/vectorstores/chroma/
- **Pinecone:** https://python.langchain.com/docs/integrations/vectorstores/pinecone/

### Advanced Topics
- **MMR Paper:** "Diversity-based Reranking" (Carbonell & Goldstein, 1998)
- **Cross-Encoders:** https://www.sbert.net/examples/applications/cross-encoder/README.html
- **RAG Evaluation:** https://github.com/explodinggradients/ragas

---

*End of Chapter 2 — Vector Stores & Retrievers: The Search Engine of RAG*