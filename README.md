# AI Engineering Learning Roadmap

A documentation-first repository for learning the concepts and tools used to build modern AI systems. The notes progress from Python and prompt engineering through LangChain, retrieval-augmented generation (RAG), observability, agentic systems, and the Model Context Protocol (MCP).

This repository is a personal learning resource, not a packaged application. It currently contains Markdown notes and guides; there is no root-level application code, dependency manifest, or test suite to install or run.

## Contents

- [Learning path](#learning-path)
- [Curriculum index](#curriculum-index)
- [Repository structure](#repository-structure)
- [How to use this repository](#how-to-use-this-repository)

## Learning path

The folders are organized as a practical progression, not as a requirement to use any one framework:

1. **Python foundations** — build fluency with the language and core software engineering practices.
2. **Prompt engineering** — learn prompt patterns, structured outputs, and reasoning approaches.
3. **LangChain** — explore models, prompts, LCEL, runnables, tools, and agents.
4. **MCP** — understand how AI applications connect to external tools and context.
5. **RAG** — learn ingestion, chunking, vector search, retrieval, and generation.
6. **LangSmith** — study tracing, evaluation, and debugging for GenAI applications.
7. **Agentic AI** — explore agent protocols, guardrails, context and token management, tools, evaluations, and testing.

For the broader roadmap—including topics not yet represented by dedicated folders—see [Curriculum](Docs/Curriculum.md) and [ROADMAP](Docs/ROADMAP.md).

## Curriculum index

### 01 - Python

The Python section has its own [module index and environment notes](01%20-%20Python/README.md).

- [Python Basics](01%20-%20Python/Basics/Python%20Basics.md)
- [Text Handling - Strings](01%20-%20Python/Collections/Text%20Handling%20-%20Strings.md)
- [Immutable Sequences - Tuples](01%20-%20Python/Collections/Immutable%20Sequences%20-%20Tuples.md)
- [Dynamic Sequences - Lists](01%20-%20Python/Collections/Dynamic%20Sequences%20-%20Lists.md)
- [Unique Collections - Sets](01%20-%20Python/Collections/Unique%20Collections%20-%20Sets.md)
- [Key Value Pairs - Dictionaries](01%20-%20Python/Collections/Key%20Value%20Pairs%20-%20Dictionaries.md)
- [Code Reusability - Functions and Modules](01%20-%20Python/Code%20Reusability%20-%20Functions%20and%20Modules.md)
- [Iterators, Generators and Decorators](01%20-%20Python/Iterators,%20Generators%20and%20Decorators.md)
- [Exception Handling and Context Managers](01%20-%20Python/Exception%20Handling%20and%20Context%20Managers.md)
- [Object Oriented Programming OOPS](01%20-%20Python/Object%20Oriented%20Programming%20OOPS.md)
- [Concurrency, Multithreading, Multiprocessing and AsyncIO](01%20-%20Python/Concurrency,%20Multithreading,%20Multiprocessing%20and%20AsyncIO.md)
- [Unit Testing with PyTest and Mocking](01%20-%20Python/Unit%20Testing%20with%20PyTest%20and%20Mocking.md)

### 02 - Prompt Engineering Foundations

- [Zero & Few Shot Prompting](02%20-%20Prompt%20Engineering%20Foundations/1%20-%20Zero%20%26%20Few%20Shot%20Prompting.md)
- [Output Formatting, JSON and Pydantic](02%20-%20Prompt%20Engineering%20Foundations/2%20-%20Output%20Formatting,%20JSON%20and%20Pydantic.md)
- [System Prompts and Role Playing](02%20-%20Prompt%20Engineering%20Foundations/3%20-%20System%20Prompts%20and%20Role%20Playing.md)
- [Chain of Thoughts and Reasoning](02%20-%20Prompt%20Engineering%20Foundations/4%20-%20Chain%20of%20Thoughts%20and%20Reasoning.md)

### 03 - LangChain

- [Introduction and Components](03%20-%20LangChain/1%20-%20Introduction%20and%20Components.md)
- [Models and Prompts](03%20-%20LangChain/2%20-%20Models%20and%20Prompts.md)
- [Structured Output and Parsers](03%20-%20LangChain/3%20-%20Structured%20Output%20and%20Parsers.md)
- [Chains and LCEL](03%20-%20LangChain/4%20-%20Chains%20and%20LCEL.md)
- [Runnables](03%20-%20LangChain/5%20-%20Runnables.md)
- [Tools and Tool-Calling](03%20-%20LangChain/6%20-%20Tools%20and%20Tool-Calling.md)
- [AI Agents](03%20-%20LangChain/7%20-%20AI%20Agents.md)

### 04 - MCP

- [MCP Complete Guide](04%20-%20MCP/MCP_Complete_Guide.md)

### 05 - RAG

- [Document Loaders and Text-Splitters](05%20-%20RAG/1%20-%20Document%20Loaders%20and%20Text-Splitters.md)
- [Vector Stores and Retrievers](05%20-%20RAG/2%20-%20Vector%20Stores%20and%20Retrievers.md)
- [Retrieval-Augmented Generation (RAG)](05%20-%20RAG/3%20-%20RAG.md)

### 06 - LangSmith

- [LangSmith: Observability, Evaluation and Debugging](06%20-%20LangSmith/LangSmith.md)

### 07 - Agentic AI

- [Agent-to-Agent (A2A) Protocol](07%20-%20Agentic%20AI/1%20-%20Agent-to-Agent%20%28A2A%29%20Protocol.md)
- [Guardrails](07%20-%20Agentic%20AI/2%20-%20Guardrails.md)
- [Context Engineering](07%20-%20Agentic%20AI/3%20-%20Context%20Engineering.md)
- [Token Optimization](07%20-%20Agentic%20AI/4%20-%20Token%20Optimization.md)
- [Local Tools](07%20-%20Agentic%20AI/5%20-%20Local%20Tools.md)
- [Prompt, Skill & Agent Evaluations](07%20-%20Agentic%20AI/6%20-%20Prompt,%20Skill%20%26%20Agent%20Evaluations.md)
- [Agent Testing](07%20-%20Agentic%20AI/7%20-%20Agent%20Testing.md)

### Docs

- [AI Philosophy](Docs/AI%20Philosophy.md)
- [Curriculum](Docs/Curriculum.md) — broad AI engineering learning curriculum.
- [Doubts about AI Engineering](Docs/Doubts%20about%20AI%20Engineering.md)
- [Project Ideas](Docs/Project%20Ideas.md)
- [ROADMAP](Docs/ROADMAP.md) — phased learning and project roadmap.

## Repository structure

```text
AI-Engineering/
├── 01 - Python/
│   ├── Basics/
│   │   └── Python Basics.md
│   ├── Collections/
│   │   ├── Dynamic Sequences - Lists.md
│   │   ├── Immutable Sequences - Tuples.md
│   │   ├── Key Value Pairs - Dictionaries.md
│   │   ├── Text Handling - Strings.md
│   │   └── Unique Collections - Sets.md
│   ├── Code Reusability - Functions and Modules.md
│   ├── Concurrency, Multithreading, Multiprocessing and AsyncIO.md
│   ├── Exception Handling and Context Managers.md
│   ├── Iterators, Generators and Decorators.md
│   ├── Object Oriented Programming OOPS.md
│   ├── README.md
│   └── Unit Testing with PyTest and Mocking.md
├── 02 - Prompt Engineering Foundations/
│   ├── 1 - Zero & Few Shot Prompting.md
│   ├── 2 - Output Formatting, JSON and Pydantic.md
│   ├── 3 - System Prompts and Role Playing.md
│   └── 4 - Chain of Thoughts and Reasoning.md
├── 03 - LangChain/
│   ├── 1 - Introduction and Components.md
│   ├── 2 - Models and Prompts.md
│   ├── 3 - Structured Output and Parsers.md
│   ├── 4 - Chains and LCEL.md
│   ├── 5 - Runnables.md
│   ├── 6 - Tools and Tool-Calling.md
│   └── 7 - AI Agents.md
├── 04 - MCP/
│   └── MCP_Complete_Guide.md
├── 05 - RAG/
│   ├── 1 - Document Loaders and Text-Splitters.md
│   ├── 2 - Vector Stores and Retrievers.md
│   └── 3 - RAG.md
├── 06 - LangSmith/
│   └── LangSmith.md
├── 07 - Agentic AI/
│   ├── 1 - Agent-to-Agent (A2A) Protocol.md
│   ├── 2 - Guardrails.md
│   ├── 3 - Context Engineering.md
│   ├── 4 - Token Optimization.md
│   ├── 5 - Local Tools.md
│   ├── 6 - Prompt, Skill & Agent Evaluations.md
│   └── 7 - Agent Testing.md
├── Docs/
│   ├── AI Philosophy.md
│   ├── Curriculum.md
│   ├── Doubts about AI Engineering.md
│   ├── Project Ideas.md
│   └── ROADMAP.md
└── README.md
```

## How to use this repository

- Read the notes in order within a numbered section, or use the curriculum index to jump to a topic.
- Treat code snippets and setup instructions inside individual notes as examples for that topic; there is no single repository-wide runtime environment.
- Follow links and references in each guide for deeper or up-to-date documentation.
- Use [Project Ideas](Docs/Project%20Ideas.md) to turn concepts into hands-on practice.
