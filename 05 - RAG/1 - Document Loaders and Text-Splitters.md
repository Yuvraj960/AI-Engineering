# Chapter 1: Document Loaders & Text Splitters — The Foundation of RAG

## 1.1 The RAG Pipeline Context

Before diving into loaders and splitters, understand where they fit in the **RAG (Retrieval-Augmented Generation)** pipeline:

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Document   │───▶│   Text       │───▶│  Embedding  │───▶│   Vector       │───▶│   Retriever    │───▶│   LLM +        │
│  Loaders    │    │  Splitters   │    │  Models     │    │   Store        │    │   (Search)     │    │   Prompt       │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────────┘    └────────────────┘    └────────────────┘
```

**Document Loaders** = "How do I get raw data into LangChain?"
**Text Splitters** = "How do I chunk documents for optimal retrieval?"

---

## 1.2 Document Loaders — Universal Data Ingestion

### 1.2.1 The Document Abstraction

Every loader returns a **list of `Document` objects**:

```python
from langchain_core.documents import Document

doc = Document(
    page_content="The actual text content...",  # str
    metadata={"source": "file.pdf", "page": 1}  # dict
)

# Key attributes:
# - doc.page_content: The text content
# - doc.metadata: Dict with source info (file path, page number, URL, etc.)
```

### 1.2.2 Loader Interface

All loaders implement the same interface:

```python
# Sync loading (loads all into memory)
docs = loader.load()  # → List[Document]

# Lazy loading (generator, memory efficient)
for doc in loader.lazy_load():  # → Generator[Document]
    process(doc)

# Async loading (for web loaders)
docs = await loader.aload()
async for doc in loader.alazy_load():
    process(doc)
```

---

### 1.2.3 Core Document Loaders

#### **TextLoader** — Plain Text Files
```python
# text_loader.py
from langchain_community.document_loaders import TextLoader

loader = TextLoader("cricket.txt", encoding="utf-8")
docs = loader.load()

print(len(docs))        # 1 document per file
print(docs[0].page_content)  # Full text content
print(docs[0].metadata)      # {'source': 'cricket.txt'}
```

**Use case:** Simple text files, logs, transcripts, code files.

---

#### **PyPDFLoader** — PDF Documents
```python
# pdf_loader.py
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("dl-curriculum.pdf")
docs = loader.load()

print(len(docs))           # 1 document PER PAGE
print(docs[0].page_content) # Page 1 text
print(docs[0].metadata)     # {'source': 'dl-curriculum.pdf', 'page': 0}
print(docs[1].metadata)     # {'source': 'dl-curriculum.pdf', 'page': 1}
```

**Key behavior:** Creates **one Document per page** — preserves page metadata for citation.

**Use case:** PDFs, research papers, reports, contracts.

---

#### **DirectoryLoader** — Batch Loading
```python
# directory_loader.py
from langchain_community.document_loaders import DirectoryLoader, PyPDFLoader

loader = DirectoryLoader(
    path="books",           # Directory path
    glob="*.pdf",           # File pattern (supports **/*.txt, *.md, etc.)
    loader_cls=PyPDFLoader, # Loader class for each file
    show_progress=True,     # Progress bar
    use_multithreading=True # Parallel loading
)

# Lazy load for memory efficiency
for document in loader.lazy_load():
    print(document.metadata)
    # {'source': 'books/paper1.pdf', 'page': 0}
    # {'source': 'books/paper1.pdf', 'page': 1}
    # {'source': 'books/paper2.pdf', 'page': 0}
```

**Use case:** Bulk processing folders of documents.

---

#### **WebBaseLoader** — Web Pages
```python
from langchain_community.document_loaders import WebBaseLoader

# Single URL
loader = WebBaseLoader("https://example.com/article")
docs = loader.load()

# Multiple URLs (parallel)
loader = WebBaseLoader([
    "https://site1.com/page",
    "https://site2.com/page"
])
docs = loader.load()  # Concurrent fetching
```

**Use case:** Blog posts, documentation, news articles, Wikipedia.

---

#### **CSVLoader** — Structured Data
```python
# csv_loader.py
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader(
    file_path="Social_Network_Ads.csv",
    source_column="User ID",        # Column to use as metadata source
    metadata_columns=["Age", "Gender"]  # Columns to include in metadata
)

docs = loader.load()
print(len(docs))  # 1 document per ROW
print(docs[0].page_content)  # "Age: 25, Gender: Male, Purchased: 0"
print(docs[0].metadata)      # {'source': 'Social_Network_Ads.csv', 'row': 0, 'Age': '25', 'Gender': 'Male'}
```

**Use case:** Databases exports, spreadsheets, structured records.

---

### 1.2.4 Other Important Loaders

| Loader | Package | Use Case |
|--------|---------|----------|
| **UnstructuredFileLoader** | `langchain-community` | Auto-detects 20+ formats (PDF, DOCX, PPTX, HTML, XML, EPUB, etc.) |
| **JSONLoader** | `langchain-community` | JSON/JSONL files with jq schema |
| **NotionDBLoader** | `langchain-community` | Notion databases |
| **ConfluenceLoader** | `langchain-community` | Confluence pages |
| **GitLoader** | `langchain-community` | Git repositories (code files) |
| **S3FileLoader** | `langchain-community` | AWS S3 objects |
| **AzureBlobStorageFileLoader** | `langchain-community` | Azure blobs |
| **GoogleDriveLoader** | `langchain-google-community` | Google Drive files |
| **SlackLoader** | `langchain-community` | Slack conversations |
| **DiscordLoader** | `langchain-community` | Discord channels |
| **OutlookMessageLoader** | `langchain-community` | .msg email files |

---

### 1.2.5 Custom Document Loader

```python
# Custom loader for proprietary format
from langchain_core.document_loaders import BaseLoader
from langchain_core.documents import Document
from typing import Iterator

class CustomAPILoader(BaseLoader):
    def __init__(self, api_endpoint: str, api_key: str):
        self.api_endpoint = api_endpoint
        self.api_key = api_key
    
    def lazy_load(self) -> Iterator[Document]:
        # Fetch from API
        import requests
        headers = {"Authorization": f"Bearer {self.api_key}"}
        response = requests.get(self.api_endpoint, headers=headers)
        data = response.json()
        
        for item in data["records"]:
            yield Document(
                page_content=item["content"],
                metadata={"id": item["id"], "source": "custom_api", "timestamp": item["created_at"]}
            )

# Usage
loader = CustomAPILoader("https://api.company.com/docs", "sk-...")
for doc in loader.lazy_load():
    process(doc)
```

---

### 1.2.6 Load vs Lazy Load — Memory Strategy

| Method | Returns | Memory | Use When |
|--------|---------|--------|----------|
| `loader.load()` | `List[Document]` | All in RAM | Small datasets (< 1000 docs) |
| `loader.lazy_load()` | `Generator[Document]` | One at a time | Large datasets, streaming pipelines |
| `loader.aload()` | `List[Document]` (async) | All in RAM | Async contexts (FastAPI) |
| `loader.alazy_load()` | `AsyncGenerator` | One at a time | Async streaming |

**Best Practice:** Default to `lazy_load()` for production pipelines.

---

## 1.3 Text Splitters — Chunking Strategies

### 1.3.1 Why Split Documents?

```
Problem: LLM context windows are limited (4K–128K tokens)
         Embedding models have token limits (512–8192)
         Large chunks = poor retrieval precision
         Small chunks = lost context

Solution: Split into overlapping chunks that:
  • Fit in context window
  • Preserve semantic coherence
  • Overlap to maintain continuity
  • Include metadata for traceability
```

### 1.3.2 The Splitter Interface

```python
from langchain_text_splitters import TextSplitter

# Split raw text
chunks = splitter.split_text(text)  # → List[str]

# Split Document objects (preserves metadata)
chunks = splitter.split_documents(docs)  # → List[Document]

# Create documents from text (adds metadata)
docs = splitter.create_documents([text])  # → List[Document]
```

All splitters inherit from `TextSplitter` base class.

---

### 1.3.3 Categories of Text Splitters

```
┌────────────────────────────────────────────────────────────────┐
│                    TEXT SPLITTER TAXONOMY                      │
├──────────────────┬──────────────────┬──────────────────────────┤
│ 1. Length-Based  │ 2. Structure-    │ 3. Document-Structure    │
│    (Character)   │    Based         │    Based                 │
├──────────────────┼──────────────────┼──────────────────────────┤
│ CharacterText    │ RecursiveChar    │ MarkdownHeader           │
│ Splitter         │ TextSplitter     │ TextSplitter             │
│                  │                  │                          │
│ RecursiveChar    │ Language-Specific│ HTMLHeaderTextSplitter   │
│ (default)        │ (Python, JS,     │                          │
│                  │  Markdown, etc.) │                          │
├──────────────────┼──────────────────┼──────────────────────────┘
│ 4. Semantic      │ 5. Specialized   │                          
│    Meaning       │                  │                          
├──────────────────┼──────────────────┤
│ SemanticChunker  │ TokenTextSplitter│                          
│ (embedding-based)│ (token-aware)    │                          
└──────────────────┴──────────────────┘
```

---

### 1.3.4 Length-Based: CharacterTextSplitter

**Simplest splitter** — splits by character count with optional separator.

```python
# length_based.py
from langchain.text_splitter import CharacterTextSplitter
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("dl-curriculum.pdf")
docs = loader.load()

splitter = CharacterTextSplitter(
    chunk_size=200,        # Target chunk size (characters)
    chunk_overlap=0,       # Overlap between chunks
    separator=""           # Split every N chars (empty = every char)
)

result = splitter.split_documents(docs)
print(result[1].page_content)
```

**Parameters:**
| Param | Default | Description |
|-------|---------|-------------|
| `chunk_size` | 4000 | Max characters per chunk |
| `chunk_overlap` | 200 | Overlap characters |
| `separator` | `"\n\n"` | Split on this string first |
| `keep_separator` | `True` | Keep separator in chunks |

**Limitation:** Doesn't respect semantic boundaries — can split mid-sentence, mid-word.

---

### 1.3.5 Structure-Based: RecursiveCharacterTextSplitter

**DEFAULT RECOMMENDED SPLITTER** — Recursively tries separators in order.

```python
# text_structure_based.py
from langchain.text_splitter import RecursiveCharacterTextSplitter

text = """
Space exploration has led to incredible scientific discoveries. 
From landing on the Moon to exploring Mars, humanity continues to push 
the boundaries of what's possible beyond our planet.

These missions have not only expanded our knowledge of the universe 
but have also contributed to advancements in technology here on Earth. 
Satellite communications, GPS, and even certain medical imaging 
techniques trace their roots back to innovations driven by space programs.
"""

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,  # Overlap helps continuity
    separators=["\n\n", "\n", ". ", " ", ""]  # Default hierarchy
)

chunks = splitter.split_text(text)
print(len(chunks))
print(chunks)
```

**Default Separator Hierarchy:**
```python
separators = [
    "\n\n",      # Paragraphs
    "\n",        # Lines
    ". ",        # Sentences
    " ",         # Words
    ""           # Characters (last resort)
]
```

**How It Works:**
```
Text → Try "\n\n" → If chunk > size, try "\n" → If still > size, try ". " → ...
```

**Why It's Best:** Respects natural document structure → better semantic coherence.

---

### 1.3.6 Document-Structure Based Splitters

#### **MarkdownHeaderTextSplitter** — Preserve Heading Hierarchy

```python
# markdown_splitting.py
from langchain.text_splitter import MarkdownHeaderTextSplitter

text = """
# Project Name: Smart Student Tracker

A simple Python-based project to manage student data.

## Features

- Add new students with relevant info
- View student details
- Check if a student is passing

## 🛠 Tech Stack

- Python 3.10+
- No external dependencies

## Getting Started

1. Clone the repo
   ```bash
   git clone https://github.com/your-username/student-tracker.git
   ```
"""

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "header_1"),
        ("##", "header_2"),
        ("###", "header_3"),
    ],
    strip_headers=False
)

chunks = splitter.split_text(text)
# Each chunk has metadata: {'header_1': 'Project Name', 'header_2': 'Features'}
print(chunks[0].metadata)  # {'header_1': 'Project Name', 'header_2': 'Features'}
```

**Output:** Chunks with metadata preserving heading hierarchy — great for citation.

---

#### **RecursiveCharacterTextSplitter.from_language()** — Code-Aware

```python
# python_code_splitting.py
from langchain.text_splitter import RecursiveCharacterTextSplitter, Language

text = """
class Student:
    def __init__(self, name, age, grade):
        self.name = name
        self.age = age
        self.grade = grade

    def get_details(self):
        return self.name

    def is_passing(self):
        return self.grade >= 6.0

student1 = Student("Aarav", 20, 8.2)
print(student1.get_details())
"""

# Language-specific splitting!
splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=300,
    chunk_overlap=0,
)

chunks = splitter.split_text(text)
# Splits on: class, def, if/else, try/except, etc.
print(chunks[1])  # Preserves function/class boundaries
```

**Supported Languages:**
```python
Language.PYTHON, Language.JS, Language.TS, Language.JAVA, 
Language.CPP, Language.CS, Language.GO, Language.RUST, 
Language.RUBY, Language.PHP, Language.MARKDOWN, Language.HTML,
Language.LATEX, Language.SOL, Language.PROTO
```

---

#### **HTMLHeaderTextSplitter** — Web Content

```python
from langchain.text_splitter import HTMLHeaderTextSplitter

html = """
<h1>Introduction to RAG</h1>
<p>RAG combines retrieval with generation...</p>
<h2>Document Loaders</h2>
<p>Loaders ingest data...</p>
<h3>PDF Loader</h3>
<p>PyPDFLoader loads PDFs...</p>
<h2>Text Splitters</h2>
<p>Splitters chunk documents...</p>
"""

splitter = HTMLHeaderTextSplitter(
    headers_to_split_on=[("h1", "header_1"), ("h2", "header_2"), ("h3", "header_3")]
)

chunks = splitter.split_text(html)
# Metadata: {'header_1': 'Introduction to RAG', 'header_2': 'Document Loaders', ...}
```

---

### 1.3.7 Semantic Meaning Based: SemanticChunker

**Advanced** — Uses embeddings to split at semantic boundaries.

```python
# semantic_meaning_based.py
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai.embeddings import OpenAIEmbeddings
from dotenv import load_dotenv

load_dotenv()

text_splitter = SemanticChunker(
    OpenAIEmbeddings(), 
    breakpoint_threshold_type="standard_deviation",  # or "interquartile", "percentile"
    breakpoint_threshold_amount=3  # Sensitivity (higher = fewer chunks)
)

sample = """
Farmers were working hard in the fields, preparing the soil and planting 
seeds for the next season. The sun was bright, and the air smelled of earth 
and fresh grass.

The Indian Premier League (IPL) is the biggest cricket league in the world. 
People all over the world watch the matches and cheer for their favourite teams.

Terrorism is a big danger to peace and safety. It causes harm to people and 
creates fear in cities and villages. When such attacks happen, they leave 
behind pain and sadness.
"""

docs = text_splitter.create_documents([sample])
print(len(docs))  # Splits at topic boundaries!
for doc in docs:
    print(f"--- Chunk ---\n{doc.page_content[:100]}...")
```

**How It Works:**
1. Embeds sentences
2. Computes cosine similarity between adjacent sentences
3. Splits where similarity drops below threshold

**Threshold Types:**
| Type | Description |
|------|-------------|
| `"standard_deviation"` | Mean - k*std |
| `"interquartile"` | Q1 - k*IQR |
| `"percentile"` | k-th percentile |

**Pros:** Best semantic coherence. **Cons:** Slower (embedding calls), requires API key.

---

### 1.3.8 Token-Based: TokenTextSplitter

```python
from langchain.text_splitter import TokenTextSplitter

# Accurate token counting (matches LLM tokenizer)
splitter = TokenTextSplitter(
    chunk_size=512,      # Tokens (not characters!)
    chunk_overlap=50,
    encoding_name="cl100k_base"  # GPT-4/3.5-turbo tokenizer
)

# Or use specific model tokenizer
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("gpt2")
splitter = TokenTextSplitter.from_huggingface_tokenizer(tokenizer, chunk_size=512)
```

**Use when:** Precise token budgeting for LLM context windows.

---

## 1.4 Choosing the Right Splitter

| Scenario | Recommended Splitter | Why |
|----------|---------------------|-----|
| **General purpose** | `RecursiveCharacterTextSplitter` | Best balance of structure + simplicity |
| **Markdown docs** | `MarkdownHeaderTextSplitter` | Preserves heading hierarchy |
| **Code files** | `RecursiveCharacterTextSplitter.from_language(Language.PYTHON)` | Respects syntax |
| **HTML/Web** | `HTMLHeaderTextSplitter` | Preserves DOM structure |
| **High-quality RAG** | `SemanticChunker` | Best semantic coherence |
| **Token budgeting** | `TokenTextSplitter` | Exact token counts |
| **Simple/fast** | `CharacterTextSplitter` | Minimal overhead |

### Chunk Size Guidelines

| Use Case | Chunk Size | Overlap |
|----------|------------|---------|
| **General RAG** | 500-1000 chars | 50-100 |
| **Long context models** | 2000-4000 chars | 200-400 |
| **Code** | 500-1500 chars | 50-100 |
| **Semantic** | N/A (auto) | N/A |
| **High precision retrieval** | 200-400 chars | 20-50 |

---

## 1.5 Integration with LCEL

### 1.5.1 Loader → Splitter → Embeddings Chain

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.runnables import RunnableLambda

# Load
loader = PyPDFLoader("document.pdf")
docs = loader.load()

# Split
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
chunks = splitter.split_documents(docs)

# Embed & Store (RunnableLambda for LCEL)
embed = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(chunks, embed)

# As LCEL Runnable
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
```

---

### 1.5.2 Full RAG Chain with Loaders/Splitters

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

RAG_PROMPT = ChatPromptTemplate.from_template("""
Answer using only the context below. Cite sources.

Context: {context}

Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini")

# Full chain with retrieval
rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough()
    )
    | RAG_PROMPT
    | llm
    | StrOutputParser()
)

# Streaming RAG
for chunk in rag_chain.stream("What is the main finding?"):
    print(chunk, end="", flush=True)
```

---

### 1.5.3 Batch Processing Pipeline

```python
from langchain_community.document_loaders import DirectoryLoader, PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.runnables import RunnableLambda
import asyncio

async def build_vectorstore():
    # 1. Load (parallel)
    loader = DirectoryLoader("documents/", glob="**/*.pdf", loader_cls=PyPDFLoader, use_multithreading=True)
    docs = loader.load()  # or await loader.aload()
    
    # 2. Split
    splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
    chunks = splitter.split_documents(docs)
    
    # 3. Embed (batch)
    embeddings = OpenAIEmbeddings()
    vectorstore = await FAISS.afrom_documents(chunks, embeddings)
    
    return vectorstore

vectorstore = asyncio.run(build_vectorstore())
```

---

## 1.6 Metadata Preservation & Enrichment

### 1.6.1 Automatic Metadata

Loaders automatically add metadata:

| Loader | Metadata Fields |
|--------|----------------|
| `PyPDFLoader` | `source`, `page` |
| `TextLoader` | `source` |
| `WebBaseLoader` | `source`, `title`, `description` |
| `CSVLoader` | `source`, `row`, all columns |
| `DirectoryLoader` | `source`, plus loader-specific |

### 1.6.2 Custom Metadata Enrichment

```python
from langchain_core.runnables import RunnableLambda

def enrich_metadata(doc: Document) -> Document:
    doc.metadata.update({
        "processed_at": datetime.now().isoformat(),
        "doc_length": len(doc.page_content),
        "chunk_id": hash(doc.page_content) % 10000
    })
    return doc

# In chain
enrich = RunnableLambda(enrich_metadata)
chunks = splitter.split_documents(docs) | enrich.map()
```

### 1.6.3 Metadata for Filtering (Hybrid Search)

```python
# At query time, filter by metadata
from langchain_community.vectorstores import Chroma

vectorstore = Chroma.from_documents(chunks, embeddings)

# Filter by metadata
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"source": "annual_report.pdf"}  # Only search specific doc
)

# Or use retriever with filter
retriever = vectorstore.as_retriever(
    search_kwargs={"k": 5, "filter": {"page": {"$gte": 10}}}  # Pages 10+
)
```

---

## 1.7 Production Best Practices

### Document Loading
- [ ] **Use `lazy_load()`** for large datasets
- [ ] **Handle encoding** explicitly (`encoding="utf-8"`)
- [ ] **Validate file existence** before loading
- [ ] **Parallelize** with `use_multithreading=True` (DirectoryLoader)
- [ ] **Error handling** — wrap in try/except, log failures

### Text Splitting
- [ ] **Default to `RecursiveCharacterTextSplitter`**
- [ ] **Tune chunk_size** for your embedding model (512-1000 typical)
- [ ] **Always use overlap** (10-20% of chunk_size)
- [ ] **Preserve metadata** — use `split_documents()`, not `split_text()`
- [ ] **Test retrieval quality** — not just chunk count

### Metadata
- [ ] **Enrich at load time** (source, date, author, category)
- [ ] **Use consistent schema** across all loaders
- [ ] **Index metadata** for filtered retrieval

### Performance
- [ ] **Batch embedding** (`embeddings.embed_documents()`)
- [ ] **Async loading** for web/io-bound sources
- [ ] **Stream processing** for massive corpora

---

## 1.8 Complete Decision Flowchart

```
Need to load data?
│
├─► File type?
│   ├─ PDF → PyPDFLoader (page-level) or UnstructuredFileLoader (auto)
│   ├─ Text/Code → TextLoader
│   ├─ CSV/Excel → CSVLoader / UnstructuredExcelLoader
│   ├─ Web → WebBaseLoader
│   ├─ Directory → DirectoryLoader + appropriate loader_cls
│   ├─ JSON → JSONLoader (with jq_schema)
│   ├─ Notion/Confluence/Slack → Specialized loaders
│   └─ Custom → Extend BaseLoader
│
├─► Memory concern?
│   └─► Use `lazy_load()` / `alazy_load()`
│
└─► Need to split?
    │
    ├─► General text → RecursiveCharacterTextSplitter (chunk_size=1000, overlap=100)
    ├─► Markdown → MarkdownHeaderTextSplitter
    ├─► Code → RecursiveCharacterTextSplitter.from_language(Language.PYTHON)
    ├─► HTML → HTMLHeaderTextSplitter
    ├─► Semantic precision → SemanticChunker (OpenAI embeddings)
    ├─► Token precision → TokenTextSplitter
    └─► Simple/fast → CharacterTextSplitter
```

---

## 1.9 Code Reference

### Document Loaders (`langchain-document-loaders`)

| File | Loader | Key Pattern |
|------|--------|-------------|
| `text_loader.py` | `TextLoader` | Basic text + chain integration |
| `pdf_loader.py` | `PyPDFLoader` | Page-per-document |
| `directory_loader.py` | `DirectoryLoader` | Lazy loading with glob |
| `csv_loader.py` | `CSVLoader` | Row-per-document with metadata |
| `web_loader.py` | `WebBaseLoader` | Single/multi-URL fetching |

### Text Splitters (`langchain-text-splitters`)

| File | Splitter | Strategy |
|------|----------|----------|
| `length_based.py` | `CharacterTextSplitter` | Fixed character count |
| `text_structure_based.py` | `RecursiveCharacterTextSplitter` | Recursive separators |
| `markdown_splitting.py` | `MarkdownHeaderTextSplitter` | Heading hierarchy |
| `python_code_splitting.py` | `RecursiveCharacterTextSplitter.from_language()` | Python syntax |
| `semantic_meaning_based.py` | `SemanticChunker` | Embedding-based boundaries |

---

## 1.10 Quick Reference: Key Classes

### Document Loaders
| Loader | Package | Best For |
|--------|---------|----------|
| `TextLoader` | `langchain_community.document_loaders` | Plain text |
| `PyPDFLoader` | `langchain_community.document_loaders` | PDFs (page-level) |
| `DirectoryLoader` | `langchain_community.document_loaders` | Batch folders |
| `WebBaseLoader` | `langchain_community.document_loaders` | URLs |
| `CSVLoader` | `langchain_community.document_loaders` | Structured data |
| `UnstructuredFileLoader` | `langchain_community.document_loaders` | Auto-detect 20+ formats |
| `JSONLoader` | `langchain_community.document_loaders` | JSON with jq |
| `BaseLoader` | `langchain_core.document_loaders` | Custom loaders |

### Text Splitters
| Splitter | Package | Strategy |
|----------|---------|----------|
| `CharacterTextSplitter` | `langchain.text_splitter` | Fixed length |
| `RecursiveCharacterTextSplitter` | `langchain.text_splitter` | **Default recommended** |
| `MarkdownHeaderTextSplitter` | `langchain.text_splitter` | Markdown headings |
| `HTMLHeaderTextSplitter` | `langchain.text_splitter` | HTML headings |
| `RecursiveCharacterTextSplitter.from_language()` | `langchain.text_splitter` | Code-aware |
| `SemanticChunker` | `langchain_experimental.text_splitter` | Embedding-based |
| `TokenTextSplitter` | `langchain.text_splitter` | Token-counting |

---

## 1.11 Summary: Loader + Splitter Cheat Sheet

```python
# THE STANDARD RAG INGESTION PIPELINE

# 1. LOAD
from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader
loader = DirectoryLoader("data/", glob="**/*.pdf", loader_cls=PyPDFLoader)
docs = loader.lazy_load()  # Generator for memory efficiency

# 2. SPLIT
from langchain.text_splitter import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""]
)
chunks = []
for doc in docs:
    chunks.extend(splitter.split_documents([doc]))  # Preserves metadata

# 3. ENRICH (optional)
for i, chunk in enumerate(chunks):
    chunk.metadata["chunk_index"] = i
    chunk.metadata["chunk_size"] = len(chunk.page_content)

# 4. EMBED & STORE
# vectorstore = Chroma.from_documents(chunks, embeddings)
```

---

## 1.12 What's Next

| Chapter | Focus | Builds On |
|---------|-------|-----------|
| **2. Vector Stores & Retrievers** | Embeddings, Chroma/FAISS/Pinecone, similarity search, hybrid search, reranking | Loaders + Splitters → Documents → Embeddings |
| **3. RAG Deep Dive** | Full retrieval + generation, citation, streaming, evaluation | Retrievers + LCEL chains |


---

## 1.13 References & Resources

### Videos
1. **CampusX Playlist:** "Generative AI using LangChain" — https://www.youtube.com/playlist?list=PLKnIA16_RmvaTbihpo4MtzVm4XOQa0ER0

### Official Documentation
- **Document Loaders:** https://python.langchain.com/docs/concepts/document_loaders/
- **Text Splitters:** https://python.langchain.com/docs/concepts/text_splitters/
- **Document Abstraction:** https://python.langchain.com/docs/concepts/documents/

### Related Resources
- **RAG Chunking Strategies:** https://www.pinecone.io/learn/chunking-strategies/
- **Semantic Chunking:** https://github.com/langchain-ai/langchain/blob/master/libs/experimental/langchain_experimental/text_splitter/semantic_chunker.py

---

*End of Chapter 1 — Document Loaders & Text Splitters: The Foundation of RAG*