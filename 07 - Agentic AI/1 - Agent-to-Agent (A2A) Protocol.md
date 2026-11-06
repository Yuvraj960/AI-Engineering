# Agent-to-Agent (A2A) Protocol — Complete Technical Guide

## 1. Executive Summary

### 1.1 What is A2A?

> **Agent2Agent (A2A) Protocol** is an **open standard protocol** for seamless communication and collaboration between AI agents, regardless of their underlying framework, vendor, or hosting platform.

> **Released:** April 2025 | **License:** Apache-2.0 | **Governance:** Linux Foundation (Technical Steering Committee: AWS, Cisco, Google, IBM Research, Microsoft, Salesforce, SAP, ServiceNow)

### 1.2 The Problem A2A Solves

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        THE FRAGMENTATION PROBLEM                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Customer Service Agent  ──► Inventory Agent ──► Supplier Agents            │
│        │                        │                    │                      │
│        │   Custom Integration   │ Custom Integration │  Custom Integration  │
│        │   (Weeks of work)      │   (Weeks of work)  │   (Weeks of work)    │
│        ▼                        ▼                    ▼                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  RESULT: Vendor lock-in, 20-40% engineering time on integrations,   │   │
│   │  governance gaps, inability to swap agents                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  A2A SOLUTION: Standardized protocol → Agents communicate as peers          │
│  without custom code, preserving IP, enabling vendor diversity              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Key Statistics (from Galileo AI Blog)

| Metric | Value | Source |
|--------|-------|--------|
| **Integration complexity** | 20-40% of engineering resources | Industry analysts |
| **Vendor lock-in risk** | Critical GenAI blind spot | Industry analysts |
| **Early adoption** | Microsoft integrated; limited production deployments (early 2025) | Galileo AI |
| **Protocol governance** | Linux Foundation (vendor-neutral) | A2A Protocol |

---

## 2. A2A vs MCP: Complementary, Not Competing

### 2.1 The Critical Distinction

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MCP vs A2A: COMPLEMENTARY                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────┐         ┌─────────────────────────┐           │
│   │      MCP (Model         │         │      A2A (Agent-to-     │           │
│   │   Context Protocol)     │         │       Agent Protocol)   │           │
│   ├─────────────────────────┤         ├─────────────────────────┤           │
│   │ Agent ←→ Tool/Resource  │         │     Agent ←→ Agent      │           │
│   │                         │         │                         │           │
│   │ • Standardizes how an   │         │ • Standardizes how      │           │
│   │   agent connects to     │         │   agents discover,      │           │
│   │   tools, APIs, data     │         │   delegate, and share   │           │
│   │   (databases, files,    │         │   results securely      │           │
│   │   APIs, services)       │         │   across frameworks     │           │
│   │                         │         │                         │           │
│   │ "Agent-to-Tool"         │         │ "Agent-to-Agent"        │           │
│   └─────────────────────────┘         └─────────────────────────┘           │
│                                                                             │
│   THEY WORK TOGETHER:                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Client Agent ──A2A──► Remote Agent ──MCP──► Remote Tool            │   │
│   │       │                    │                    │                   │   │
│   │   Uses A2A              Uses A2A            Uses MCP                │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Comparison Matrix

| Aspect | MCP (Model Context Protocol) | A2A (Agent-to-Agent) |
|--------|------------------------------|----------------------|
| **Purpose** | Agent ↔ Tool/Resource | Agent ↔ Agent |
| **Scope** | Single agent's toolbox | Multi-agent collaboration |
| **Standardizes** | Tool definitions, resource access | Agent discovery, task delegation, result sharing |
| **Communication** | Agent → Tool | Agent ↔ Agent (peer-to-peer) |
| **Use Case** | Equip agent with tools | Enable multi-agent workflows |
| **Analogy** | USB for AI tools | HTTP for AI agents |
| **Governance** | Anthropic-led | Linux Foundation (multi-vendor) |

---

## 3. Core A2A Concepts

### 3.1 The A2A Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         A2A SYSTEM ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐         A2A Protocol          ┌──────────────┐            │
│  │   Client     │ ◄──────────────────────────►  │   Remote     │            │
│  │   Agent      │   (HTTP/HTTPS, JSON-RPC 2.0,  │   Agent      │            │
│  │              │   SSE, OAuth 2.0, mTLS)       │              │            │
│  └──────┬───────┘                               └──────┬───────┘            │
│         │                                              │                    │
│         ▼                                              ▼                    │
│  ┌──────────────────┐                         ┌──────────────────┐          │
│  │  Agent Card      │                         │  Agent Card      │          │
│  │  (Discovery)     │                         │  (Discovery)     │          │
│  │  • Name          │                         │  • Name          │          │
│  │  • Description   │                         │  • Description   │          │
│  │  • Capabilities  │                         │  • Capabilities  │          │
│  │  • Skills        │                         │  • Skills        │          │
│  │  • Auth methods  │                         │  • Auth methods  │          │
│  │  • Endpoints     │                         │  • Endpoints     │          │
│  └──────────────────┘                         └──────────────────┘          │
│                                                                             │
│  Task Lifecycle:                                                            │
│  SUBMITTED → WORKING → INPUT_REQUIRED → WORKING → COMPLETED/FAILED/CANCELLED│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Core Components

| Component | Purpose | Key Fields |
|-----------|---------|------------|
| **Agent Card** | Discovery & capability advertisement | name, description, version, capabilities, skills, auth, endpoints |
| **Task** | Unit of work with lifecycle | id, contextId, status, artifacts, history |
| **Message** | Communication unit | role (user/agent), parts (text, file, data) |
| **Artifact** | Output/result of task | name, mimeType, parts |
| **Push Notification** | Async updates | webhook URL, authentication |

---

## 4. A2A Protocol Specification Deep Dive

### 4.1 Transport & Encoding

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Transport** | HTTP/HTTPS | Standard web transport |
| **Encoding** | JSON-RPC 2.0 | Structured RPC calls |
| **Streaming** | Server-Sent Events (SSE) | Real-time updates |
| **Auth** | OAuth 2.0, API Keys, mTLS | Enterprise-grade security |
| **Serialization** | JSON | Universal compatibility |

### 4.2 Agent Card (Discovery)

```json
{
  "name": "Weather Agent",
  "description": "Provides current weather and forecasts",
  "version": "1.0.0",
  "url": "https://weather-agent.example.com",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["Bearer", "APIKey"],
    "credentials": "https://auth.example.com/.well-known/jwks.json"
  },
  "skills": [
    {
      "id": "current_weather",
      "name": "Current Weather",
      "description": "Get current weather for a city",
      "tags": ["weather", "current"],
      "inputModes": ["text"],
      "outputModes": ["text", "json"],
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "City name"}
        },
        "required": ["city"]
      }
    }
  ],
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text", "json"]
}
```

### 3.3 Task Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TASK LIFECYCLE STATES                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SUBMITTED ──► WORKING ──► INPUT_REQUIRED ──► WORKING ──► COMPLETED        │
│      │           │              │               │           │               │
│      │           │              │               │           └──► FAILED     │
│      │           │              │               │                           │
│      │           │              │               └──────────► CANCELLED      │
│      │           │              │                                           │
│      │           │              └────────────────────────────────────────►  │
│      │           │  (Agent needs more input from user)                      │
│      │           │                                                          │
│      │           └───────────────────────────────────────────────────────►  │
│      │    (Agent processing)                                                │
│      │                                                                      │
│      └──────────────────────────────────────────────────────────────────►   │
│   (Task submitted by client)                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Core RPC Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `message/send` | Client → Agent | Send message, get response |
| `message/stream` | Client → Agent | Streaming response (SSE) |
| `tasks/get` | Client → Agent | Get task status/result |
| `tasks/cancel` | Client → Agent | Cancel running task |
| `tasks/pushNotificationConfig/set` | Client → Agent | Configure webhooks |
| `tasks/resubscribe` | Client → Agent | Reconnect to stream |
| `agent/card/get` | Client → Agent | Get agent card (discovery) |

---

## 5. A2A vs MCP: Detailed Complementary Roles

### 5.1 The Complete Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE A2A + MCP ECOSYSTEM                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐     A2A      ┌─────────────┐     MCP      ┌───────────┐   │
│   │   User      │ ◄──────────► │  Client     │ ◄──────────► │  Tools/   │   │
│   │   (Human)   │              │  Agent      │              │  Resources│   │
│   └─────────────┘              └──────┬──────┘              └───────────┘   │
│                                       │                                     │
│                                       │ A2A                                 │
│                                       ▼                                     │
│                              ┌─────────────────┐                            │
│                              │  Remote Agent   │                            │
│                              │  (Specialist)   │                            │
│                              └────────┬────────┘                            │
│                                       │                                     │
│                                       │ MCP                                 │
│                                       ▼                                     │
│                              ┌─────────────────┐                            │
│                              │  Remote Tools   │                            │
│                              │  (APIs, DBs,    │                            │
│                              │   Files, APIs)  │                            │
│                              └─────────────────┘                            │
│                                                                             │
│  FLOW:                                                                      │
│  1. User sends request to Client Agent                                      │
│  2. Client Agent discovers Remote Agent via A2A (Agent Card)                │
│  3. Client Agent delegates task via A2A (task/send)                         │
│  4. Remote Agent uses MCP to access tools (databases, APIs)                 │
│  5. Remote Agent returns result via A2A                                     │
│  6. Client Agent synthesizes and responds to user                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 When to Use Each

| Scenario | Use MCP | Use A2A |
|----------|---------|---------|
| Agent needs to query database | ✅ | ❌ |
| Agent needs to call external API | ✅ | ❌ |
| Agent needs to read files | ✅ | ❌ |
| Two agents need to collaborate | ❌ | ✅ |
| Agent delegates subtask to specialist | ❌ | ✅ |
| Cross-organization agent communication | ❌ | ✅ |
| Agent discovers capabilities of peer | ❌ | ✅ |

---

## 6. A2A Implementation: Python SDK Quickstart

### 6.1 Installation

```bash
pip install a2a-sdk
# or
pip install a2a-sdk[all]  # with all extras
```

### 6.2 Building an A2A Server (Agent)

```python
# server.py
from a2a.server import A2AServer
from a2a.types import AgentCard, AgentSkill, AgentCapabilities
from a2a.server.task_manager import InMemoryTaskManager

# Define agent capabilities
card = AgentCard(
    name="Weather Agent",
    description="Provides weather information",
    version="1.0.0",
    url="http://localhost:8000",
    capabilities=AgentCapabilities(
        streaming=True,
        push_notifications=True,
        state_transition_history=True
    ),
    skills=[
        AgentSkill(
            id="current_weather",
            name="Current Weather",
            description="Get current weather for a city",
            tags=["weather"],
            input_modes=["text"],
            output_modes=["text", "json"]
        )
    ]
)

# Implement task manager
class WeatherTaskManager(InMemoryTaskManager):
    async def execute_task(self, task_id: str, message: Message) -> AsyncIterable[TaskStatus]:
        # Extract city from message
        city = message.parts[0].text
        
        # Call weather API
        weather = await self.get_weather(city)
        
        # Yield progress
        yield TaskStatus(state=TaskState.WORKING, message="Fetching weather...")
        
        # Return result
        yield TaskStatus(
            state=TaskState.COMPLETED,
            artifacts=[Artifact(
                name="weather_report",
                parts=[TextPart(text=f"Weather in {city}: {weather}")]
            )]
        )

# Create and run server
server = A2AServer(
    agent_card=card,
    task_manager=WeatherTaskManager()
)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(server.app, host="0.0.0.0", port=8000)
```

### 6.3 Building an A2A Client

```python
# client.py
from a2a.client import A2AClient
from a2a.types import Message, TextPart

async def main():
    # Discover agent
    client = A2AClient("http://localhost:8000")
    card = await client.get_agent_card()
    print(f"Connected to: {card.name}")
    print(f"Skills: {[s.name for s in card.skills]}")
    
    # Send message
    message = Message(
        role="user",
        parts=[TextPart(text="What's the weather in Tokyo?")]
    )
    
    # Stream response
    async for event in client.send_message_streaming(message):
        if event.type == "status":
            print(f"Status: {event.message}")
        elif event.type == "artifact":
            print(f"Result: {event.artifact}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## 7. Security & Enterprise Features

### 7.1 Authentication Methods

| Method | Use Case | Implementation |
|--------|----------|----------------|
| **OAuth 2.0** | Enterprise SSO | Bearer tokens, JWKS validation |
| **API Keys** | Simple service-to-service | Header-based auth |
| **mTLS** | Zero-trust networks | Mutual TLS certificates |
| **Custom** | Proprietary systems | Extensible auth framework |

### 7.2 Security Model

| Principle | Implementation |
|-----------|----------------|
| **Opaque Agents** | Internal state, memory, tools never exposed |
| **Least Privilege** | Agents only expose declared skills |
| **Audit Trail** | Full task history with signatures |
| **Encryption** | TLS 1.3 for transport, optional message encryption |
| **Rate Limiting** | Per-agent, per-skill quotas |

---

## 8. Deployment Patterns

### 8.1 Cloud Run (Google Cloud)

```yaml
# cloudrun.yaml
service: weather-agent
image: gcr.io/project/weather-agent:latest
env:
  - name: LANGCHAIN_TRACING_V2
    value: "true"
  - name: LANGCHAIN_PROJECT
    value: "weather-agent-prod"
scaling:
  min_instances: 1
  max_instances: 100
```

### 8.2 Kubernetes Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: weather-agent
  template:
    spec:
      containers:
      - name: agent
        image: weather-agent:latest
        ports:
        - containerPort: 8000
        env:
        - name: A2A_PORT
          value: "8000"
---
apiVersion: v1
kind: Service
metadata:
  name: weather-agent
spec:
  selector:
    app: weather-agent
  ports:
  - port: 80
    targetPort: 8000
```

### 8.3 Cloud Run Managed A2A (Google)

```bash
# Deploy to Cloud Run with A2A support
gcloud run deploy weather-agent \
  --image gcr.io/project/weather-agent \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars=A2A_ENABLED=true
```

---

## 8. A2A vs Other Protocols

### 8.1 Comparison Matrix

| Protocol | Scope | Transport | Auth | Best For |
|----------|-------|-----------|------|----------|
| **A2A** | Agent↔Agent | HTTP/JSON-RPC/SSE | OAuth2/mTLS/APIKey | Multi-agent systems |
| **MCP** | Agent↔Tool | stdio/HTTP | OAuth2 | Tool access |
| **gRPC** | Service↔Service | HTTP/2 + Protobuf | mTLS/OAuth | Microservices |
| **REST** | Service↔Service | HTTP/JSON | Various | General APIs |
| **WebSocket** | Real-time | WS/WSS | Token | Real-time apps |
| **Message Queue** | Async | AMQP/Kafka | SASL | Event-driven |

### 8.2 When NOT to Use A2A

| Scenario | Better Alternative |
|----------|-------------------|
| Single agent, no collaboration | Direct function calls |
| Agent needs tool access | MCP |
| High-frequency, low-latency | gRPC / WebSocket |
| Human chat | Slack/Discord/Teams |
| Simple request-response | REST API |

---

## 9. Ecosystem & Adoption

### 9.1 Framework Support

| Framework | A2A Support | Status |
|-----------|-------------|--------|
| **LangGraph** | ✅ Native | Production |
| **CrewAI** | ✅ Native | Production |
| **Semantic Kernel** | ✅ Native | Beta |
| **AutoGen** | ✅ Native | Beta |
| **ADK (Google)** | ✅ Native | Production |
| **Custom** | ✅ SDK | Any |

### 9.2 SDKs Available

| Language | Package | Install |
|----------|---------|---------|
| **Python** | `a2a-sdk` | `pip install a2a-sdk` |
| **Go** | `a2a-go` | `go get github.com/a2aproject/a2a-go` |
| **JavaScript/TS** | `@a2a-js/sdk` | `npm install @a2a-js/sdk` |
| **Java** | `a2a-java` | Maven/Gradle |
| **.NET** | `A2A` | `dotnet add package A2A` |
| **Rust** | `a2a-rs` | `cargo add a2a-lf` |

### 9.3 Industry Partners (Linux Foundation TSC)

- **Cloud:** AWS, Google Cloud, Microsoft Azure
- **Enterprise:** IBM Research, Salesforce, SAP, ServiceNow
- **Networking:** Cisco
- **Community:** Open source contributors

---

## 10. A2A Request Lifecycle Example

### 10.1 Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    A2A REQUEST LIFECYCLE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. DISCOVERY                                                               │
│  ┌─────────┐     GET /.well-known/agent.json     ┌─────────────┐            │
│  │ Client  │ ──────────────────────────────────► │   Remote    │            │
│  │ Agent   │ ◄────────────────────────────────── │   Agent     │            │
│  └─────────┘         Agent Card (JSON)           └─────────────┘            │
│                                                                             │
│  2. TASK SUBMISSION                                                         │
│  ┌─────────┐     POST /tasks/send (JSON-RPC)    ┌─────────────┐             │
│  │ Client  │ ─────────────────────────────────► │   Remote    │             │
│  │ Agent   │                                    │   Agent     │             │
│  └─────────┘         Task ID + Status           └─────────────┘             │
│                                                                             │
│  3. EXECUTION (Streaming)                                                   │
│  ┌─────────┐     SSE Stream (Server-Sent Events) ┌─────────────┐            │
│  │ Client  │ ◄────────────────────────────────── │   Remote    │            │
│  │ Agent   │  WORKING → INPUT_REQUIRED → WORKING │   Agent     │            │
│  └─────────┘                                     └─────────────┘            │
│                                                                             │
│  4. COMPLETION                                                              │
│  ┌─────────┐     GET /tasks/{id} (JSON-RPC)    ┌─────────────┐              │
│  │ Client  │ ─────────────────────────────────►│   Remote    │              │
│  │ Agent   │ ◄─────────────────────────────────│   Agent     │              │
│  └─────────┘     COMPLETED + Artifacts         └─────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Example JSON-RPC Messages

**Request (Client → Agent):**
```json
{
  "jsonrpc": "2.0",
  "id": "req-123",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{"type": "text", "text": "What's the weather in Tokyo?"}],
      "messageId": "msg-456"
    },
    "configuration": {
      "acceptedOutputModes": ["text", "json"]
    }
  }
}
```

**Response (Agent → Client):**
```json
{
  "jsonrpc": "2.0",
  "id": "req-123",
  "result": {
    "taskId": "task-789",
    "contextId": "ctx-123",
    "status": {
      "state": "COMPLETED",
      "message": "Task completed successfully"
    },
    "artifacts": [
      {
        "name": "weather_report",
        "mimeType": "application/json",
        "parts": [{"type": "text", "text": "{\"city\":\"Tokyo\",\"temp\":22,\"condition\":\"Sunny\"}"}]
      }
    ]
  }
}
```

---

## 11. Testing & Development Tools

### 11.1 A2A CLI

```bash
# Install CLI
pip install a2a-cli

# Discover agent
a2a discover http://localhost:8000

# Send message
a2a send http://localhost:8000 "What's the weather in Tokyo?"

# Stream response
a2a stream http://localhost:8000 "Research solar energy trends"

# Manage tasks
a2a tasks list http://localhost:8000
a2a tasks get http://localhost:8000 task-123
a2a tasks cancel http://localhost:8000 task-123
```

### 11.2 Testing with A2A Mock Server

```python
# test_agent.py
import pytest
from a2a.testing import MockA2AServer
from a2a.client import A2AClient

@pytest.fixture
async def mock_server():
    server = MockA2AServer()
    server.add_skill("weather", lambda city: f"Sunny in {city}")
    await server.start()
    yield server
    await server.stop()

async def test_weather_agent(mock_server):
    client = A2AClient(mock_server.url)
    result = await client.send_message("Weather in Paris")
    assert "Sunny in Paris" in result
```

---

## 12. Governance & Community

### 11.1 Governance Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        A2A GOVERNANCE (Linux Foundation)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Technical Steering Committee (TSC)               │    │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌────────────┐ ┌─────────┐ ┌──────────┐ │    │
│  │  │ AWS  │ │Cisco │ │Google│ │IBM Research│ │Microsoft│ │Salesforce│ │    │
│  │  └──────┘ └──────┘ └──────┘ └────────────┘ └─────────┘ └──────────┘ │    │
│  │  ┌──────┐ ┌──────────┐                                              │    │
│  │  │SAP   │ │ServiceNow│                                              │    │
│  │  └──────┘ └──────────┘                                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  • Specification evolution                                                  │
│  • Extension governance (tiered promotion)                                  │
│  • Release management                                                       │
│  • Conformance testing                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Extension Governance (Tiered Promotion)

| Tier | Status | Requirements |
|------|--------|--------------|
| **Experimental** | Proposed | Community review |
| **Standard** | Accepted | Multiple implementations, test suite |
| **Core** | Mandatory | All compliant agents must support |

---

## 12. Quick Reference Card

### 12.1 Key URLs

| Resource | URL |
|----------|-----|
| **Protocol Spec** | https://a2a-protocol.org/latest/specification/ |
| **Documentation** | https://a2a-protocol.org/ |
| **GitHub** | https://github.com/a2aproject/A2A |
| **Python SDK** | https://github.com/a2aproject/a2a-python |
| **Discord** | https://discord.gg/a2aprotocol |
| **DeepLearning.AI Course** | https://goo.gle/dlai-a2a |

### 12.2 Key Classes (Python SDK)

| Class | Purpose |
|-------|---------|
| `A2AServer` | HTTP server for A2A agent |
| `A2AClient` | Client for calling A2A agents |
| `AgentCard` | Agent metadata/description |
| `AgentSkill` | Individual capability |
| `TaskManager` | Task execution logic |
| `InMemoryTaskManager` | Simple task storage |

### 12.3 Key Types

| Type | Description |
|------|-------------|
| `TaskState` | SUBMITTED, WORKING, INPUT_REQUIRED, COMPLETED, FAILED, CANCELLED |
| `MessageRole` | user, agent |
| `PartType` | text, file, data |
| `AuthenticationScheme` | Bearer, APIKey, MutualTLS |

---

## 13. Decision Flowchart

```
Need agent communication?
│
├─► Agent needs tool access?
│   └─► Use MCP (Model Context Protocol)
│
├─► Multiple agents need to collaborate?
│   ├─► Same organization, same framework?
│   │   └─► Native framework communication (LangGraph, CrewAI)
│   │
│   ├─► Different frameworks/vendors?
│   │   └─► Use A2A Protocol ✓
│   │
│   └─► Cross-organization?
│       └─► Use A2A Protocol ✓
│
├─► Agent needs to discover peer capabilities?
│   └─► Use A2A Agent Cards ✓
│
├─► Need secure, opaque collaboration?
│   └─► Use A2A (no internal exposure) ✓
│
└─► Simple request-response, no autonomy?
    └─► REST/gRPC better suited
```

---

## 14. Summary: A2A in Your Agentic Stack

| Layer | Technology | Role |
|-------|------------|------|
| **Framework** | LangGraph, CrewAI, ADK | Build agents |
| **Tool Access** | MCP | Equip agents with tools |
| **Agent Communication** | **A2A** | **Inter-agent collaboration** |
| **Orchestration** | LangGraph / Custom | Multi-agent workflows |
| **Observability** | LangSmith / Langfuse | Trace, evaluate, monitor |
| **Deployment** | Cloud Run / K8s / Cloud | Scale agents |
| **Security** | OAuth2 / mTLS / API Keys | AuthZ/AuthN |

---

## 15. References & Learning Resources

### Official
- **A2A Protocol Spec:** https://a2a-protocol.org/latest/specification/
- **GitHub:** https://github.com/a2aproject/A2A
- **Python SDK:** https://github.com/a2aproject/a2a-python
- **DeepLearning.AI Course:** https://goo.gle/dlai-a2a

### Articles & Videos
- **Krish Naik (Video):** https://www.youtube.com/watch?v=56BXHCkngss
- **Anthropic Research:** https://www.anthropic.com/research/building-effective-agents
- **Galileo AI Blog:** https://galileo.ai/blog/google-agent2agent-a2a-protocol-guide
- **Google Cloud Run:** https://docs.cloud.google.com/run/docs/ai/a2a-agents
- **Google Developers Blog:** https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/

### Standards
- **JSON-RPC 2.0:** https://www.jsonrpc.org/specification
- **OAuth 2.0:** https://oauth.net/2/
- **Server-Sent Events:** https://html.spec.whatwg.org/multipage/server-sent-events.html

---