# Local Tools: MCP & Claude Code — Complete Technical Guide

## 1. What Is the Model Context Protocol (MCP)?

### 1.1 Definition

> **MCP (Model Context Protocol)** is an open-source standard for connecting AI applications to external systems. Think of MCP like a **USB-C port for AI applications** — a standardized way to connect AI applications to data sources, tools, and workflows.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MCP ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐         MCP Protocol            ┌────────────────────┐   │
│   │  AI CLIENT   │ ◄─────────────────────────────▶ │   MCP SERVER       │   │
│   │ (Claude,     │    JSON-RPC over stdio/HTTP     │  (Files, DB,       │   │
│   │  ChatGPT,    │                                 │   Search, API,     │   │
│   │  VS Code,    │   ┌───────────────────────┐     │   Git, Blender,    │   │
│   │  Cursor)     │   │  Tools / Resources    │     │   3D Printer...)   │   │
│   └──────────────┘   │  • read_file          │     └────────────────────┘   │
│                      │  • write_file         │                │             │
│                      │  • search             │                │             │
│                      │  • execute_code       │                ▼             │
│                      │  • query_db           │     ┌────────────────────┐   │
│                      │  • git_ops            │     │  EXTERNAL SYSTEMS  │   │
│                      └───────────────────────┘     │  (Local / Remote)  │   │
│                                                    └────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 What MCP Enables

| Capability | Example |
|------------|---------|
| **File System Access** | Read/write local files, search codebase |
| **Database Queries** | SQL/NoSQL queries, vector search |
| **API Integration** | GitHub, Jira, Slack, Notion, Google Calendar |
| **Code Execution** | Run tests, lint, build, deploy |
| **Specialized Tools** | Blender 3D, 3D printers, Figma design |
| **Workflow Automation** | Custom prompts, multi-step pipelines |

### 1.3 Why MCP Matters

| Role | Benefit |
|------|---------|
| **Developers** | Reduced integration complexity, build once integrate everywhere |
| **AI Applications** | Access to ecosystem of data sources & tools |
| **End Users** | More capable agents with real-world actions |

### 1.4 Ecosystem Support (2026)

| Category | Examples |
|----------|----------|
| **AI Assistants** | Claude, ChatGPT |
| **IDEs/Editors** | VS Code, Cursor, Windsurf, Zed |
| **CLI Tools** | Claude Code, MCPJam, various CLIs |
| **Servers** | Filesystem, Git, PostgreSQL, Redis, GitHub, Docker, Kubernetes |

---

## 2. MCP Protocol Deep Dive

### 2.1 Transport Layer

```
MCP TRANSPORTS:
─────────────────────────────────────────────────────────────────────────────
├── stdio (Local)     ← Most common for local tools
│   ├── Client spawns server process
│   ├── Communication via stdin/stdout
│   └── JSON-RPC messages
│
├── HTTP/SSE (Remote) ← For remote servers
│   ├── REST-like endpoints
│   ├── Server-Sent Events for streaming
│   └── Auth via headers
│
└── WebSocket (Real-time)
    ├── Bidirectional streaming
    └── Lower latency for interactive tools
```

### 2.2 Core MCP Primitives

```typescript
// MCP Resource (read-only data)
interface Resource {
  uri: string;           // "file:///project/src/main.py"
  name: string;          // "main.py"
  description?: string;
  mimeType?: string;     // "text/x-python"
}

// MCP Tool (executable function)
interface Tool {
  name: string;          // "read_file"
  description: string;   // "Read a file from the filesystem"
  inputSchema: {         // JSON Schema
    type: "object",
    properties: {
      path: { type: "string" }
    },
    required: ["path"]
  };
}

// MCP Prompt (reusable prompt template)
interface Prompt {
  name: string;          // "code_review"
  description: string;
  arguments?: PromptArgument[];
}
```

### 2.3 Example: Filesystem Server

```python
# mcp_servers/filesystem.py
from mcp.server import Server
from mcp.types import Tool, TextContent
import os

server = Server("filesystem")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="read_file",
            description="Read a file from the filesystem",
            inputSchema={
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"]
            }
        ),
        Tool(
            name="write_file",
            description="Write a file to the filesystem",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                    "content": {"type": "string"}
                },
                "required": ["path", "content"]
            }
        ),
        Tool(
            name="list_directory",
            description="List directory contents",
            inputSchema={
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "read_file":
        path = arguments["path"]
        with open(path, 'r') as f:
            return [TextContent(type="text", text=f.read())]
    elif name == "write_file":
        path = arguments["path"]
        content = arguments["content"]
        with open(path, 'w') as f:
            f.write(content)
        return [TextContent(type="text", text=f"Written to {path}")]
    # ...

if __name__ == "__main__":
    server.run_stdio()
```

### 2.4 MCP Client Configuration (Claude Code)

```json
// .mcp.json or claude_desktop_config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "python",
      "args": ["-m", "mcp_servers.filesystem"],
      "env": {}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${DATABASE_URL}"
      }
    },
    "docker": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "mcp/docker"],
      "env": {}
    }
  }
}
```

---

## 3. Claude Code: Agentic Coding Assistant

### 3.1 Overview

> **Claude Code** is an agentic coding tool that reads your codebase, edits files, runs commands, and integrates with your development tools. Available in terminal, IDE, desktop app, and browser.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLAUDE CODE ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SURFACES (all share same engine & config)                                 │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│   │Terminal │ │VS Code  │ │JetBrains│ │Desktop  │ │ Web     │               │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘               │
│        │           │           │           │           │                    │
│        └───────────┼───────────┼───────────┼───────────┘                    │
│                    ▼           ▼           ▼                                │
│         ┌─────────────────────────────────────────┐                         │
│         │         CLAUDE CODE ENGINE              │                         │
│         │  • Codebase understanding (AST, grep)   │                         │
│         │  • Multi-file editing                   │                         │
│         │  • Tool execution (bash, git, test)     │                         │
│         │  • MCP server integration               │                         │
│         │  • Memory (CLAUDE.md + auto-memory)     │                         │
│         │  • Skills (reusable workflows)          │                         │
│         │  • Hooks (pre/post action commands)     │                         │
│         └─────────────────────────────────────────┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Installation

```bash
# macOS (Homebrew)
brew install claude-code          # Stable (week behind)
brew install claude-code@latest   # Latest channel

# Windows (WinGet)
winget install Anthropic.ClaudeCode

# Linux (apt/dnf/apk)
# Ubuntu/Debian:
curl -fsSL https://claude.ai/install.sh | sh

# Or npm (cross-platform)
npm install -g @anthropic/claude-code

# Start in project directory
cd your-project
claude
```

### 3.3 Core Capabilities

| Capability | Example Command |
|------------|-----------------|
| **Write Tests** | `claude "write tests for auth module, run them, fix failures"` |
| **Fix Lint Errors** | `claude "fix all lint errors in the project"` |
| **Resolve Merge Conflicts** | `claude "resolve merge conflicts in feature branch"` |
| **Update Dependencies** | `claude "update all deps, run tests, fix breaking changes"` |
| **Write Release Notes** | `claude "generate release notes from commits since v1.0"` |
| **Build Features** | `claude "add OAuth2 login with Google and GitHub"` |
| **Debug Bugs** | `claude "trace this error: [paste stack trace]"` |

---

## 4. Customization: CLAUDE.md, Skills & Hooks

### 4.1 CLAUDE.md — Project Instructions

```markdown
# CLAUDE.md (Project Root)

## Project Overview
This is a Python FastAPI microservice for user authentication.

## Coding Standards
- Use type hints on all functions
- Follow PEP 8, max line length 100
- Use Pydantic v2 for validation
- Async/await for all I/O

## Architecture Decisions
- Repository pattern for data access
- JWT tokens with RS256
- Redis for session storage
- PostgreSQL with asyncpg

## Preferred Libraries
- HTTP: httpx (not requests)
- Validation: pydantic v2
- Testing: pytest + pytest-asyncio
- Linting: ruff (not flake8)

## Review Checklist
- [ ] All public functions have docstrings
- [ ] Error handling for external calls
- [ ] No hardcoded secrets
- [ ] Tests cover happy + error paths
```

### 4.2 Skills — Reusable Workflows

```markdown
# .claude/skills/review-pr.md
---
name: review-pr
description: Review a pull request for code quality, security, and correctness
---

# PR Review Skill

## Steps
1. Fetch PR diff: `gh pr diff <number>`
2. Analyze changes for:
   - Security issues (SQL injection, XSS, secrets)
   - Performance problems (N+1 queries, missing indexes)
   - Code style violations
   - Missing tests
3. Run tests: `pytest -xvs`
4. Run linter: `ruff check .`
5. Check types: `mypy .`
6. Post review comment with findings

## Usage
`/review-pr 123`
```

```markdown
# .claude/skills/deploy-staging.md
---
name: deploy-staging
description: Deploy current branch to staging environment
---

# Deploy to Staging

## Steps
1. Run full test suite
2. Build Docker image: `docker build -t app:staging .`
3. Push to registry: `docker push registry/app:staging`
4. Deploy to Kubernetes: `kubectl set image deployment/app app=registry/app:staging`
5. Run smoke tests against staging
6. Notify team in Slack

## Usage
`/deploy-staging`
```

### 4.3 Hooks — Automation Triggers

```json
// .claude/settings.json
{
  "hooks": {
    "post-edit": [
      "ruff format {file}",
      "ruff check --fix {file}"
    ],
    "pre-commit": [
      "pytest -x --tb=short",
      "mypy ."
    ],
    "post-task": [
      "git diff --stat"
    ]
  }
}
```

```bash
# Hook examples
# Auto-format after every edit
ruff format {file}

# Run tests before commit
pytest -x --tb=short

# Show diff summary after task
git diff --stat
```

---

## 5. Auto-Memory & Cross-Session Persistence

### 5.1 How It Works

```
AUTO-MEMORY FLOW:
─────────────────────────────────────────────────────────────────────────────
1. During work, Claude Code observes:
   - Patterns in your codebase
   - Recurring decisions
   - Project conventions
   - Failed attempts & solutions

2. Automatically saves to:
   ~/.claude/memory/<project>/auto_memory.md

3. Next session, loads automatically:
   - No manual setup required
   - Improves over time

4. You can also manually add:
   # CLAUDE.md (explicit, version-controlled)
   # .claude/memory/manual_notes.md (private)
```

### 5.2 Memory Structure

```
~/.claude/memory/<project>/
├── auto_memory.md          # Auto-discovered patterns
├── manual_notes.md         # User-added notes
├── session_history.json    # Recent sessions
└── preferences.json        # Learned preferences
```

---

## 6. MCP Integration with Claude Code

### 6.1 Built-in MCP Support

Claude Code **natively supports MCP servers** — same config works across all surfaces (Terminal, VS Code, Desktop, Web).

```json
// .mcp.json (project-level, checked into git)
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/project/root"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "POSTGRES_CONNECTION_STRING": "${DATABASE_URL}" }
    }
  }
}
```

### 6.2 Popular MCP Servers

| Server | Package | Capabilities |
|--------|---------|--------------|
| **Filesystem** | `@modelcontextprotocol/server-filesystem` | Read/write/list files |
| **Git** | `@modelcontextprotocol/server-git` | Git operations |
| **GitHub** | `@modelcontextprotocol/server-github` | Issues, PRs, repos |
| **PostgreSQL** | `@modelcontextprotocol/server-postgres` | SQL queries |
| **SQLite** | `@modelcontextprotocol/server-sqlite` | Local DB |
| **Redis** | `@modelcontextprotocol/server-redis` | Cache/queue ops |
| **Docker** | `mcp/docker` | Container management |
| **Kubernetes** | `mcp/kubernetes` | Cluster ops |
| **Puppeteer** | `@modelcontextprotocol/server-puppeteer` | Browser automation |
| **Slack** | `@modelcontextprotocol/server-slack` | Channel/message ops |

### 6.3 Custom MCP Server for Your Project

```python
# mcp_servers/project_tools.py
from mcp.server import Server
from mcp.types import Tool, TextContent
import subprocess
import json

server = Server("project-tools")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="run_tests",
            description="Run pytest with coverage",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "default": "."},
                    "coverage": {"type": "boolean", "default": True}
                }
            }
        ),
        Tool(
            name="run_linter",
            description="Run ruff linter",
            inputSchema={"type": "object", "properties": {}}
        ),
        Tool(
            name="db_migrate",
            description="Run database migrations",
            inputSchema={
                "type": "object",
                "properties": {
                    "direction": {"type": "string", "enum": ["up", "down"]},
                    "steps": {"type": "integer", "default": 1}
                }
            }
        ),
        Tool(
            name="deploy_preview",
            description="Deploy preview environment",
            inputSchema={
                "type": "object",
                "properties": {
                    "branch": {"type": "string"}
                }
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, args: dict) -> list[TextContent]:
    if name == "run_tests":
        cmd = ["pytest", args.get("path", "."), "-v"]
        if args.get("coverage"):
            cmd += ["--cov", "--cov-report=term-missing"]
        result = subprocess.run(cmd, capture_output=True, text=True, cwd="/project")
        return [TextContent(type="text", text=result.stdout + result.stderr)]
    
    elif name == "run_linter":
        result = subprocess.run(["ruff", "check", "."], capture_output=True, text=True, cwd="/project")
        return [TextContent(type="text", text=result.stdout + result.stderr)]
    
    # ...
```

---

## 7. CLI & Automation: Unix Philosophy

### 7.1 Piping & Scripting

```bash
# Pipe logs into Claude Code
cat error.log | claude "analyze this error and suggest fix"

# Run in CI/CD
claude "review PR #123 for security issues" --output json > review.json

# Chain with other tools
git diff main..feature | claude "summarize changes and identify risks"

# Non-interactive mode for automation
claude -p "fix all lint errors" --non-interactive
```

### 7.2 CI/CD Integration

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Claude Code
        uses: anthropic/claude-code-action@v1
        with:
          api_key: ${{ secrets.ANTHROPIC_API_KEY }}
      - name: Run Review
        run: |
          claude -p "
            Review this PR for:
            1. Security vulnerabilities
            2. Performance issues
            3. Code style violations
            4. Missing tests
            Post findings as PR comments
          " --non-interactive
```

### 7.3 Slack Integration

```bash
# Mention @Claude in Slack with bug report
# Gets PR back automatically

# Setup: Install Claude Code Slack app
# Configure: /claude connect <repo>
```

---

## 8. Multi-Agent Orchestration (Dynamic Workflows)

### 8.1 Subagent Delegation

```python
# Claude Code can spawn multiple subagents in parallel
# Each gets its own context, tools, and checkpointer

# Example: "A harness for every task" (Claude Code blog)
async def parallel_feature_work(task):
    # Decompose into independent sub-tasks
    sub_tasks = [
        "Write backend API for user auth",
        "Write frontend login components", 
        "Write integration tests",
        "Update API documentation",
        "Add database migrations"
    ]
    
    # Spawn subagents (each with own context budget)
    subagents = [
        create_subagent(t, budget=10000, tools=[read, write, bash, test])
        for t in sub_tasks
    ]
    
    # Run in parallel
    results = await asyncio.gather(*[s.ainvoke() for s in subagents])
    
    # Aggregate
    return integrate_results(results)
```

### 8.2 When to Use Subagents

| Scenario | Subagent Strategy |
|----------|-------------------|
| **Large feature** | Decompose by layer (backend, frontend, tests, docs) |
| **Refactoring** | One subagent per module/file |
| **Bug investigation** | Parallel: reproduce, trace, fix, test |
| **Code review** | Multiple reviewers (security, perf, style) |

---

## 9. Decision Framework

```
NEED LOCAL TOOLS / AGENTIC CODING?
│
├─► Standardize tool access across AI apps?
│   └─► MCP (Model Context Protocol)
│       ├─ Local tools → stdio transport
│       ├─ Remote tools → HTTP/SSE
│       └─ Build once, use everywhere (Claude, ChatGPT, VS Code, Cursor)
│
├─► Full-featured coding agent?
│   └─► Claude Code
│       ├─ Surfaces: Terminal, VS Code, JetBrains, Desktop, Web
│       ├─ Customization: CLAUDE.md, Skills, Hooks
│       ├─ Memory: Auto + Manual (cross-session)
│       ├─ MCP: Native integration
│       ├─ Automation: CLI, CI/CD, Slack
│       └─ Multi-agent: Dynamic workflows, subagents
│
├─► Just need file/db access?
│   └─► MCP Filesystem/Postgres servers (minimal setup)
│
├─► Team workflow automation?
│   └─► Skills + Hooks + CI/CD integration
│
├─► Cross-session context?
│   └─► Auto-memory + CLAUDE.md
│
└─► Complex multi-repo work?
    └─► Subagent orchestration + MCP Git/GitHub servers
```

---

## 10. Quick Reference: Complete Setup

```bash
# ============================================
# QUICK START: LOCAL TOOLS ECOSYSTEM
# ============================================

# 1. INSTALL CLAUDE CODE
brew install claude-code@latest    # macOS
winget install Anthropic.ClaudeCode # Windows
npm i -g @anthropic/claude-code     # Cross-platform

# 2. INIT PROJECT
cd your-project
claude  # Creates .claude/ directory

# 3. ADD CLAUDE.md (project instructions)
cat > CLAUDE.md << 'EOF'
# Project Instructions
- Type hints required
- pytest for testing
- ruff for linting
EOF

# 4. CONFIGURE MCP SERVERS (.mcp.json)
cat > .mcp.json << 'EOF'
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
EOF

# 5. ADD SKILLS (.claude/skills/)
mkdir -p .claude/skills
cat > .claude/skills/review-pr.md << 'EOF'
---
name: review-pr
description: Review PR for security, performance, style
---
# Review PR Skill
1. Fetch diff: gh pr diff $1
2. Analyze for security, perf, style
3. Run tests: pytest
4. Run linter: ruff check .
5. Post findings
EOF

# 6. ADD HOOKS (.claude/settings.json)
cat > .claude/settings.json << 'EOF'
{
  "hooks": {
    "post-edit": ["ruff format {file}", "ruff check --fix {file}"],
    "pre-commit": ["pytest -x", "mypy ."]
  }
}
EOF

# 7. START CODING
claude "add OAuth2 login with Google and GitHub"
```

---

## 11. Production Checklist

### Pre-Launch

- [ ] **Claude Code installed** on all developer machines
- [ ] **CLAUDE.md** committed with project standards
- [ ] **MCP servers configured** (.mcp.json committed)
- [ ] **Skills created** for common workflows
- [ ] **Hooks configured** for auto-format, pre-commit
- [ ] **CI/CD integration** for PR review
- [ ] **Slack integration** for team notifications
- [ ] **Auto-memory** verified working across sessions
- [ ] **Token budgets** set for subagents

### Ongoing

- [ ] **Weekly** skill review & updates
- [ ] **Monthly** MCP server audit
- [ ] **Quarterly** CLAUDE.md refresh
- [ ] **Per-session** hook effectiveness check

---

## 12. Resources & References

### Official Documentation
- **MCP Spec:** https://modelcontextprotocol.io/
- **MCP Getting Started:** https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro
- **Claude Code Docs:** https://code.claude.com/docs/en/overview
- **Claude Code Quickstart:** https://code.claude.com/docs/en/quickstart.md
- **MCP Servers Registry:** https://github.com/modelcontextprotocol/servers

### Key Pages (Claude Code)
- **Settings:** https://code.claude.com/docs/en/settings.md
- **Memory (CLAUDE.md):** https://code.claude.com/docs/en/memory.md
- **Skills:** https://code.claude.com/docs/en/skills.md
- **Hooks:** https://code.claude.com/docs/en/hooks.md
- **Workflows:** https://code.claude.com/docs/en/workflows.md
- **MCP Integration:** https://code.claude.com/docs/en/mcp.md
- **CI/CD:** https://code.claude.com/docs/en/ci-cd.md
- **Slack:** https://code.claude.com/docs/en/slack.md

### Blog & Tutorials
- **Dynamic Workflows:** https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
- **MCP Announcement:** https://modelcontextprotocol.io/blog

---

## 13. Summary: Local Tools Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Protocol** | MCP (Model Context Protocol) | Standardized tool access |
| **Transport** | stdio (local), HTTP/SSE (remote) | Communication |
| **Servers** | Filesystem, Git, GitHub, DB, Docker, K8s | External capabilities |
| **Client** | Claude Code (Terminal, IDE, Desktop, Web) | Agentic coding |
| **Customization** | CLAUDE.md, Skills, Hooks | Project-specific behavior |
| **Memory** | Auto-memory + Manual notes | Cross-session persistence |
| **Automation** | CLI, CI/CD, Slack | Team workflows |
| **Orchestration** | Subagents, Dynamic workflows | Parallel multi-agent |
