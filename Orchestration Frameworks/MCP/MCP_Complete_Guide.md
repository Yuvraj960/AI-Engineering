# The Model Context Protocol (MCP) — Complete Study Guide

## Table of Contents

1. [What MCP Is and Why It Exists](#1-what-mcp-is-and-why-it-exists)
2. [The Mental Model: USB-C for AI Applications](#2-the-mental-model-usb-c-for-ai-applications)
3. [Architecture: Hosts, Clients, and Servers](#3-architecture-hosts-clients-and-servers)
4. [The Wire Protocol: JSON-RPC 2.0, `_meta`, and Discovery](#4-the-wire-protocol-json-rpc-20-_meta-and-discovery)
5. [Versioning and Compatibility](#5-versioning-and-compatibility)
6. [Server Primitives: Tools, Resources, Prompts](#6-server-primitives-tools-resources-prompts)
7. [Client Features: Elicitation (and the Deprecated Roots & Sampling)](#7-client-features-elicitation-and-the-deprecated-roots--sampling)
8. [Transports: stdio and Streamable HTTP](#8-transports-stdio-and-streamable-http)
9. [Building an MCP Server](#9-building-an-mcp-server)
10. [Building an MCP Client](#10-building-an-mcp-client)
11. [Connecting Servers to Hosts (Local & Remote)](#11-connecting-servers-to-hosts-local--remote)
12. [Testing and Debugging: MCP Inspector](#12-testing-and-debugging-mcp-inspector)
13. [Client Best Practices at Scale](#13-client-best-practices-at-scale)
14. [Authorization (OAuth 2.1) and Security Best Practices](#14-authorization-oauth-21-and-security-best-practices)
15. [The Wider Ecosystem: SDKs, Registry, Extensions, SEPs, Governance](#15-the-wider-ecosystem-sdks-registry-extensions-seps-governance)
16. [DeepLearning.AI Course — Full Walkthrough](#16-deeplearningai-course--full-walkthrough)
17. [Codebasics YouTube Tutorial — Full Walkthrough](#17-codebasics-youtube-tutorial--full-walkthrough)
18. [MCP at the Intersection of Agentic AI and AI Engineering](#18-mcp-at-the-intersection-of-agentic-ai-and-ai-engineering)
19. [Suggested Learning Path and Projects](#19-suggested-learning-path-and-projects)
20. [Quick Reference](#20-quick-reference)
21. [Glossary](#21-glossary)
22. [Sources Read for This Guide](#22-sources-read-for-this-guide)

---

## 1. What MCP Is and Why It Exists

**MCP (Model Context Protocol) is an open-source standard for connecting AI applications to external systems.** Using MCP, AI applications like Claude or ChatGPT can connect to:

- **Data sources** — local files, databases
- **Tools** — search engines, calculators, APIs
- **Workflows** — specialized prompts

…enabling them to access key information and perform tasks.

**History.** MCP was developed by Anthropic and launched in **November 2024**. It originated as an internal project to extend Claude Desktop so it could interact with local file systems and other external systems. Anthropic recognized the protocol they built was useful to many AI applications with similar needs, so they published the specification and opened its development to the open-source community. Since then the ecosystem has grown rapidly: it is **model-agnostic**, completely open source, and supported across a wide range of clients.

**The problem it solves (the DeepLearning.AI framing).** Everything you can do with MCP, you *could* do without it — but before MCP, every team kept reinventing the wheel: many different AI applications talking to similar data sources, each written in a different way. Where do you store your tools? Where do you store custom prompts? Where do you put the data-access layer and authentication logic? MCP standardizes the connection so that:

- **Developers** — less development time and complexity when building or integrating with AI applications/agents.
- **AI applications / agents** — access to an ecosystem of data sources, tools, and apps, which enhances capabilities and the end-user experience.
- **End-users** — more capable AI applications that can access user data and take actions on the user's behalf.

MCP also cleanly **shifts the burden of responsibility and separates concerns**: tool authors build servers once; application authors host clients; users pick which servers to trust.

**What can MCP enable (official examples)?**

- Agents accessing your Google Calendar and Notion, acting as a personalized AI assistant.
- Claude Code generating an entire web app from a Figma design.
- Enterprise chatbots connecting to multiple databases across an organization so users can analyze data via chat.
- AI models creating 3D designs in Blender and printing them on a 3D printer.

**Broad ecosystem support.** MCP is an open protocol supported across clients and servers: AI assistants like **Claude** and **ChatGPT**, development tools like **Visual Studio Code**, **Cursor**, **MCPJam**, and many others — "build once, integrate everywhere."

**Scope (important).** MCP includes three projects: the **MCP Specification**, the **language SDKs**, and **MCP development tools** (like the MCP Inspector). MCP focuses *solely on the protocol for context exchange* — it does **not** dictate how AI applications use LLMs or manage the provided context.

---

## 2. The Mental Model: USB-C for AI Applications

The official analogy: **think of MCP like a USB-C port for AI applications.** Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems.

| USB-C world | MCP world |
|---|---|
| One port shape on every laptop | One protocol spoken by every AI app (host) |
| Any peripheral (display, disk, keyboard) just works | Any MCP server (GitHub, filesystem, database) just works |
| Device manufacturers build for the port, not for each laptop | Server authors build once for MCP, not per application |
| Plugging in is safe and standardized | Connections are discoverable, permissioned, and versioned |

Before MCP: **N AI apps × M data sources = N×M custom integrations.**
After MCP: **N apps + M servers, each built once against one protocol.**

The DeepLearning.AI course demonstrates the payoff concretely: if you build a research-assistant agent that should interact with GitHub repos, read Google Drive notes, and store a summary on your local file system, you *don't* write your own custom LLM tools for each — you connect your agent to the **GitHub, Google Drive, and Filesystem MCP servers**, which provide the tool/API-call definitions *and* handle the tool execution.

---

## 3. Architecture: Hosts, Clients, and Servers

MCP follows a **client-server architecture**:

```
┌──────────────────────────── HOST (LLM application) ────────────────────────────┐
│  e.g. Claude Desktop, Claude.ai, ChatGPT, VS Code, Cursor, your own chatbot    │
│                                                                                 │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐      Clients maintain a 1-to-1        │
│  │Client 1 │   │Client 2 │   │Client 3 │      connection with ONE server each │
│  └────┬────┘   └────┬────┘   └────┬────┘                                         │
└───────┼─────────────┼─────────────┼──────────────────────────────────────────────┘
        │ stdio or    │             │ Streamable HTTP (JSON-RPC 2.0 messages)
        ▼             ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ Server A│   │ Server B│   │ Server C│      Lightweight programs that expose
   │(SQLite) │   │(GitHub) │   │ (fetch) │      specific capabilities via the
   └─────────┘   └─────────┘   └─────────┘      protocol
```

**The three roles (definitions that matter):**

- **Host** — the LLM application the user interacts with (Claude Desktop, Claude AI, your chatbot). The host is responsible for storing and maintaining all the clients and the connections to MCP servers. Hosts are LLM applications that want to access data through MCP.
- **Client** — the protocol-level component *inside* the host. Each client maintains a strict **1-to-1 connection** with exactly one MCP server. Understanding the host-vs-client distinction is important: the *host* is what users interact with; *clients* are what enable server connections.
- **Server** — a lightweight program that exposes specific capabilities (tools, resources, prompts) through the protocol. A server can be a **subprocess launched by the client and running locally** (stdio), or an **independent process running remotely** (Streamable HTTP).

**Division of labor** (from the course): the **client's job** is to find resources and find tools; the **server's job** is to expose that information to the client. The host decides what to do with the LLM.

The course's architecture lesson demos this live: Claude Desktop (host) connected to an **SQLite MCP server** exposing tools, resources, and prompts — then later lessons build the client and even a host containing multiple clients from scratch in Python, so you see there is no magic, just protocol messages.

---

## 4. The Wire Protocol: JSON-RPC 2.0, `_meta`, and Discovery

### JSON-RPC 2.0 underneath

MCP's data layer uses **JSON-RPC 2.0** as its underlying RPC protocol. Clients and servers send requests to each other and respond accordingly; **notifications** are used when no response is required. Every request/response carries a unique `id` for correlation.

### Statelessness and per-request metadata (2026-07-28 spec)

MCP (current spec) is **stateless**: there are no protocol-level sessions. Instead:

- **Every request** declares the protocol version it is using via the `io.modelcontextprotocol/protocolVersion` key in its **`_meta`** field, and the server accepts or rejects each request independently.
- Clients should identify themselves via `io.modelcontextprotocol/clientInfo` (name + version) unless configured not to.
- Clients declare what they can support via `io.modelcontextprotocol/clientCapabilities`.
- On Streamable HTTP, the protocol version is also carried in the **`MCP-Protocol-Version` header**.

A request missing a required `_meta` field is rejected with JSON-RPC error **`-32602` (Invalid params)**. If the server needs a capability the request's `clientCapabilities` didn't declare (e.g. elicitation), it returns **`MissingRequiredClientCapabilityError` (`-32021`)** naming the missing capabilities.

### Discovery: `server/discover`

Servers advertise their supported versions, capabilities, and identity through the mandatory **`server/discover`** request, which clients *may* send before any other request. Calling it is optional (any request can be sent directly, handling version errors if they come back), but it's the convenient way to fetch identity + capabilities + supported versions in one round trip.

**Discover request:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "server/discover",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": { "name": "example-client", "version": "1.0.0" },
      "io.modelcontextprotocol/clientCapabilities": { "elicitation": {} }
    }
  }
}
```

**Discover response:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "supportedVersions": ["2026-07-28"],
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": {}
    },
    "_meta": {
      "io.modelcontextprotocol/serverInfo": { "name": "example-server", "version": "1.0.0" }
    },
    "ttlMs": 3600000,
    "cacheScope": "public"
  }
}
```

**How to read this:**

- `supportedVersions` — versions the server accepts. If it doesn't support the requested version it rejects with `UnsupportedProtocolVersionError` (**`-32022`**) listing the versions it does support, and the client retries with a mutually supported version.
- `"tools": {"listChanged": true}` — server supports the tools primitive and can honor a `toolsListChanged` filter in `subscriptions/listen`; clients that request that filter receive `notifications/tools/list_changed` when the tool list changes.
- `"resources": {}` — server supports `resources/list` and `resources/read`.
- `ttlMs` / `cacheScope` — caching hints: how long (ms) the response may be reused and by whom (see the spec's caching utility).

### Change notifications (opt-in subscriptions)

Listings are dynamic. To hear about changes, the client opens a long-lived notification stream with a **`subscriptions/listen`** request naming the event types it wants:

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "subscriptions/listen",
  "params": {
    "_meta": { "...": "standard per-request fields" },
    "notifications": { "toolsListChanged": true }
  }
}
```

Why this matters: **dynamic environments** (tools come and go with server state, dependencies, permissions), **efficiency** (no polling), **consistency** (accurate capability info), and **real-time collaboration**.

### Layer summary (official "Concepts of MCP")

- **Data layer**: Discovery (`server/discover`), server features (tools/resources/prompts), client features (elicitation), utility features (notifications, progress).
- Each primitive has methods for discovery (`*/list`), retrieval (`*/get`), and where applicable execution (`tools/call`). A client lists tools first (`tools/list`), then executes (`tools/call`).

---

## 5. Versioning and Compatibility

- Protocol versions are **string identifiers in `YYYY-MM-DD` format**, indicating the last date backwards-incompatible changes were made.
- **Revisions**: **Draft** (in progress, not ready for consumption) → **Current** (ready for use, may receive backwards-compatible changes) → **Final** (past, complete, frozen).
- The **current protocol version is 2026-07-28**. Final (archived) versions remain published on the site: `2025-11-25`, `2025-06-18`, `2025-03-26`, `2024-11-05` — useful because much community content (including the DeepLearning.AI course, May 2025) targets the earlier specs.
- **Feature states**: individual features may additionally be marked **Deprecated** under the feature-lifecycle-and-deprecation policy: the feature remains in the specification but is scheduled for removal, with a documented migration path, and stays for at least twelve months under the standard policy. Examples deprecated as of 2026-07-28: **Roots, Sampling, and protocol-level Logging** (see §7).
- **Negotiation**: every request declares its version in `_meta` (and on Streamable HTTP, in the `MCP-Protocol-Version` header); the server accepts or rejects each request independently; version mismatch → `-32022` with the supported list, and the client retries.

**Practical takeaway for you:** when reading tutorials, check which spec era they target. Older material shows an `initialize` handshake, sessions, and heavy use of roots/sampling; current-era code uses `server/discover`, per-request `_meta`, statelessness, and deprecated primitives replaced by direct alternatives. The concepts transfer; the method names and lifecycle details shifted.


---

## 6. Server Primitives: Tools, Resources, Prompts

Servers expose three main primitives. The control question ("who decides when it's used?") is the key differentiator:

| Primitive | What it is | Examples | Who controls it |
|---|---|---|---|
| **Tools** | Functions the LLM can actively call; the **model** decides when to use them based on user requests. Can write to databases, call external APIs, modify files, trigger logic. | Search flights, send messages, create calendar events | **Model** |
| **Resources** | Passive, **read-only** data for context: file contents, database schemas, API docs. | Retrieve documents, access knowledge bases, read calendars | **Application** (host decides what to attach) |
| **Prompts** | Pre-built instruction templates that tell the model to work with specific tools/resources. | Plan a vacation, summarize my meetings, draft an email | **User** (explicit invocation) |

### Tools

Each tool defines a specific operation with typed inputs and outputs. A tool descriptor contains:

- **`name`** — unique identifier in the server's namespace; follow a clear naming pattern (e.g. `calculator_arithmetic` rather than just `calculate`). Tool-name format is standardized (SEP-986); a conforming name must not fail client validation.
- **`title`** — human-readable display name.
- **`description`** — detailed explanation of what the tool does and when to use it. **This is critical:** it is how the LLM chooses the tool. The codebasics tutorial shows Claude mapping "4th July" → `2025-07-04` purely because the docstring specified the date format.
- **`inputSchema`** — a JSON Schema of expected parameters (required/optional, types).

`tools/list` returns the tool array with caching hints (`ttlMs`, `cacheScope`); `tools/call` executes one and returns a `content` array allowing rich multi-format responses (text, images, resources, ...). **Errors are results, not failures**: a failing tool returns a successful JSON-RPC response with `isError: true` rather than a protocol error — clients pass that on so the model can read the message and try something else.

Example tool descriptor from `tools/list`:

```json
{
  "name": "weather_current",
  "title": "Weather Information",
  "description": "Get current weather information for any location worldwide",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "City name, address, or coordinates (latitude,longitude)" },
      "units": { "type": "string", "enum": ["metric", "imperial", "kelvin"], "default": "metric" }
    },
    "required": ["location"]
  }
}
```

And the corresponding `tools/call`:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "weather_current",
    "arguments": { "location": "San Francisco", "units": "imperial" },
    "_meta": { "...": "standard per-request fields" }
  }
}
```

**How this plays out in an AI application** (official pseudo-code):

```python
async def handle_tool_call(conversation, tool_name, arguments):
    client = app.find_mcp_client_for_tool(tool_name)
    result = await client.call_tool(tool_name, arguments)
    conversation.add_tool_result(result.content)
```

### Resources

Read-only context exposed through URIs. **Resource Templates** add metadata (title, description, expected MIME type) making them discoverable and self-documenting. The travel-planning example: calendar data (`calendar://events/2024`), travel documents (`file:///Documents/Travel/passport.pdf`), previous itineraries (`trips://history/barcelona-2023`).

**User interaction model:** the protocol does NOT mandate UI patterns — hosts may use resource pickers with previews, smart suggestions from conversation context, bulk selection, or integration into existing file browsers.

### Prompts

Reusable templates with arguments. Protocol operations: `prompts/list` (discover) and `prompts/get` (retrieve full definition with arguments).

```json
{
  "name": "plan-vacation",
  "title": "Plan a vacation",
  "description": "Guide through vacation planning process",
  "arguments": [
    { "name": "destination", "type": "string", "required": true },
    { "name": "duration", "type": "number", "description": "days" },
    { "name": "budget", "type": "number", "required": false },
    { "name": "interests", "type": "array", "items": { "type": "string" } }
  ]
}
```

**User interaction model:** prompts are user-controlled, requiring explicit invocation (e.g., slash-commands in Claude Desktop). One prompt can orchestrate multiple tools and resources into a workflow — e.g., a Google Drive server might expose a prompt that searches, reads, and summarizes, so the user does not need to write the perfect multi-step prompt themselves.

### Server utilities

- **Pagination** (`cursor` param on list requests) for large tool/resource/prompt sets.
- **Completion** — argument autocompletion for prompts/resources.
- **Caching** — `ttlMs`/`cacheScope` hints on list results.
- **Logging** — protocol-level logging (`notifications/message`) is **deprecated** as of 2026-07-28; use stderr (stdio) / OpenTelemetry / your own log aggregation instead.

---

## 7. Client Features: Elicitation (and the Deprecated Roots & Sampling)

Clients (hosts) expose features **to servers**:

| Feature | What it does | Status (2026-07-28) |
|---|---|---|
| **Elicitation** | Servers request structured input from the user mid-operation (seat preference, room type, contact number to finalize a booking). | **Active** |
| **Roots** | Client tells server which directories/files are in scope. | **Deprecated** — pass paths via tool parameters, resource URIs, or server config instead |
| **Sampling** | Server asks the client's LLM for a completion (agentic workflow, client stays in control of permissions). | **Deprecated** — servers should integrate directly with LLM provider APIs |

### Elicitation and Multi Round-Trip Requests (MRTR)

Elicitation follows the **Multi Round-Trip Requests** pattern. When a server needs user input while processing a request such as `tools/call`, it responds with an **`InputRequiredResult`** whose `inputRequests` field carries one or more **`elicitation/create`** requests. The client gathers the input (rendering a form, with clear context about which server is asking and why) and **retries the original request**, attaching collected `inputResponses` and echoing back any `requestState`.

Flow: server needs data → returns `InputRequiredResult` → client shows UI → user provides input → client retries original request with `inputResponses` → server completes with the new context.

**Privacy rules (important):** servers must NOT use form mode to request sensitive information (passwords, API keys, access tokens, payment credentials). Those belong in **URL mode**, which keeps the data out-of-band so it never passes through the client or the LLM context. Clients warn about suspicious requests and let users review form data before sending.

### Why Roots and Sampling were deprecated

- **Roots** were filesystem-scope hints (`file:///Users/agent/travel-planning` style URIs). Migration: pass directories/files as tool parameters, resource URIs, or server configuration.
- **Sampling** let a server request LLM completions via `sampling/createMessage` — useful for server authors who wanted LLM access without shipping an LLM SDK, keeping the client in control. Migration: servers integrate directly with LLM provider APIs. (SEP-1577 explores sampling-with-tools; the deprecation reflects the ecosystem moving to direct provider APIs.)

---

## 8. Transports: stdio and Streamable HTTP

Two standard transports carry the JSON-RPC messages:

### stdio (local)

- The client **launches the server as a subprocess** and communicates over the process's stdin/stdout. This is the "server as a subprocess launched by the client" model.
- Simplest possible deployment — nothing to host, no ports, no auth needed for local use.
- **The golden rule: never write to stdout in a stdio server.** stdout *is* the protocol channel; stray `print()` output corrupts the JSON-RPC stream and breaks the server. Log to **stderr** instead. (Python: use `logging`, which defaults to stderr; TypeScript: `console.error()`; C#: avoid `Console.WriteLine`.)
- Environment variables are NOT fully inherited — pass what you need via the config's `env` key.
- Use **absolute paths** everywhere in client configs; the working directory of a client-launched server is undefined.

### Streamable HTTP (remote)

- The server runs as an independent network process; clients send JSON-RPC over HTTP POST, with responses and server-initiated messages delivered via **SSE (Server-Sent Events)** streams.
- Carries the protocol version in the **`MCP-Protocol-Version` header** (SEP-2243 standardizes headers).
- Enables remote deployment, OAuth authorization, one deployment serving all users.
- Debugging uses standard HTTP tooling (curl, browser DevTools); stderr is not visible to the client, so use server-side log aggregation or OpenTelemetry.

The docs also describe a "protocol eras" concept — legacy (pre-2026-07-28) vs modern MCP — and tools like the Inspector negotiate both. (SEP-2575 "Make MCP Stateless", SEP-2567 sessionless MCP via state handles, and SEP-1699 SSE polling cover the evolution from the older HTTP+SSE transport.)

---

## 9. Building an MCP Server

### The official quickstart (weather server, Python)

Requirements: Python 3.10+, Python MCP SDK **2.0.0+**, and `uv` (the fast Python package manager from Astral):

```bash
# install uv (Windows)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# project setup
uv init weather
cd weather
uv venv
.venv\Scripts\activate        # Windows (macOS/Linux: source .venv/bin/activate)
uv add "mcp[cli]"
# create weather.py
```

The server (note: the current-era SDK uses the `MCPServer` class; the `FastMCP` class from the DeepLearning.AI course is the same high-level decorator interface under its earlier name):

```python
from typing import Any
import httpx2
from mcp.server import MCPServer

# Initialize MCPServer
mcp = MCPServer("weather")

NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"

async def make_nws_request(url: str) -> dict[str, Any] | None:
    """Make a request to the NWS API with proper error handling."""
    headers = {"User-Agent": USER_AGENT, "Accept": "application/geo+json"}
    async with httpx2.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None

@mcp.tool()
async def get_alerts(state: str) -> str:
    """Get weather alerts for a US state.

    Args:
        state: Two-letter US state code (e.g. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)
    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."
    if not data["features"]:
        return "No active alerts for this state."
    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """Get weather forecast for a location."""
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)
    if not points_data:
        return "Unable to fetch forecast data for this location."
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)
    if not forecast_data:
        return "Unable to fetch detailed forecast."
    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:  # Only show next 5 periods
        forecast = (
            f'{period["name"]}:\n'
            f'Temperature: {period["temperature"]}°{period["temperatureUnit"]}\n'
            f'Wind: {period["windSpeed"]} {period["windDirection"]}\n'
            f'Forecast: {period["detailedForecast"]}'
        )
        forecasts.append(forecast)
    return "\n---\n".join(forecasts)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**The key insight:** the class uses **Python type hints and docstrings to auto-generate the tool definitions** (name, description, inputSchema). Writing good docstrings *is* writing your tool spec.

Run with `uv run weather.py`. Then register in Claude Desktop's `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": ["--directory", "C:\\ABSOLUTE\\PATH\\TO\\weather", "run", "weather.py"]
    }
  }
}
```

**TypeScript equivalent:** `McpServer` from `@modelcontextprotocol/server` + `server.registerTool("get_alerts", { description, inputSchema: z.object({...}) }, handler)` using **zod** schemas. SDKs also exist for Java/Kotlin (Gradle + shadow JAR), C# (`dotnet add package ModelContextProtocol`, `AddMcpServer().WithStdioServerTransport().WithToolsFromAssembly()`), and more.

### Logging rules (memorize these)

- **stdio servers: NEVER write to stdout** — it corrupts the JSON-RPC stream. `print()` (Python), `console.log()` (TS), `Console.WriteLine()` (C#) are all banned in stdio servers.
- Use `logging.getLogger(__name__)` (writes to stderr) or `console.error()`.
- HTTP servers: stdout logging is fine (doesn't interfere with HTTP responses).

### Build with Agent Skills (AI-assisted development)

The docs include a meta-layer for agentic development: install the **`mcp-server-dev`** plugin (Claude Code: `/plugin marketplace add anthropics/claude-plugins-official`, `/plugin install mcp-server-dev`) to get three composing skills your AI coding agent reads:

- **`build-mcp-server`** — entry point; interrogates the use case, picks a deployment model and tool-design pattern, routes to specialized skills.
- **`build-mcp-app`** — adds interactive UI widgets (forms, pickers, dashboards) rendered inline in chat (MCP Apps).
- **`build-mcpb`** — packages a local stdio server with its runtime as a `.mcpb` archive so users can install it without Node/Python (MCP Bundles).

Discovery questions the skill asks: what it connects to, who uses it, how big the action surface is, what user interaction is needed (plain text / elicitation / rich UI widgets), and upstream auth (API keys / OAuth / none). Deployment paths: **Remote Streamable HTTP** (default for cloud APIs; scaffolds for Cloudflare Workers, Express/FastMCP), **MCP Apps** (when elicitation's flat forms don't fit), **MCPB bundles** (when the server must touch the user's machine), **local stdio** (prototyping with an upgrade path to MCPB).

---

## 10. Building an MCP Client

The official quickstart builds an LLM-powered chatbot client (Python, `uv add mcp anthropic python-dotenv`). Current-era SDK shape:

```python
import asyncio, sys
from mcp import Client, StdioServerParameters
from mcp.client.stdio import stdio_client
from mcp_types import TextContent
from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv()
MODEL = "claude-opus-5"
anthropic = Anthropic()

def server_params(server_script_path: str) -> StdioServerParameters:
    """Describe the subprocess that runs an MCP server (.py or .js)."""
    if server_script_path.endswith(".py"):
        command = "python"
    elif server_script_path.endswith(".js"):
        command = "node"
    else:
        raise ValueError("Server script must be a .py or .js file")
    return StdioServerParameters(command=command, args=[server_script_path])

async def process_query(client: Client, query: str) -> str:
    """Process a query using Claude and available tools."""
    messages = [{"role": "user", "content": query}]

    tool_list = await client.list_tools()
    available_tools = [{
        "name": tool.name,
        "description": tool.description,
        "input_schema": tool.input_schema
    } for tool in tool_list.tools]

    response = anthropic.messages.create(
        model=MODEL, max_tokens=1000, messages=messages, tools=available_tools
    )

    final_text, tool_results = [], []
    for content in response.content:
        if content.type == "text":
            final_text.append(content.text)
        elif content.type == "tool_use":
            result = await client.call_tool(content.name, content.input)
            final_text.append(f"[Calling tool {content.name} with args {content.input}]")
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": content.id,
                "content": "\n".join(b.text for b in result.content
                                     if isinstance(b, TextContent)),
                "is_error": result.is_error
            })

    if tool_results:
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        response = anthropic.messages.create(
            model=MODEL, max_tokens=1000, messages=messages, tools=available_tools
        )
        for content in response.content:
            if content.type == "text":
                final_text.append(content.text)

    return "\n".join(final_text)

async def main() -> None:
    # The async with block IS the connection lifecycle:
    # entering launches the server subprocess and agrees a protocol version;
    # leaving disconnects and shuts the subprocess down.
    async with Client(stdio_client(server_params(sys.argv[1]))) as client:
        tool_list = await client.list_tools()
        print("\nConnected to server with tools:",
              [t.name for t in tool_list.tools])
        # ... interactive chat loop ...

asyncio.run(main())
```

**Key concepts to internalize:**

- `StdioServerParameters` is *configuration*, not a connection. `stdio_client()` turns it into a transport; `Client` opens it. The `async with` block **is** the entire connection lifecycle — no connect/close pair, nothing to clean up by hand.
- `call_tool` returns a `CallToolResult`; `content` is a list of typed blocks. **A failing tool does not raise** — it answers with `is_error` set; pass that flag back so Claude can read the error and try something else.
- The **agentic loop**: list tools → send user query + tool descriptions to the LLM → LLM decides which tools to use → client executes tool calls through the server → results go back to the LLM → LLM produces the natural-language answer.

**Best practices:** check `result.is_error` rather than expecting raises; let `async with` own the connection; store API keys in `.env` (never commit); validate tool names against the spec's format; first response may take up to 30s (server init + LLM + tool execution) — don't interrupt. Common errors: `FileNotFoundError` → check server path; `Connection refused` → server running?; `Tool execution failed` → missing env vars; `Timeout` → raise `read_timeout_seconds`.

TypeScript equivalent: `Client` from `@modelcontextprotocol/client` + `StdioClientTransport` + `@anthropic-ai/sdk`, same loop. C#: `McpClient.CreateAsync(new StdioClientTransport(...))` + `ListToolsAsync()`.

---

## 11. Connecting Servers to Hosts (Local & Remote)

### Local servers via Claude Desktop config

Config file locations:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Open via Settings → Developer → **Edit Config**. Register the reference filesystem server:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\username\\Desktop",
        "C:\\Users\\username\\Downloads"
      ]
    }
  }
}
```

Reading the config: `command` = executable to run, `args` = command-line arguments (the trailing paths are the **directories the server is allowed to access**), optional `env` = environment variables for the subprocess. Fully quit and restart Claude Desktop after editing. Tools appear under the "Add files, connectors, and more" icon → Connectors.

**Security note from the docs:** the server runs with your user permissions — only grant access to directories you're comfortable with Claude reading and modifying. Every file operation still requires explicit user approval (human-in-the-loop).

**Troubleshooting checklist:** restart fully; validate JSON syntax; use absolute paths; check logs (`%APPDATA%\Claude\logs\mcp.log` and `mcp-server-SERVERNAME.log` on Windows; `~/Library/Logs/Claude` on macOS); run the server command manually in a terminal to see startup errors; on Windows an `ENOENT ... ${APPDATA}` error means you must add the expanded `APPDATA` to the server's `env` block; npm may need a global install (`npm install -g npm`).

### Remote servers via Custom Connectors

Remote MCP servers are hosted on the internet and accessible from any MCP client — ideal for web-based applications, ease-of-use integrations, and anything requiring server-side processing or authentication. In Claude: Settings → Connectors → **Add custom connector** → paste the server URL (e.g. `https://example-server.modelcontextprotocol.io/mcp`) → complete authentication (commonly OAuth, API keys, or username/password) → the server's resources and prompts become attachable in conversations → configure **tool permissions** (enable/disable specific tools, set usage limits) in the connector settings.

Best practices: verify server authenticity before connecting; only connect to trusted sources; review requested permissions; organize connectors by purpose; regularly remove unused ones.

---

## 12. Testing and Debugging: MCP Inspector

The **MCP Inspector** (`@modelcontextprotocol/inspector`) is the reference developer tool for testing and debugging MCP servers — your **first stop** when anything misbehaves. It ships as a single package providing **three clients behind one binary**:

- **Web client** — graphical, tab-by-tab walkthrough (connect, tools, prompts, resources, notifications stream).
- **CLI client** — scripting: methods, output formats, exit codes, CI recipes.
- **TUI client** — terminal UI with tabs and keyboard navigation.

All three share one core: same transports, same config files, same OAuth state on disk, same **protocol-era negotiation** (legacy vs. modern 2026-07-28 behavior). Flags are split between the launcher and per-client flags (`--catalog`, `--config`, `--server-url`, `--transport`, `--method`, OAuth flags) — the "Configuration and flags" page catalogs them by owner. Recipes cover transports, importing configs, reviewing MCP Apps, Docker, and network hosting.

**Debugging workflow** (from the debugging guide):

1. **MCP Inspector** — connect to stdio or Streamable HTTP servers, invoke tools/prompts/resources, watch the notification stream.
2. **Server logging** — stderr for stdio (captured by the host automatically); OpenTelemetry or your own aggregation for HTTP. Protocol-level `notifications/message` logging is deprecated.
3. **Client developer tools** — most hosts expose logs and connection state (Claude Desktop's `mcp.log` etc.).

**Common issues and fixes:**

- **Working directory** — client-launched servers get an undefined cwd; always use absolute paths in configs and `.env` files.
- **Environment variables** — stdio servers inherit only a limited subset; pass what you need via the config's `env` key.
- **Startup failures** — bad executable path, invalid JSON, missing fields, permission problems → try an absolute `command` path.
- **Connection failures** — check client logs; verify the server process runs; test standalone with the Inspector; call `server/discover` to check protocol compatibility (`-32022` lists supported versions); verify `_meta` completeness (`-32602` invalid params = required `_meta` fields missing; `-32021` = server needed a client capability, e.g. elicitation, that wasn't declared).
- **Testing changes** — config changes: restart the client. Server code changes: fully quit and reopen the client (closing the window is not enough). Quick iteration: use the Inspector.

Log what matters: startup steps, resource access, tool execution, error conditions, performance metrics. Sanitize logs — never log credentials or PII.

---

## 13. Client Best Practices at Scale

From the official "Client Best Practices" guide — patterns for hosts connecting to many servers with hundreds or thousands of tools. Naive loading breaks down: every tool definition in context upfront **wastes tokens, increases latency, and degrades model performance**; passing large intermediate results through the model between sequential tool calls compounds it.

### Progressive tool discovery — controls *when* definitions enter context

- Host fetches tool definitions via `tools/list` as normal but **defers injecting them into context**.
- Host gives the model a lightweight **`search_tools` meta-tool**.
- Full definitions load **only as needed**.

The headline number from the docs: upfront loading can consume **~150,000 tokens** on definitions alone; progressive discovery uses **~2,000** by loading only what the task requires.

**When to switch:** implement a threshold as a percentage of the context window (recommended **1%–5%**); below it, load everything; above it, go progressive.

**Search strategies:** keyword (BM25/regex — simple, works well with descriptive names), embedding-based (vector similarity over descriptions — handles synonyms), subagent-based (a small fast model like Claude Haiku picks tools — works very well but costs more), hybrid (combine rankings; route by use case). Check your provider first: OpenAI and Anthropic both offer **built-in tool search** — prefer the platform's when available; build your own when you need specialized retrieval (domain-specific ranking, access-control filtering).

**The three-layer pattern (catalog → inspect → execute):**

```text
Layer 1 — Catalog:   search_tools({ query: "update salesforce record" })
                     → [{ name: "salesforce_updateRecord", description: "..." }, ...]
Layer 2 — Inspect:   get_tool_details({ name: "salesforce_updateRecord" })
                     → full inputSchema/outputSchema for ONE tool
Layer 3 — Execute:   tools/call with full knowledge of the interface
```

### Programmatic tool calling — controls *how* tools are invoked

Instead of the model making many round-trip tool calls and passing large intermediate results through its own context, the model **writes a short script** that calls typed tool-wrapper functions; the host executes it in a sandboxed broker environment. Requirements from the docs:

- **Network isolation** — the sandbox has no direct network access; all external communication flows through the host broker, which enforces authorization and access control.
- **No credential exposure** — API keys and tokens are held by the host; the generated code calls typed functions and the host adds auth when forwarding to servers.
- **Resource limits** — timeouts and memory limits on sandbox execution.
- **Output filtering** — validate and truncate sandbox console output before feeding it back to the model.
- **Cross-server data flow** — tool results from one server are **untrusted input** to another; the broker applies the same input-review policy to brokered calls as to direct ones; output truncation alone does not prevent exfiltration.
- **Error handling** — MCP tool errors arrive as results with `isError: true`; wrappers should convert to thrown exceptions so model-authored code can `try`/`catch`; if an uncaught error kills the script, surface it as the script's result so the model self-corrects (the model is responsible for reporting partial side effects already committed).

**The two patterns combine well:** discovery to identify tools → load schemas → one script calling multiple tools in a single execution pass. This minimizes *both* the token cost of definitions *and* of results, keeping context focused on reasoning.

---

## 14. Authorization (OAuth 2.1) and Security Best Practices

### Authorization in MCP

MCP authorization secures access to sensitive resources and operations exposed by servers. It doesn't invent a new identity system — it follows the conventions of **OAuth 2.1** (the draft consolidating OAuth 2.0 + security best practices: authorization-code + PKCE, refresh tokens, no implicit grant).

**The flow, step by step:**

1. **Initial handshake** — client tries to connect; server responds **`401 Unauthorized`** with a `WWW-Authenticate` header whose `resource_metadata` parameter points to a **Protected Resource Metadata (PRM)** document (RFC 9728) hosted at a predictable path on the MCP server.
2. **Authorization Server discovery** — client fetches the PRM, learns the authorization server(s); if multiple are listed, the client picks one. (RFC 8414 AS metadata; RFC 7591 dynamic client registration; RFC 8707 resource indicators.)
3. **OAuth 2.1 authorization code flow with PKCE** — user authenticates and consents; client receives `{ "access_token": "...", "refresh_token": "...", "token_type": "Bearer", "expires_in": 3600 }`.
4. The client presents the bearer token on subsequent requests; **servers MUST validate tokens** (signature, expiry, `aud` audience claim — the audience must be tied to the `resource` parameter the client passed, not a fixed value in production).

The docs include a full worked example standing up **Keycloak** as the authorization server and protected MCP servers in TypeScript (express + token introspection), Python (the `MCPServer` class publishes the PRM document, answers 401s with proper headers, and hands bearer tokens to a verifier you supply), and C#.

Extensions for machine-to-machine and enterprise cases: **OAuth Client Credentials** extension (client-credentials flow for service-to-service), **Enterprise-Managed Authorization** (centralized access control via enterprise identity providers).

### Security Best Practices — the attack catalog

This is one of the most valuable pages in the docs. Know these attack names:

- **Confused Deputy Problem** — a malicious server or resource tricks the privileged MCP client into misusing its authority on the attacker's behalf. Mitigation: the MCP-level consent page MUST clearly show which server is requesting which action on which resource before tool use.
- **SSRF (Server-Side Request Forgery)** — attacker induces the client (or an authorization server fetching a Client ID Metadata Document URL) to make HTTP requests to unintended destinations: internal networks, cloud metadata endpoints, protected services. Mitigations: block private IP ranges, egress proxies, validate redirect targets.
- **State Handle Hijacking** — MCP is stateless; servers needing multi-request state mint explicit handles (shopping-cart ID, workflow ID) passed as ordinary tool arguments. An attacker who obtains/guesses a handle can access another user's state. Mitigation: possession of a state handle is NOT authentication; servers MUST verify all inbound requests and bind handles to authenticated identities.
- **OAuth authorization URL validation** — clients MUST NOT open URLs via shell execution (`cmd.exe`, `sh`, PowerShell); use platform-native, non-shell mechanisms; web clients SHOULD set Content-Security-Policy headers to prevent JavaScript execution.
- **Mix-Up attacks** — authorization responses must be bound to the AS the client recorded before redirecting, so the code can't be redeemed at an attacker's token endpoint; PKCE alone doesn't prevent this (the `code_verifier` would be transmitted to the attacker's token endpoint too); resource indicators help.
- **Localhost Redirect URI Impersonation** — a native attacker registers the legitimate client's metadata URL as `client_id`, binds any localhost port as `redirect_uri`, and catches the authorization code when the user approves. The server sees the legitimate client's metadata and the user sees the legitimate client's name, making detection hard. Mitigations: ASs display extra warnings for localhost-only redirect URIs and clearly show the redirect URI hostname during consent.
- **CIMD Trust Policies** — authorization servers accepting Client ID Metadata Documents apply domain-based trust policies for URL-based client IDs; servers keep full control over access policies.
- **Scope minimization** — when the initial challenge carries no `scope`, clients fall back to requesting all scopes in `scopes_supported` (general-purpose clients lack domain knowledge to pick individually); the AS and end-user narrow during consent, following least privilege. Scope accumulation is client-side: compute the union of previously granted + newly challenged scopes on re-authorization (step-up flow). Common mistakes: wildcard/omnibus scopes (`*`, `all`, `full-access`), bundling unrelated privileges, returning the entire scope catalog in every challenge, silent scope-semantic changes without versioning, treating claimed scopes in a token as sufficient without server-side authorization logic.

General MCP security maxims worth engraving: **treat all server output as untrusted input** (potential prompt injection); user consent before every state-changing or sensitive operation; least-privilege scopes; explicit authentication independent of state handles; sanitize logs.


---

## 15. The Wider Ecosystem: SDKs, Registry, Extensions, SEPs, Governance

### SDKs

Official SDKs implement the protocol per language so you never hand-roll JSON-RPC. The docs list them on the SDKs page; each SDK has its own documentation site (e.g. Python SDK docs at `py.sdk.modelcontextprotocol.io`). SDKs are tiered by feature completeness, protocol support, and maintenance commitment (SEP-1730 SDK tiering system). Languages with quickstarts in the docs: **Python, TypeScript, Java/Kotlin, C#** — plus community SDKs in Go, Rust, Swift, Ruby, PHP and more.

### Example / reference servers

Official reference servers (demonstrate core MCP features and SDK usage):

| Server | What it does |
|---|---|
| **Everything** | Reference/test server with prompts, resources, and tools |
| **Fetch** | Web content fetching and conversion for efficient LLM usage |
| **Filesystem** | Secure file operations with configurable access controls |
| **Git** | Read, search, and manipulate Git repositories |
| **Memory** | Knowledge-graph-based persistent memory |
| **Sequential Thinking** | Dynamic and reflective problem-solving through thought sequences |
| **Time** | Time and timezone conversion |

Run TypeScript servers directly: `npx -y @modelcontextprotocol/server-memory`. Python servers: `uvx mcp-server-git`. Beyond these: **official integrations** (companies maintain servers for their platforms — GitHub, Brave, etc.) and a large **community servers** section, all in the `modelcontextprotocol/servers` repo. The DeepLearning.AI course uses **filesystem** and **fetch** as its "reference servers built by Anthropic's MCP team."

### The MCP Registry

The official **centralized metadata repository for publicly accessible MCP servers** (currently in preview), backed by Anthropic, GitHub, PulseMCP, Microsoft, and other trusted ecosystem contributors. It provides:

- A single place for server creators to publish **metadata** about their servers (a standardized `server.json` format: unique name like `io.github.user/server-name`, where to locate the server — npm package, remote URL — execution instructions, description, capabilities).
- **Namespace management through DNS verification** — reverse-DNS names tied to verified GitHub accounts or domains, so only the legitimate owner can publish under a namespace.
- A **REST API** for MCP clients and aggregators to discover servers, plus an **OpenAPI spec** other registries can implement.

Ecosystem relationships: package registries (npm, PyPI, Docker Hub) host the *code*; the MCP Registry hosts *metadata pointing to* that code. It's consumed primarily by **downstream aggregators** (marketplaces), not directly by host applications; it does **not** support private servers (host your own private registry for those). Spam prevention: namespace authentication, character limits and regex validation, manual takedown (moderation policy). Security scanning is delegated to the underlying package registries and aggregators. Publishing quickstart, package types (npm, PyPI, Docker, remote servers), versioning, GitHub Actions automation, and an authentication guide cover the publish workflow.

### Extensions (optional protocol add-ons)

Extensions define capabilities beyond the core protocol. Identified by `{vendor-prefix}/{extension-name}` — official ones use `io.modelcontextprotocol/…`; third parties use a reversed domain they own (e.g. `com.example/…`) to avoid collisions. Lifecycle is SEP-based (SEP-2133): propose → implement (at least one reference implementation in an official SDK) → review → publish → adopt.

Current official extensions:

- **MCP Apps** (`ext-apps`) — interactive UI elements (charts, forms, video players, dashboards) rendered inline within conversations, with graceful degradation to text content for clients that don't support it.
- **Tasks** (`ext-tasks`) — asynchronous task execution for long-running operations: polling, mid-flight input, durable handles (SEP-1686/2663).
- **Skills** — discover and read Agent Skills from MCP servers (SEP-2640).
- **Auth extensions** — **OAuth Client Credentials** (machine-to-machine) and **Enterprise-Managed Authorization** (centralized access control via identity providers).

Mechanics: clients advertise extension support in `_meta["io.modelcontextprotocol/clientCapabilities"].extensions` per request; servers advertise in the `server/discover` response's `capabilities.extensions`. Each side must degrade gracefully (fall back to core behavior, or reject if the extension is mandatory). Experimental extensions incubate in Working Groups under `experimental-ext-*` repos.

### SEPs (Specification Enhancement Proposals)

Protocol changes run through the SEP process (like RFCs or PEPs). Notable ones you'll encounter in the docs and roadmap:

- **SEP-1686 / 2663 (Tasks)** — async, long-running operations
- **SEP-1865 (MCP Apps)** — interactive user interfaces
- **SEP-2575 (Stateless MCP)**, **SEP-2567 (sessionless via state handles)** — the statelessness overhaul
- **SEP-2577 (Deprecate Roots, Sampling, Logging)** — the deprecation wave of 2026-07-28
- **SEP-2322 (MRTR)** — Multi Round-Trip Requests
- **SEP-986 (tool name format)**, **SEP-2106/1613 (JSON Schema 2020-12)** — schema standards
- **SEP-985/9728 (PRM alignment)**, **SEP-991 (URL-based client registration)**, **SEP-1046 (client credentials)** — auth evolution
- **SEP-2243 (HTTP header standardization)**, **SEP-1699 (SSE polling)** — transport work
- **SEP-2640 (Skills)** — skills over MCP

### Governance and community

MCP is openly governed: contributing guide, contributor communication channels, contributor ladder (first contribution → Core Maintainer), design principles, security policy, antitrust policy, and a feature-lifecycle/deprecation policy (Active → Deprecated → Removed, with documented migration paths and ≥12-month notice under the standard policy). Active **Working Groups** drive workstreams — Agents, File Uploads, Filesystems, Infrastructure, Inspector V2, Interceptors, Registry, SDK, Server Card, Skills Over MCP, Transports, Triggers and Events — and **Interest Groups** include Authorization, Enterprise, Enterprise-Managed Authorization, Financial Services, Primitive Grouping, Scientific Computing, Security, and Tool Annotations. The roadmap page covers priority areas for the next specification release.

---

## 16. DeepLearning.AI Course — Full Walkthrough

**Course:** *MCP: Build Rich-Context AI Apps with Anthropic* (May 2025, built in partnership with Anthropic; instructor **Elie Schoppik**, Head of Technical Education at Anthropic; contribution from Hawraa Salami at DeepLearning.AI; presented with Andrew Ng). ~1h 38m, 11 video lessons + appendix + quiz. Free to audit.

### Lesson-by-lesson summary

**1. Introduction (3 min).** MCP is an open protocol that standardizes how LLM applications access context — tools, data resources, prompt templates — based on client-server architecture. Schoppik recounts the origin: MCP came out of an internal Anthropic project extending Claude Desktop to interact with local file systems and other external systems; they found the protocol useful for many applications, so they published the spec and opened development to the community. The ecosystem includes servers developed by the community and by Anthropic's MCP team; MCP is model-agnostic. Course arc: architecture → chatbot → server → client → reference servers → prompts/resources → Claude Desktop config → remote deployment.

**2. Why MCP (7 min).** The problem: without MCP you reinvent the wheel for every integration — where do tools live, where do custom prompts live, where is the data-access layer and auth logic? Repeated across teams, many AI applications talking to similar data sources but written differently. **Live demo:** Claude Desktop connected to a GitHub MCP server and an Asana MCP server. Asking in natural language about issues in a repo → reading from GitHub → triaging issues → creating and assigning Asana tickets — *reading from one data source and writing to another*, with a **human-in-the-loop** verifying each action. Claude 3.5 Sonnet + a few MCP tools + an environment to run over and over = "a very lightweight agent." With MCP the burden of responsibility shifts and concerns separate cleanly; model-agnostic and open source.

**3. MCP Architecture (14 min).** Client-server architecture; clients maintain **1-to-1 connections** with servers; communication via messages MCP itself defines; clients live inside a **host** (Claude Desktop, Claude AI, or your own app); the host stores and maintains all clients and connections; servers are lightweight programs exposing capabilities. Primitives: tools (model-controlled), resources (app-controlled), prompts (user-controlled). The client's job is to find tools/resources; the server's job is to expose them. Demo: Claude Desktop connected to an **SQLite MCP server** exposing tools, resources, and prompts (configuration details come later). Later lessons build a client and a host containing multiple clients in lower-level Python to show the architecture has no magic.

**4. Chatbot Example (7 min, code).** Starting point: a chatbot with two hand-rolled tools for searching academic papers — `search_papers` (query arXiv) and `extract_info` — defined with explicit JSON schemas and passed to the LLM. This is "tool use as usual" — the pre-MCP baseline every developer writes by hand. The lesson then transforms this chatbot into an MCP-compatible application.

**5. Creating an MCP Server (8 min, code).** Take the same two functions and wrap them in an MCP server over **stdio** transport using **FastMCP** (`from mcp.server.fastmcp import FastMCP`) — a high-level interface where decorating a function with `@mcp.tool()` turns it into an MCP tool, with the docstring becoming the description and type hints becoming the schema. Server named "research." Run locally with **uv** (faster than pip, manages dependencies: `uv init`, `uv add mcp arxiv`), then test in the **MCP Inspector** in the browser: connect, see the two tools, inspect their schemas, invoke them with arguments, watch JSON-RPC traffic and logs live.

**6. Creating an MCP Client (9 min, code).** Build the client side inside the chatbot: spawn the server as a subprocess over stdio, run the MCP session, list tools dynamically at connect time (instead of hardcoding schemas), and loop: user query → LLM sees tool definitions → LLM emits tool call → client routes it through the MCP session → result goes back to the LLM → answer. This replaces the hand-written tool plumbing from Lesson 4 with protocol-driven discovery — the same chatbot now works with *any* MCP server.

**7. Connecting the MCP Chatbot to Reference Servers (12 min, code).** Point the chatbot at trusted third-party servers built by Anthropic's MCP team: **filesystem** (filesystem operations) and **fetch** (extracts web contents as markdown). The client talks to each server through its own 1-to-1 connection; the LLM now has tools across *multiple servers simultaneously* — search papers, read local files, fetch web pages — all discovered dynamically. This demonstrates the "build once, integrate everywhere" payoff from the host side.

**8. Adding Prompt and Resource Features (11 min, code).** Extend the research server beyond tools: add **prompts** (e.g., a prompt template that composes multi-step workflows like "search, then summarize the results" so users don't have to write the perfect multi-step prompt) and **resources** (e.g., exposing the latest preprint as a resource the client can attach for context). Test prompts and resources in the Inspector, then consume them from the chatbot client — completing all three primitives end-to-end.

**9. Configuring Servers for Claude Desktop (6 min).** Reuse is the theme: the *same* research server, no code changes, wired into Claude Desktop via `claude_desktop_config.json` (the `mcpServers` key with command/args/env). Claude Desktop acts as the host and abstracts away all the low-level client logic you wrote by hand in Lesson 6 — proof that a server built once serves every client. Restart Claude Desktop and the tools/prompts appear; tools require approval (human-in-the-loop).

**10. Creating and Deploying Remote Servers (7 min, code).** Move from stdio to a remote deployment so the server runs as an independent process accessible over the network. Deploy the server (e.g., containerized/hosted) and test it with the Inspector or other MCP-compatible applications pointing at its URL. This is the path to production: one deployment serving all users, with OAuth-based auth in front (the course predates the 2026-07-28 stateless/streamable-HTTP refinements, but the concepts map directly).

**11. Conclusion / Roadmap (9 min).** Recap, plus what was on the MCP roadmap at recording time: **multi-agent architecture** (agents talking to agents over MCP), the **MCP Registry API**, **server discovery**, and richer **authorization/authentication** — all of which have since shipped or are shipping (registry is in preview; extensions handle auth; agents working group active). The quiz closes the course.

**Appendix** — notebook mechanics (File → Open for helper utils; download `.ipynb`; upload files; restore original version to reset the workspace), platform features (video speed, captions, quality, PiP), and study tips (dedicated study space, consistent schedule, Pomodoro breaks, community forums, active learning).

### What the course teaches that the docs don't emphasize

- The **before/after** contrast: hand-rolled tool schemas (Lesson 4) vs protocol-driven discovery (Lesson 6) — the fastest way to *feel* what MCP standardizes.
- **FastMCP's decorator ergonomics**: docstring + type hints = tool definition. (In the current SDK era this class is presented as `MCPServer`; same interface.)
- Building a **host with multiple clients** yourself, so the "host vs client" distinction is concrete.
- The human-in-the-loop demo of reading from one system and writing to another (GitHub → Asana) as "a very lightweight agent."

---

## 17. Codebasics YouTube Tutorial — Full Walkthrough

**Video:** *"MCP Tutorial: Build Your First MCP Server"* (codebasics). A beginner-friendly, end-to-end build of a real-life-style MCP server: an **HR leave-management server** for a fictional company "ATL Technologies," used by an HR manager to manage employee leaves.

### The scenario and architecture

- **ATL employees database** holds leave information and leave history (for the tutorial: a mock dict with two employees — E001 with 18 leave days and two leaves already taken; E002 with 20 days and none taken).
- An **MCP server** is built to access this database.
- Ideally the **MCP client** would be an ATL internal chatbot (an "ATL ChatGPT" HR assistant), but to keep the tutorial short, the presenter uses a **generic MCP client: Claude Desktop**, which calls the leave-management MCP server.

### Setup steps shown

1. Install **Claude Desktop** for your OS.
2. Install the MCP Python SDK: `pip install mcp`.
3. Install **uv** (Python package manager): `pip install uv` (or the PowerShell install command on Windows).
4. `uv init my-first-mcp-server` → creates the project skeleton (`main.py`, `pyproject.toml`, `README`).
5. Reference the basic server example on **pypi.org/project/mcp** (a simple two-number addition server), then — the modern workflow — hand that code to **ChatGPT and ask it to generate the leave-management server**, then iterate with follow-up prompts.

### The server code (structure)

```python
from mcp.server.fastmcp import FastMCP

# Mock "database": E001 → 18 days, 2 taken; E002 → 20 days, 0 taken
EMPLOYEES = {...}

mcp = FastMCP("leave-manager")

@mcp.tool()
def get_leave_balance(employee_id: str) -> str:
    """Get the remaining leave balance for an employee."""
    ...

@mcp.tool()
def apply_leave(employee_id: str, leave_dates: list[str]) -> str:
    """Apply for leave. Dates must be in YYYY-MM-DD format."""
    ...

@mcp.tool()
def get_leave_history(employee_id: str) -> str:
    """Get the leave history for an employee."""
    ...

# plus a simple greeting resource

if __name__ == "__main__":
    mcp.run()   # stdio transport by default
```

**The docstring point the presenter hammers on:** the docstring is *very* important — it guides the LLM/MCP client to make the right function call with the right arguments. In the demo, when the user types "4th July," Claude converts it to `2025-07-04` *because the docstring specified the date format*. (This is exactly the official docs' point that descriptions drive tool selection.)

### Installing into Claude Desktop

```bash
uv mcp install main.py
```

This command **registers the server in Claude Desktop's config automatically** ("Successfully installed leave-manager in claude config"). Alternative: manually edit `claude_desktop_config.json` and add the entry (command = `uv`, args pointing at `main.py`), same structure as the official docs. When Claude Desktop starts, it notices the config entry and **internally launches the MCP server as a subprocess**.

Gotchas covered: enable **Developer Mode** in Claude Desktop to see the tools; if you hit a `typer` library error (`str` type errors during install), `pip install --upgrade typer` fixes it.

### The live demo (the payoff)

With three tools exposed (`get_leave_balance`, `apply_leave`, `get_leave_history` — visible under the tools icon):

1. *"How many leaves are available for employee E001?"* → Claude recognizes this isn't a generic internet question, picks `get_leave_balance`, supplies `E001` as the argument → "E001 has 18 days."
2. *"For the same person, tell me the exact dates they took leave"* → context carried across turns; Claude reuses E001 → the two dates (Christmas Day and January 1st).
3. *"Employee E002 would like to apply for a leave on 4th July"* → Claude picks `apply_leave`, converts "4th July" to `2025-07-04` via the docstring format → leave applied.
4. *"How many leave days remaining for E002?"* → 19 days — the balance dropped, proving the write worked.
5. *"Tell me the days of leave for the same person"* → July 4th, with context understood.

The language model (Claude 3.7 in the video) maps natural-language questions to the right function *and* the right arguments — including typos and fuzzy dates — purely from tool names + descriptions + schemas.

### What this tutorial adds on top of the docs/course

- The **fastest zero-to-demo path**: one `uv mcp install` command wires your server into a real host without touching JSON.
- **AI-assisted development** as the expected workflow (generate the server with ChatGPT, iterate).
- A **CRUD-flavored business example** (read balance, write a leave application, read history) closer to what you'd build at work than a weather API.
- Practical Windows pitfalls (typer upgrade, developer mode, config location).

---

## 18. MCP at the Intersection of Agentic AI and AI Engineering

Since you're learning both tracks simultaneously, here is how MCP maps onto each:

### Agentic AI Engineering view

- **Tools are the agent's action space.** An agent is a loop (perceive → reason → act → observe); MCP tools are the "act" step, standardized. The DeepLearning.AI demo calls the GitHub→Asana flow "a very lightweight agent": model + tools + an environment to run over and over.
- **Discovery = dynamic action spaces.** Because tools are listed at runtime (`tools/list`) and can change (`toolsListChanged` notifications), an agent's capabilities can grow or shrink without redeploying the agent.
- **Elicitation = human-in-the-loop agencies.** When the agent needs information only the human has, the server asks; the MRTR retry loop is a structured way to pause the agent on a human.
- **Tasks extension = long-running autonomy.** Async task execution with polling and mid-flight input covers jobs that outlive a single request — the agent delegates and checks back.
- **Multi-agent architecture.** MCP servers can themselves be agents, and agents can expose themselves as servers (the Agents working group's domain; the course's roadmap mentioned multi-agent over MCP).
- **Client best practices are agent-engineering.** Progressive discovery, `search_tools` meta-tools, and programmatic tool calling are exactly how you scale agents from 5 tools to 5000 without melting the context window.
- **The agent reliability lesson:** a failing tool returns `isError: true` and the model reads the message and retries — error *messages* are prompts. Design them for the model.

### AI Engineering view

- **The N×M integration problem → N+M.** This is the core architectural win: write each integration once, against a protocol.
- **Context engineering.** Resources attach data to context; tool descriptions consume context; progressive discovery manages context. MCP makes context costs *measurable and controllable* (tokens per tool definition, ttlMs caching).
- **Evals and observability.** The Inspector is your manual eval harness; OpenTelemetry support covers tracing; logging per request (`_meta` log level) covers inspection.
- **Security engineering.** OAuth 2.1 flows, PRM documents, audience validation, SSRF/prompt-injection hardening — an AI engineer ships these, not just the happy path.
- **Versioning discipline.** Client/server negotiate versions per request; deprecations come with 12-month runways. Building on a protocol with published lifecycle beats building on private interfaces.
- **Statelessness as a scaling decision.** The 2026-07-28 redesign (stateless, per-request `_meta`, explicit state handles) is a masterclass in protocol evolution driven by production scaling needs — read the SEPs (2575, 2567) as design case studies.

### Where the two tracks meet in MCP

The **host** is where agentic loop engineering meets AI engineering: the host owns the LLM calls (AI engineering: prompts, context, tokens, evals) *and* the agent loop (agentic: tool routing, retries, human approval, multi-step planning). The **server** is classic integration/backend engineering with an LLM-facing API design layer (descriptions, schemas, error messages written for a model, not a human).

---

## 19. Suggested Learning Path and Projects

**Phase 1 — Concepts (days 1–2).** Read §1–§8 of this guide. Then skim the official *Architecture overview* and *Server/Client concepts* pages. Mental checkpoint: explain tools vs resources vs prompts, and host vs client vs server, from memory.

**Phase 2 — First server, the codebasics way (days 3–4).** Build the leave-management server from §17: `uv init`, FastMCP decorators, mock data, `uv mcp install main.py`, demo in Claude Desktop. Add one tool of your own invention.

**Phase 3 — The official quickstart (days 5–6).** Do the docs' weather server (§9) in Python or TypeScript. Test it in the **MCP Inspector** before ever touching a host. Then wire it into Claude Desktop with a hand-written config entry.

**Phase 4 — Client + host, the course way (week 2).** Follow the DeepLearning.AI lessons 4–7 (§16): write a chatbot with hand-rolled tools, then make it MCP-compatible; build the client loop (`async with Client(...)`, `list_tools`, `call_tool`, tool-result plumbing); connect to filesystem + fetch reference servers; add a prompt and a resource to your server.

**Phase 5 — Remote and secure (week 3).** Deploy the server remotely (Streamable HTTP); stand up OAuth (Keycloak or any AS) using the authorization tutorial; understand the attack catalog in §14.

**Phase 6 — Scale patterns (week 4).** Implement progressive discovery (`search_tools` + threshold) in your host once you have >1% of context in tool definitions. Try programmatic tool calling for a multi-tool workflow.

**Project ideas that exercise everything:**

1. **Personal research assistant** (course-flavored): arXiv search + fetch + filesystem servers + a "literature review" prompt; your chatbot as host.
2. **Company HR/IT helpdesk** (codebasics-flavored): leave management + knowledge-base resources + elicitation for missing info.
3. **Data analyst agent**: SQLite server + charting via MCP Apps widgets; long queries via the Tasks extension.
4. **Ops agent**: GitHub server (read issues) + Asana/Linear server (write tickets) — the exact Why-MCP demo, rebuilt by you.

---

## 20. Quick Reference

**Core protocol methods (2026-07-28 era):**

| Method | Direction | Purpose |
|---|---|---|
| `server/discover` | Client → Server | Versions, capabilities, identity (optional but recommended first call) |
| `tools/list` | Client → Server | List tool descriptors (+ cursor pagination) |
| `tools/call` | Client → Server | Execute a tool → `content[]` (+ `isError`) |
| `resources/list`, `resources/read` | Client → Server | List/read read-only context |
| `prompts/list`, `prompts/get` | Client → Server | List/get prompt templates |
| `elicitation/create` | Server → Client (inside `InputRequiredResult`) | Ask the user for structured input |
| `subscriptions/listen` | Client → Server | Open notification stream (e.g. `toolsListChanged`) |
| `notifications/tools/list_changed` | Server → Client | Tool set changed |

**Every request carries** (`_meta`): `io.modelcontextprotocol/protocolVersion` (required), `io.modelcontextprotocol/clientCapabilities` (required), `io.modelcontextprotocol/clientInfo` (should).

**JSON-RPC error codes worth knowing:** `-32602` invalid params (missing `_meta`), `-32021` missing required client capability, `-32022` unsupported protocol version (data lists supported versions).

**Config file (Claude Desktop):** Windows `%APPDATA%\Claude\claude_desktop_config.json`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`. Shape: `{ "mcpServers": { "<name>": { "command": "...", "args": [...], "env": {...} } } }`.

**Cheats:**

- stdio server → never print to stdout.
- Tool errors → `isError: true` results, write helpful messages for the model.
- Tool descriptions/docstrings are prompts — invest in them.
- Absolute paths in configs; env vars don't inherit.
- Inspector first, host second.
- First tool call may take 30s; that's normal.
- Deprecated (2026-07-28): roots, sampling, protocol logging. Migrate per §7.

---

## 21. Glossary

- **Host** — the LLM application users interact with (Claude Desktop, VS Code, your chatbot); owns one or more clients.
- **Client** — protocol component inside a host; one client ↔ exactly one server connection.
- **Server** — lightweight program exposing tools/resources/prompts over MCP; subprocess (stdio) or remote (Streamable HTTP).
- **Tool** — model-controlled executable function with a JSON Schema.
- **Resource** — application-controlled read-only data (URI-addressable).
- **Prompt** — user-controlled reusable template with arguments.
- **Elicitation** — server asking the user for structured input mid-request (MRTR pattern).
- **Roots** *(deprecated)* — client-declared filesystem scope.
- **Sampling** *(deprecated)* — server requesting LLM completions from the client.
- **Discovery** — `server/discover` + `*/list` methods; runtime capability detection.
- **Transport** — stdio (subprocess stdio/stdout) or Streamable HTTP (POST + SSE).
- **`_meta`** — per-request metadata block (protocol version, client info, capabilities).
- **PRM** — Protected Resource Metadata (RFC 9728) document pointing to authorization servers.
- **MRTR** — Multi Round-Trip Requests: server pauses a request to gather input, client retries with responses.
- **State handle** — explicit identifier (cart ID, workflow ID) carrying state across stateless requests.
- **Inspector** — official interactive testing/debugging tool (web/CLI/TUI).
- **Registry** — official metadata repository for public MCP servers (reverse-DNS namespaces).
- **Extension** — optional spec add-on (`io.modelcontextprotocol/...`).
- **SEP** — Specification Enhancement Proposal (the protocol's RFC process).
- **MCPB** — MCP Bundle: packaged local server + runtime (`.mcpb`).
- **MCP Apps** — extension for inline interactive UI in chat.
- **FastMCP / MCPServer** — high-level Python server class (decorator-based); FastMCP was the 2025-era name, MCPServer the current one.
- **uv / uvx** — fast Python package/dependency manager; `uvx` runs published servers on demand.
- **Confused deputy** — attack where a privileged client is tricked into misusing authority.
- **Progressive discovery** — pattern: defer tool definitions, expose `search_tools`, load on demand.
- **Programmatic tool calling** — model writes a script against typed tool wrappers; host brokers execution in a sandbox.

---

## 22. Sources Read for This Guide

**Official documentation — modelcontextprotocol.io (spec version 2026-07-28):**

- Documentation index (`llms.txt`) — complete site map: docs, all spec versions (2024-11-05 → 2026-07-28 → draft), extensions, registry, SEPs, community
- What is the Model Context Protocol (MCP)? (getting-started/intro)
- Architecture overview (learn/architecture)
- Understanding MCP servers (learn/server-concepts)
- Understanding MCP clients (learn/client-concepts)
- Versioning (learn/versioning)
- Connect to local MCP servers (develop/connect-local-servers)
- Connect to remote MCP Servers (develop/connect-remote-servers)
- Build with Agent Skills (develop/build-with-agent-skills)
- Build an MCP server (develop/build-server)
- Build an MCP client (develop/build-client)
- Client Best Practices (develop/clients/client-best-practices)
- SDKs (sdk)
- Understanding Authorization in MCP (tutorials/security/authorization)
- Security Best Practices (tutorials/security/security_best_practices)
- MCP Inspector (tools/inspector) + Debugging (tools/debugging)
- Example Servers (examples)
- The MCP Registry (registry/about)
- Extensions Overview (extensions/overview)

**DeepLearning.AI:** *MCP: Build Rich-Context AI Apps with Anthropic* — course overview page, syllabus (11 lessons + appendix + quiz), and lesson content/transcripts for Introduction, Why MCP, MCP Architecture, Creating an MCP Server, plus the full lesson listing (Chatbot Example, Creating an MCP Client, Connecting the MCP Chatbot to Reference Servers, Adding Prompt and Resource Features, Configuring Servers for Claude Desktop, Creating and Deploying Remote Servers, Conclusion/Roadmap, Appendix). Course landing page (deeplearning.ai/courses/mcp-build-rich-context-ai-apps-with-anthropic).

**YouTube (codebasics):** *"MCP Tutorial: Build Your First MCP Server"* — full transcript read (HR leave-management server for "ATL Technologies": mock DB, FastMCP tools with docstring-driven schemas, `uv mcp install`, Claude Desktop config, live natural-language demo with argument mapping and date coercion, typer/dev-mode gotchas).

---

*Guide compiled October 2026. The protocol evolves quickly — for anything security- or spec-critical, verify against `modelcontextprotocol.io/specification/2026-07-28` and the deprecation page, and watch the draft spec for what's next.*
