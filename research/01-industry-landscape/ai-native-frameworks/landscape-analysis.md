# AI-Native & Agent Development Frameworks — Landscape Analysis

> **Category 3** of the AI-Native Runtime Research Initiative  
> **Last updated:** 2026-11-12  
> **Scope:** LangChain/LangGraph, Google ADK, Microsoft Semantic Kernel, AutoGen/AG2, CrewAI, MCP, A2A

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Framework-by-Framework Deep Dive](#2-framework-by-framework-deep-dive)
   - 2.1 [LangChain / LangGraph](#21-langchain--langgraph)
   - 2.2 [Google Agent Development Kit (ADK)](#22-google-agent-development-kit-adk)
   - 2.3 [Microsoft Semantic Kernel](#23-microsoft-semantic-kernel)
   - 2.4 [AutoGen / AG2](#24-autogen--ag2)
   - 2.5 [CrewAI](#25-crewai)
   - 2.6 [Model Context Protocol (MCP)](#26-model-context-protocol-mcp)
   - 2.7 [Agent-to-Agent Protocol (A2A)](#27-agent-to-agent-protocol-a2a)
3. [Architecture Comparison Diagrams](#3-architecture-comparison-diagrams)
4. [Comparative Evaluation Matrix](#4-comparative-evaluation-matrix)
5. [Cross-Cutting Analysis](#5-cross-cutting-analysis)
6. [Limitations for AI-Native Runtime Research](#6-limitations-for-ai-native-runtime-research)
7. [Key Takeaways & Recommendations](#7-key-takeaways--recommendations)
8. [References](#8-references)

---

## 1. Executive Summary

The AI agent development ecosystem has rapidly expanded from simple prompt-chaining libraries into sophisticated orchestration frameworks. This document analyzes seven major frameworks and protocols that represent the state of the art in building AI-native applications. The core finding is:

> **All current frameworks are "wrappers" around LLM APIs—not runtimes.** They orchestrate calls to external inference services and manage conversational state, but none provides a complete application runtime with native scheduling, hardware awareness, distributed execution, or process isolation.

The frameworks divide into three categories:

| Category | Frameworks | Focus |
|---|---|---|
| **Agent Orchestration** | LangGraph, Google ADK, Semantic Kernel, AutoGen/AG2, CrewAI | Building and executing AI agent workflows |
| **Tool Integration Protocol** | MCP (Model Context Protocol) | Standardizing agent-to-tool/data connections |
| **Agent Communication Protocol** | A2A (Agent-to-Agent) | Standardizing agent-to-agent collaboration |

MCP and A2A are protocols, not frameworks—they define interfaces. The orchestration frameworks increasingly adopt both protocols but remain fundamentally application-layer libraries running atop traditional runtimes (Python, .NET, Node.js).

---

## 2. Framework-by-Framework Deep Dive

### 2.1 LangChain / LangGraph

| Attribute | Details |
|---|---|
| **Maintainer** | LangChain Inc. |
| **Language** | Python, TypeScript |
| **License** | MIT |
| **GitHub** | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) |
| **Docs** | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) |

#### 2.1.1 Agent Execution Model

LangGraph models agent workflows as **directed cyclic graphs** using three primitives:

- **State (`StateGraph`):** A centralized, typed data schema (typically `TypedDict` or Pydantic) representing the shared snapshot of the application. Reducer functions govern how state keys are updated across nodes.
- **Nodes:** Python functions (or runnables) that receive the current state, perform work (LLM calls, tool execution, data processing), and return partial state updates.
- **Edges:** Control flow connectors—fixed (unconditional), conditional (dynamic routing based on state inspection), or `Command`-based (simultaneously returning state updates and routing directives).

The execution model follows Google's **Pregel** system with **discrete super-steps:**
1. Nodes scheduled concurrently run within the same super-step.
2. A node activates when it receives messages on incoming edges.
3. Execution terminates when all nodes are inactive and no messages remain in transit.
4. Graphs must be compiled (`.compile()`) before execution, validating structure and binding checkpointers.

```
                    ┌──────────────────────────┐
                    │      StateGraph           │
                    │  (TypedDict / Pydantic)   │
                    └──────────┬───────────────┘
                               │ .compile()
                    ┌──────────▼───────────────┐
                    │   CompiledStateGraph      │
                    │  ┌─────┐  ┌─────┐        │
                    │  │Node1│→→│Node2│──┐      │
                    │  └─────┘  └──┬──┘  │      │
                    │      ↑       │     ▼      │
                    │      │  ┌────▼──┐ ┌────┐  │
                    │      └──│Node3  │ │END │  │
                    │         └───────┘ └────┘  │
                    └───────────────────────────┘
```

#### 2.1.2 Tool & Context Integration

- **Tools** are Python functions decorated with `@tool` or wrapped via `StructuredTool`. LangChain provides a large ecosystem of pre-built tool integrations.
- **Context/Memory:** Short-term memory lives in the graph state. Long-term memory uses vector stores (FAISS, Chroma, Pinecone, Weaviate) for RAG. LangGraph's `MemorySaver` persists thread state across turns.
- **MCP Support:** LangChain provides MCP client integration via `langchain-mcp-adapters`.

#### 2.1.3 State Persistence & Checkpointing

LangGraph checkpoints state **after every node execution** using pluggable backends:

| Backend | Use Case | Limitations |
|---|---|---|
| `MemorySaver` | Development/testing | In-memory only; lost on restart |
| `SqliteSaver` | Single-process persistence | SQLite locking under concurrency |
| `PostgresSaver` | Production multi-process | Requires connection pooling (PgBouncer); checkpoint tables can balloon to GBs |

**Key issues:**
- Full state serialization via `JsonPlusSerializer` at every step creates write-heavy I/O.
- Schema migrations on live threads require careful handling of default values.
- No built-in TTL/retention policy for checkpoint data.

#### 2.1.4 Model Abstraction

LangChain provides broad model abstraction via `BaseChatModel`:
- **Cloud providers:** OpenAI, Anthropic, Google (Vertex/Gemini), Azure, Cohere, Mistral, etc.
- **Local models:** Ollama, llama.cpp (via `ChatOllama`), HuggingFace (via `ChatHuggingFace`), vLLM.
- **ONNX/ExecuTorch:** No native integration; requires custom wrapper implementing `BaseChatModel`.

The abstraction is **loose coupling**—switching providers requires only changing the model class and API key configuration.

#### 2.1.5 Distributed Execution

**Open-source LangGraph is fundamentally single-process.** Key limitations:
- No built-in distributed orchestration, heartbeats, supervision trees, or dead-letter queues.
- Race conditions on thread resumption across distributed workers (no distributed locking).
- Fan-out state merging during mid-execution interruptions is ambiguous across nodes.

**LangGraph Platform** (commercial) adds: queue-based task distribution, horizontal scaling, automatic retry, and cron-based scheduling—but this is a proprietary cloud service.

#### 2.1.6 Hardware Awareness

**None.** The execution engine treats all nodes as generic Python callables. It cannot:
- Distinguish I/O-bound vs. CPU/GPU-bound steps.
- Route inference to specific accelerators.
- Page large state objects to external stores based on RAM pressure.
- Schedule work based on GPU memory availability.

#### 2.1.7 Interoperability

- **MCP:** Client-side integration via adapters; can consume MCP tools.
- **A2A:** No native support; would require custom implementation.
- **Cross-framework:** LangChain tools can be imported into ADK and CrewAI via adapters.

#### 2.1.8 Limitations for Our Research

LangGraph is the most mature stateful agent graph engine, but it is an **orchestration library, not a runtime.** It:
- Depends entirely on the host Python process for execution.
- Has no concept of compute resources, networking, or deployment.
- Cannot survive host OS restarts without external infrastructure.
- Requires external systems (Temporal, Celery, Ray) for true distributed execution.

---

### 2.2 Google Agent Development Kit (ADK)

| Attribute | Details |
|---|---|
| **Maintainer** | Google DeepMind / Google Cloud |
| **Language** | Python, TypeScript, Go, Java |
| **License** | Apache 2.0 |
| **GitHub** | [google/adk-python](https://github.com/google/adk-python) |
| **Docs** | [adk.dev](https://adk.dev) |

#### 2.2.1 Agent Execution Model

ADK uses `LlmAgent` as the fundamental execution unit, combining:
- **AI Model:** The reasoning engine (optimized for Gemini but model-agnostic).
- **Instructions:** System prompts defining persona, goals, and constraints.
- **Tools:** Capabilities for environment interaction.

**Execution flow:**
1. Agent receives user request via session.
2. LLM evaluates instructions, selects tools, and executes an autonomous reasoning loop.
3. Multi-agent architectures delegate sub-tasks via structured task modes.
4. Workflow Runtime handles deterministic pipelines: routing, fan-out/fan-in, loops, retries.

**Human-in-the-loop** via Tool Confirmation flows that pause execution for user validation.

```
    ┌────────────────────────────────────────┐
    │           ADK Runtime                  │
    │  ┌──────────┐    ┌───────────────┐     │
    │  │  Session  │◄──│  State Store   │     │
    │  │  Manager  │    │ (SQLite/Cloud)│     │
    │  └────┬─────┘    └───────────────┘     │
    │       │                                │
    │  ┌────▼─────────────────────────┐      │
    │  │  Workflow Runtime (Graph)     │      │
    │  │  ┌────────┐  ┌────────┐      │      │
    │  │  │Agent A │──│Agent B │      │      │
    │  │  │(Router)│  │(Worker)│      │      │
    │  │  └────────┘  └───┬────┘      │      │
    │  │       ↕          │           │      │
    │  │  ┌────────┐  ┌───▼────┐      │      │
    │  │  │Agent C │  │Agent D │      │      │
    │  │  │(Critic)│  │(Output)│      │      │
    │  │  └────────┘  └────────┘      │      │
    │  └──────────────────────────────┘      │
    └────────────────────────────────────────┘
```

#### 2.2.2 Tool & Context Integration

ADK supports a rich tool ecosystem:
- **Python/TS/Go/Java functions:** Native code blocks invoked via function calling.
- **OpenAPI specs:** Seamless external API integration.
- **MCP tools:** First-class support for MCP server connections.
- **Ecosystem adapters:** Can import LangChain and CrewAI tools.

**Context management:** `ToolContext` and `session.state` for short-term; Firestore/Redis for long-term persistence.

#### 2.2.3 State Persistence & Checkpointing

- **Development:** SQLite-backed local session stores.
- **Production:** Firestore, Redis, or SQL-backed persistence layers.
- **Evolution:** Moved from Pickle-based serialization to structured JSON schemas.
- **Checkpointing:** Supports pause/resume for long-running tasks and human-in-the-loop workflows.

#### 2.2.4 Model Abstraction

- **Optimized for Gemini** but designed as model-agnostic.
- Supports any model exposed via OpenAI-compatible API endpoints (Ollama, vLLM, LM Studio).
- No native ONNX/ExecuTorch integration.
- Local models struggle with ADK's strict schema requirements for tool calling.

#### 2.2.5 Distributed Execution

- **Local development:** `adk run` CLI or `adk web` browser UI.
- **Production:** Containerized deployment to Cloud Run, GKE, or Vertex AI Agent Platform.
- No native distributed agent orchestration across machines—relies on cloud infrastructure for scaling.

#### 2.2.6 Hardware Awareness

**None.** ADK delegates inference to model endpoints. No GPU/NPU awareness, VRAM management, or accelerator routing.

#### 2.2.7 Interoperability

- **MCP:** First-class client support.
- **A2A:** Native support as both client and server (Google co-designed A2A).
- **Cross-framework:** Adapters for LangChain, CrewAI tool import.

#### 2.2.8 Limitations for Our Research

ADK is well-designed for agent *application* development but remains a Python/TS library:
- Cloud-centric deployment model (Cloud Run, GKE, Vertex).
- No local distributed execution primitive.
- Tight coupling to Google Cloud ecosystem for production features.
- Not a runtime—relies on underlying container/serverless platforms.

---

### 2.3 Microsoft Semantic Kernel

| Attribute | Details |
|---|---|
| **Maintainer** | Microsoft |
| **Language** | C# (.NET), Python, Java |
| **License** | MIT |
| **GitHub** | [microsoft/semantic-kernel](https://github.com/microsoft/semantic-kernel) |
| **Docs** | [learn.microsoft.com/semantic-kernel](https://learn.microsoft.com/en-us/semantic-kernel/) |

#### 2.3.1 Agent Execution Model

Semantic Kernel uses the **Kernel** object as the central orchestrator and dependency injection container:

1. **Intent Interpretation:** Agent receives user prompt or inter-agent message.
2. **Context & Policy Evaluation:** Evaluates persona, system instructions, memory context.
3. **Function Choice & Planning:** Uses native LLM function calling to determine tool needs.
4. **Middleware & Filters:** Intercepts execution for logging, security, telemetry.

Agent types: `ChatCompletionAgent`, `OpenAIAssistantAgent`, custom agents via `Agent` base class.

**Multi-agent collaboration** via `AgentGroupChat` with selection and termination strategies.

```
    ┌──────────────────────────────────────┐
    │            Kernel                     │
    │  ┌──────────┐   ┌────────────────┐   │
    │  │AI Services│   │   Plugins      │   │
    │  │(Chat,Emb) │   │ ┌──────────┐  │   │
    │  └─────┬────┘   │ │KernelFunc│  │   │
    │        │        │ │KernelFunc│  │   │
    │  ┌─────▼────┐   │ └──────────┘  │   │
    │  │ Filters   │   └────────────────┘   │
    │  │(Pre/Post) │                        │
    │  └─────┬────┘   ┌────────────────┐   │
    │        │        │   Memory       │   │
    │  ┌─────▼────┐   │ (Vector Store) │   │
    │  │ Planner   │   └────────────────┘   │
    │  └──────────┘                         │
    └──────────────────────────────────────┘
```

#### 2.3.2 Tool & Context Integration

- **Plugins:** Collections of `KernelFunction`s with `[KernelFunction]` attribute and descriptions.
- **Dynamic invocation:** `FunctionChoiceBehavior.Auto()` advertises plugins to the model.
- **Execution loop:** Model emits tool-call request → Kernel executes native function → result feeds back to context.
- **Memory:** Vector store abstractions (Qdrant, Redis, Azure AI Search, Chroma) for RAG.

#### 2.3.3 State Persistence & Checkpointing

- **Chat history** managed via `ChatHistory` objects—persisted externally by the developer.
- **No built-in checkpoint/resume** for multi-step workflows (unlike LangGraph).
- **Vector stores** for long-term semantic memory.
- **Distributed state** offloaded to Redis Stack or Azure Cosmos DB.

#### 2.3.4 Model Abstraction

Strong model abstraction via connector interfaces (`IChatCompletionService`):
- **Cloud:** Azure OpenAI, OpenAI, Google Gemini, Anthropic, Mistral.
- **Local:** Ollama, LM Studio, ONNX Runtime (native DirectML/HuggingFace), Foundry Local.
- **ONNX Runtime** integration allows direct in-process inference with hardware acceleration.
- Switching providers requires zero changes to orchestration or plugin logic.

#### 2.3.5 Distributed Execution

- No native distributed execution primitives.
- Multi-agent collaboration is in-process.
- Microsoft converging toward **Microsoft Agent Framework (MAF)** which unifies SK + AutoGen for distributed agent runtimes.
- Enterprise scaling relies on Azure infrastructure.

#### 2.3.6 Hardware Awareness

- **ONNX Runtime** connector provides some hardware awareness (DirectML for GPU acceleration).
- No native GPU/NPU routing or VRAM management in the orchestration layer.
- Hardware awareness delegated to the inference runtime (Ollama, ONNX, etc.).

#### 2.3.7 Interoperability

- **MCP:** MCP server connector for consuming MCP-exposed tools.
- **A2A:** Supported via Microsoft Agent Framework integration.
- **Cross-framework:** SK plugins can be exposed as OpenAI-compatible function schemas.

#### 2.3.8 Limitations for Our Research

Semantic Kernel is the most enterprise-mature SDK but remains an orchestration library:
- Tightly coupled to Azure/Microsoft ecosystem for production scaling.
- No built-in workflow checkpointing (added via MAF).
- In-process execution model limits distributed scenarios.
- Best suited as an "AI middleware" layer, not a standalone runtime.

---

### 2.4 AutoGen / AG2

| Attribute | Details |
|---|---|
| **Maintainer** | AG2 (community, forked from Microsoft) / Microsoft (AutoGen v0.4+) |
| **Language** | Python |
| **License** | Apache 2.0 (AG2) / MIT (Microsoft AutoGen) |
| **GitHub** | [ag2ai/ag2](https://github.com/ag2ai/ag2) / [microsoft/autogen](https://github.com/microsoft/autogen) |
| **Docs** | [ag2.ai](https://ag2.ai) / [microsoft.github.io/autogen](https://microsoft.github.io/autogen/) |

#### 2.4.1 Agent Execution Model

The ecosystem split into two lineages:

**AG2 Classic (formerly AutoGen 0.2):**
- Synchronous, peer-to-peer conversational loops.
- `ConversableAgent`, `AssistantAgent`, `UserProxyAgent`, `GroupChat`.
- Execution via `agent.initiate_chat()` method calls.

**Modern AG2 / AutoGen v0.4+ (`autogen-core`):**
- Decoupled, asynchronous, event-driven architecture.
- Agents as independent entities with private state, communicating exclusively via messages.
- **Agent Runtime** acts as message broker routing between agents.
- Process-agnostic: same-process (`LocalLink`) or cross-machine (`WsLink` via WebSockets).

**AG2 Network (`ag2.network`):**
- Central **Hub** with agent registry, channel transcripts, write-ahead logs.
- Enforced turn-taking, access controls, inbox limits.
- Deterministic or dynamic execution ordering.

#### 2.4.2 Tool & Context Integration

- **Code execution:** Native Docker-sandboxed code execution for generated Python/shell scripts.
- **Function registration:** Tools registered via `@agent.register_for_llm()` decorator.
- **MCP support:** MCP client integration for tool discovery.
- **Memory:** Teachability modules for cross-session learning; custom RAG integrations.

#### 2.4.3 State Persistence & Checkpointing

- **First-class state objects** in modern versions—serializable agent state beyond raw transcripts.
- **Checkpoint snapshots** at event milestones enabling crash recovery and resume.
- **Hub audit trails** (AG2 Network) provide write-ahead logging for state reconstruction.
- Classic AG2: No built-in checkpointing; state lived in Python memory.

#### 2.4.4 Model Abstraction

- Unified client interface standardizing parameters across providers.
- Supports OpenAI, Anthropic, Gemini, local models (Ollama, vLLM).
- **Fallback and routing:** Dynamic model routing—simple tasks to lightweight models, complex reasoning to frontier models.
- No native ONNX/ExecuTorch integration.

#### 2.4.5 Distributed Execution

The most architecturally advanced for distribution among the agent frameworks:
- **Actor-model message passing** via gRPC, Redis, or Azure Service Bus.
- Agents as independent microservices across containers/nodes.
- `WsLink` for cross-machine WebSocket communication.
- AG2 Hub provides centralized governance for distributed agent networks.

#### 2.4.6 Hardware Awareness

- **Compute-aware tool execution:** Can offload code execution to GPU-enabled Docker containers.
- **Resource balancing:** Can integrate with hardware-accelerated serving engines.
- No native VRAM monitoring, NPU detection, or accelerator affinity scheduling.

#### 2.4.7 Interoperability

- **MCP:** Client-side integration.
- **A2A:** Supported via Microsoft Agent Framework.
- **Cross-framework:** AG2 can orchestrate agents from other frameworks as sub-agents.

#### 2.4.8 Limitations for Our Research

AutoGen/AG2 has the most distributed-aware architecture, but:
- The ecosystem split (AG2 vs. Microsoft AutoGen) creates fragmentation.
- Distributed features are still evolving and not production-hardened.
- No hardware scheduling, compute topology awareness, or resource negotiation.
- Still a Python library requiring external infrastructure for scaling.

---

### 2.5 CrewAI

| Attribute | Details |
|---|---|
| **Maintainer** | CrewAI Inc. |
| **Language** | Python |
| **License** | MIT |
| **GitHub** | [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) |
| **Docs** | [docs.crewai.com](https://docs.crewai.com) |

#### 2.5.1 Agent Execution Model

CrewAI splits logic into two pillars:

- **Flows (Backbone):** Macro-level process definition managing event-driven architectures, state persistence, conditional logic, loops, and branching.
- **Crews (Intelligence):** Groups of role-playing agents (`role`, `goal`, `backstory`) with tools and memory, executing tasks collaboratively.

**Process orchestration modes:**
- **Sequential:** Linear task execution; output of one feeds as context to next.
- **Hierarchical:** Automated manager agent (via `manager_llm`) that plans, delegates, and validates.

**Execution hooks** intercept LLM and tool calls for logging, safety filtering, token tracking, and human-in-the-loop approval.

```
    ┌──────────────────────────────────────────┐
    │              Flow                         │
    │  ┌──────────────────────────────────┐     │
    │  │  Step 1: Data Ingestion          │     │
    │  └───────────┬──────────────────────┘     │
    │              │                             │
    │  ┌───────────▼──────────────────────┐     │
    │  │  Step 2: Crew Execution           │     │
    │  │  ┌─────────┐  ┌─────────┐        │     │
    │  │  │Researcher│→→│Analyst  │        │     │
    │  │  │(Agent)   │  │(Agent)  │        │     │
    │  │  └─────────┘  └────┬────┘        │     │
    │  │                    │             │     │
    │  │              ┌─────▼────┐        │     │
    │  │              │Writer    │        │     │
    │  │              │(Agent)   │        │     │
    │  │              └──────────┘        │     │
    │  └──────────────────────────────────┘     │
    │              │                             │
    │  ┌───────────▼──────────────────────┐     │
    │  │  Step 3: Output Formatting        │     │
    │  └──────────────────────────────────┘     │
    └──────────────────────────────────────────┘
```

#### 2.5.2 Tool & Context Integration

- **Pydantic-powered tools:** Structured input/output schemas; JSON serializable.
- **Built-in toolkits:** Web search, file I/O, enterprise connectors.
- **Tool call hooks:** Pre/post interception for PII redaction, sandboxing, approval gates.
- **Memory:** Short-term (ChromaDB vectors), long-term (SQLite3), entity memory (vector-based).

#### 2.5.3 State Persistence & Checkpointing

- **Flows:** `@persist()` decorator enables state checkpointing with Pydantic schemas.
- **Defaults:** ChromaDB for vector memory, SQLite3 for long-term storage—both local-first.
- **Production challenges:** Ephemeral containers wipe local files; SQLite locking under concurrency.
- **Production mitigation:** Must swap to distributed vector DBs (Qdrant, Pinecone) and PostgreSQL/Redis.
- No native deep cyclic recovery or distributed rollbacks.

#### 2.5.4 Model Abstraction

- Model-agnostic via LiteLLM and native client integrations.
- Supports OpenAI, Anthropic, Google Gemini, local models (Ollama, vLLM).
- **Abstraction tax:** High-level agent definitions (`role`/`goal`/`backstory`) trade fine-grained prompt control for ease of use.
- **Local model challenges:** Models <30B parameters frequently fail structured tool-calling, causing `ValueError` exceptions and infinite loops.

#### 2.5.5 Distributed Execution

- **No native distributed execution.** Single-process Python runtime.
- CrewAI Enterprise (commercial) adds managed deployment and scaling.
- Flow state persistence does not include distributed locking or consensus.

#### 2.5.6 Hardware Awareness

**None.** No GPU/NPU awareness, VRAM management, or hardware-aware scheduling.

#### 2.5.7 Interoperability

- **MCP:** Limited/community MCP client support.
- **A2A:** No native support.
- **Cross-framework:** Tools can be shared with LangChain/ADK via adapters.

#### 2.5.8 Limitations for Our Research

CrewAI excels at rapid prototyping with its intuitive mental model but:
- Weakest distributed story among the frameworks analyzed.
- Local-first storage defaults collapse in containerized environments.
- Heavy token overhead from multi-layered abstractions.
- Not a runtime—a high-level Python orchestration library.

---

### 2.6 Model Context Protocol (MCP)

| Attribute | Details |
|---|---|
| **Maintainer** | Anthropic (originated); open governance |
| **Type** | Open protocol specification |
| **Spec** | [modelcontextprotocol.io](https://modelcontextprotocol.io) |
| **GitHub** | [modelcontextprotocol](https://github.com/modelcontextprotocol) |
| **SDKs** | TypeScript, Python, Java, Go, Rust, C# |

#### 2.6.1 Architecture: Client-Host-Server Model

MCP defines a standardized interface between AI applications and external data/tools, solving the N×M integration problem with a uniform N+M approach.

```
    ┌─────────────────────────────────────────────┐
    │                MCP Host                      │
    │  (AI IDE, Agent Application, Chat UI)        │
    │                                              │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
    │  │MCP Client│  │MCP Client│  │MCP Client│   │
    │  │    1:1   │  │    1:1   │  │    1:1   │   │
    │  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
    └───────┼─────────────┼─────────────┼──────────┘
            │             │             │
     ┌──────▼──────┐ ┌────▼────┐ ┌─────▼─────┐
     │ MCP Server  │ │MCP Server│ │MCP Server │
     │ (Database)  │ │ (APIs)   │ │  (Files)  │
     └─────────────┘ └─────────┘ └───────────┘
```

- **Host:** AI application managing the LLM and user interaction.
- **Client:** Maintains isolated 1:1 connection with a single MCP server (JSON-RPC 2.0 over stdio or HTTP/SSE).
- **Server:** Lightweight program exposing capabilities via standardized primitives.

#### 2.6.2 Key Primitives

| Primitive | Direction | Purpose |
|---|---|---|
| **Resources** | Server → Client | Read-only context data (files, schemas, logs) |
| **Prompts** | Server → Client | Templated workflows and interaction patterns |
| **Tools** | Server → Client (model-driven) | Executable functions for real-world actions |

**Tool flow:** `tools/list` (discovery) → LLM selects tool → `tools/call` (execution) → result returned to model context.

#### 2.6.3 Relationship to Frameworks

MCP is a **vertical protocol** (agent-to-tool/data). It is:
- Consumed by all major frameworks (LangChain, ADK, SK, AutoGen, CrewAI).
- Not an execution framework itself—no agent lifecycle, scheduling, or state management.
- Transport-agnostic: stdio for local, Streamable HTTP for remote.

#### 2.6.4 Limitations for Our Research

MCP standardizes the tool integration surface but:
- Defines no execution model, no scheduling, no state management.
- 1:1 client-server topology limits complex service mesh patterns.
- No native support for distributed tool discovery or load balancing.
- Security model relies entirely on the host application.

---

### 2.7 Agent-to-Agent Protocol (A2A)

| Attribute | Details |
|---|---|
| **Maintainer** | Linux Foundation (originated at Google) |
| **Type** | Open protocol specification |
| **Spec** | [a2a-protocol.org](https://a2a-protocol.org) |
| **GitHub** | [a2a-protocol](https://github.com/google/A2A) |

#### 2.7.1 Architecture

A2A enables **cross-framework, cross-vendor agent collaboration** via a client-server model:

```
    ┌────────────────────┐          ┌────────────────────┐
    │   A2A Client       │          │   A2A Server       │
    │ (Orchestrator      │  JSON-   │ (Specialized       │
    │  Agent)            │  RPC/    │  Remote Agent)     │
    │                    │  HTTP(S) │                    │
    │  sends Task ──────►│─────────►│  executes Task     │
    │  receives Artifacts│◄─────────│◄── returns result  │
    └────────────────────┘          └────────────────────┘
                                            │
                                    ┌───────▼───────┐
                                    │  Agent Card   │
                                    │ (.well-known/ │
                                    │  agent.json)  │
                                    └───────────────┘
```

#### 2.7.2 Core Concepts

- **Agent Cards:** JSON metadata at `/.well-known/agent-card.json` describing agent capabilities, skills, security schemes, and supported media types.
- **Tasks:** Stateful, long-running operations with explicit lifecycle (initialized → running → input-required → completed → failed).
- **Messages & Parts:** Context exchange via content blocks (text, JSON, files, media streams).
- **Artifacts:** Final or intermediate deliverables from task execution.

#### 2.7.3 Communication Modes

| Mode | Mechanism | Use Case |
|---|---|---|
| Synchronous | HTTP request/response | Quick query-answer |
| Streaming | Server-Sent Events (SSE) | Real-time tokens, progress |
| Async Push | Webhook notifications | Long-running decoupled tasks |

#### 2.7.4 A2A vs. MCP Relationship

| Dimension | MCP (Vertical) | A2A (Horizontal) |
|---|---|---|
| **Connection** | Agent → Tool/Data | Agent → Agent |
| **Autonomy** | Tool is passive (invoked) | Remote agent is autonomous |
| **Scope** | Single agent's capabilities | Cross-boundary collaboration |
| **Example** | Agent queries SQL database | Orchestrator delegates to specialist agent |

These protocols are **complementary:** an agent uses MCP to access its tools and A2A to delegate to peer agents.

#### 2.7.5 Enterprise Features

- **Authentication:** OpenAPI-compliant security schemes, OAuth/Bearer tokens.
- **Identity:** JSON Web Signatures (JWS) for Agent Card verification.
- **Auditability:** Task lifecycle tracking.

#### 2.7.6 Limitations for Our Research

A2A standardizes agent-to-agent communication but:
- Is a communication protocol, not an execution runtime.
- No agent lifecycle management, deployment, or scheduling.
- Still early-stage; few production-grade implementations.
- No hardware awareness, resource negotiation, or compute placement.

---

## 3. Architecture Comparison Diagrams

### 3.1 Conceptual Layers

```
┌────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                               │
│  ┌──────────┐ ┌─────────┐ ┌──────┐ ┌────────┐ ┌──────┐          │
│  │LangGraph │ │Google   │ │Sem.  │ │AutoGen/│ │CrewAI│          │
│  │          │ │ADK      │ │Kernel│ │AG2     │ │      │          │
│  └────┬─────┘ └────┬────┘ └──┬───┘ └───┬────┘ └──┬───┘          │
│       │             │         │         │         │               │
├───────┼─────────────┼─────────┼─────────┼─────────┼───────────────┤
│       │      PROTOCOL LAYER   │         │         │               │
│       │    ┌────────┴──┐  ┌───┴────┐    │         │               │
│       │    │    MCP     │  │   A2A  │    │         │               │
│       │    │(Tool/Data) │  │(Agent) │    │         │               │
│       │    └────────────┘  └────────┘    │         │               │
├───────┼──────────────────────────────────┼─────────┼───────────────┤
│       │        INFERENCE LAYER           │         │               │
│  ┌────▼──────────────────────────────────▼─────────▼──────────┐   │
│  │  Cloud APIs (OpenAI, Gemini, Claude, Mistral, Cohere)      │   │
│  │  Local Engines (Ollama, vLLM, llama.cpp, ONNX Runtime)     │   │
│  └────────────────────────────────────────────────────────────┘   │
├───────────────────────────────────────────────────────────────────┤
│                    RUNTIME LAYER  ← THE GAP                       │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  Python/Node.js Process  │  .NET CLR  │  Container/VM     │   │
│  │  (No hardware awareness, no native scheduling,             │   │
│  │   no distributed primitives, no resource management)       │   │
│  └────────────────────────────────────────────────────────────┘   │
├───────────────────────────────────────────────────────────────────┤
│                    OS / HARDWARE LAYER                             │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │ CPU  │  │ GPU  │  │ NPU  │  │ RAM  │  │ Disk │               │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘               │
└───────────────────────────────────────────────────────────────────┘
```

The **RUNTIME LAYER** gap is the key finding: no framework provides native scheduling, hardware awareness, process isolation, or distributed execution primitives. They all sit atop traditional language runtimes.

---

## 4. Comparative Evaluation Matrix

Rating scale: ★☆☆☆☆ (1) to ★★★★★ (5)

| # | Dimension | LangGraph | Google ADK | Semantic Kernel | AutoGen/AG2 | CrewAI | MCP | A2A |
|---|---|---|---|---|---|---|---|---|
| 1 | **Developer Experience** | ★★★★☆ | ★★★★☆ | ★★★★☆ | ★★★☆☆ | ★★★★★ | ★★★★☆ | ★★★☆☆ |
| 2 | **Local Execution / Offline** | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | ★★☆☆☆ |
| 3 | **Distributed Execution** | ★★☆☆☆ | ★★☆☆☆ | ★★☆☆☆ | ★★★☆☆ | ★☆☆☆☆ | ★☆☆☆☆ | ★★★☆☆ |
| 4 | **AI-Native Capabilities** | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| 5 | **Data Locality / Sync** | ★★★☆☆ | ★★★☆☆ | ★★★☆☆ | ★★☆☆☆ | ★★☆☆☆ | ★★★☆☆ | ★★☆☆☆ |
| 6 | **Networking Overhead** | ★★★★☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ | ★★★★☆ | ★★★★★ | ★★★☆☆ |
| 7 | **Hardware Awareness** | ★☆☆☆☆ | ★☆☆☆☆ | ★★☆☆☆ | ★★☆☆☆ | ★☆☆☆☆ | ★☆☆☆☆ | ★☆☆☆☆ |
| 8 | **Security / Isolation** | ★★☆☆☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ | ★★★☆☆ | ★★★★☆ |
| 9 | **Cross-Platform Portability** | ★★★☆☆ | ★★★★☆ | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | ★★★★★ |
| 10 | **Scalability / Performance** | ★★★☆☆ | ★★★☆☆ | ★★★☆☆ | ★★★☆☆ | ★★☆☆☆ | ★★★★☆ | ★★★★☆ |
| 11 | **Extensibility / Modularity** | ★★★★★ | ★★★★☆ | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★★ | ★★★★☆ |
| 12 | **Deployment / Distribution** | ★★★☆☆ | ★★★★☆ | ★★★★☆ | ★★☆☆☆ | ★★☆☆☆ | ★★★★☆ | ★★★☆☆ |

### Rating Rationale Summary

| Framework | Strengths | Weaknesses |
|---|---|---|
| **LangGraph** | Best-in-class stateful graph engine; rich ecosystem; excellent checkpointing | Single-process; no distributed primitives; no hardware awareness |
| **Google ADK** | Multi-language; MCP+A2A native; Google Cloud integration; clean architecture | Cloud-centric; limited local model support; young ecosystem |
| **Semantic Kernel** | Enterprise-grade; ONNX support; multi-language (.NET, Python, Java); strong typing | No workflow checkpointing; Azure-coupled for scaling; complex API surface |
| **AutoGen/AG2** | Most distributed-aware; actor model; Hub governance; code execution | Ecosystem fragmentation; unstable APIs; steep learning curve |
| **CrewAI** | Best DX for prototyping; intuitive role-based agents; rapid setup | Weakest distributed/persistence story; token overhead; local-only defaults |
| **MCP** | Universal tool protocol; wide adoption; language-agnostic; lightweight | Protocol only—no execution; 1:1 topology; no scheduling |
| **A2A** | Cross-vendor agent collaboration; stateful tasks; enterprise security | Early-stage; protocol only; few implementations; no resource negotiation |

---

## 5. Cross-Cutting Analysis

### 5.1 Agent Execution Models Compared

| Aspect | LangGraph | ADK | Semantic Kernel | AutoGen/AG2 | CrewAI |
|---|---|---|---|---|---|
| **Graph type** | Directed cyclic | Hierarchical + graph | Sequential pipeline | Conversation/actor | Sequential/hierarchical |
| **Concurrency** | Super-step parallelism | Workflow fan-out | In-process | Async message passing | Sequential only |
| **Cycles** | Native | Supported | Limited | Via conversation loops | Not native |
| **Human-in-the-loop** | Interrupts + state edit | Tool confirmation | Filter pipeline | Turn-based | Hooks |
| **Compile step** | Required (`.compile()`) | Implicit | Implicit | Implicit | Implicit |

### 5.2 State & Persistence Compared

| Aspect | LangGraph | ADK | Semantic Kernel | AutoGen/AG2 | CrewAI |
|---|---|---|---|---|---|
| **Auto-checkpoint** | ✅ Every node | ✅ Sessions | ❌ Manual | ✅ (v0.4+) | ✅ (`@persist`) |
| **Default backend** | In-memory | SQLite | None | In-memory | SQLite + ChromaDB |
| **Production backend** | PostgreSQL | Firestore/Redis | Redis/Cosmos DB | Redis/custom | Qdrant/PostgreSQL |
| **Schema migration** | Manual | JSON schema evolution | N/A | Manual | Pydantic validation |
| **Process restart survival** | ✅ (with persistent checkpointer) | ✅ | ❌ (developer-managed) | ✅ (v0.4+) | ✅ (with prod backend) |

### 5.3 Model Support Matrix

| Provider | LangGraph | ADK | Semantic Kernel | AutoGen/AG2 | CrewAI |
|---|---|---|---|---|---|
| OpenAI | ✅ | ✅ | ✅ | ✅ | ✅ |
| Anthropic | ✅ | ✅ | ✅ | ✅ | ✅ |
| Google Gemini | ✅ | ✅ (native) | ✅ | ✅ | ✅ |
| Ollama (local) | ✅ | ✅ | ✅ | ✅ | ✅ |
| vLLM | ✅ | ✅ | ✅ | ✅ | ✅ |
| llama.cpp | ✅ (via wrapper) | ⚠️ (API compat) | ✅ (via Ollama) | ⚠️ | ⚠️ |
| ONNX Runtime | ❌ | ❌ | ✅ (native) | ❌ | ❌ |
| ExecuTorch | ❌ | ❌ | ❌ | ❌ | ❌ |

### 5.4 Protocol Adoption

| Protocol | LangGraph | ADK | Semantic Kernel | AutoGen/AG2 | CrewAI |
|---|---|---|---|---|---|
| **MCP (client)** | ✅ | ✅ | ✅ | ✅ | ⚠️ (community) |
| **MCP (server)** | ❌ | ❌ | ⚠️ | ❌ | ❌ |
| **A2A (client)** | ❌ | ✅ | ⚠️ (via MAF) | ⚠️ | ❌ |
| **A2A (server)** | ❌ | ✅ | ⚠️ (via MAF) | ❌ | ❌ |

---

## 6. Limitations for AI-Native Runtime Research

### 6.1 The "Wrapper vs. Runtime" Problem

Every framework in this analysis is fundamentally a **library/SDK** that wraps LLM API calls with orchestration logic. None qualifies as a "runtime" in the systems programming sense:

| Runtime Capability | Traditional Runtimes (JVM, V8, BEAM) | Agent Frameworks |
|---|---|---|
| Process scheduling | ✅ | ❌ |
| Memory management | ✅ | ❌ |
| I/O multiplexing | ✅ | ❌ (via host runtime) |
| Thread/coroutine management | ✅ | ❌ (via host runtime) |
| Hardware abstraction | ✅ | ❌ |
| Fault isolation | ✅ | ❌ |
| Code loading / linking | ✅ | ❌ |
| Network stack | ✅ | ❌ |
| Security sandbox | ✅ | ❌ |

### 6.2 Specific Gaps

1. **No compute placement:** No framework can decide where to run a task based on hardware availability (GPU, NPU, VRAM, network topology).

2. **No native networking:** All networking is done via HTTP/gRPC to cloud APIs. No peer-to-peer, no mesh networking, no local-first sync.

3. **No resource budgeting:** No framework tracks token costs, inference latency, memory usage, or compute time as first-class schedulable resources.

4. **No deployment model:** Frameworks produce Python/TS/C# code that must be manually containerized and deployed via external tooling (Docker, K8s, Cloud Run).

5. **No offline-first design:** All frameworks assume network access to LLM APIs. Local model support exists but is secondary and often broken for complex workflows.

6. **No cross-device execution:** No framework can seamlessly move agent execution between phone, laptop, and cloud based on context.

7. **No data sovereignty:** State persistence is bolted on; no native CRDT, conflict resolution, or data locality guarantees.

### 6.3 What These Frameworks DO Well (and Should Be Learned From)

| Pattern | Framework | Value for Runtime Research |
|---|---|---|
| Stateful graph execution | LangGraph | Graph-based agent workflows with checkpointing are the right abstraction |
| Protocol standardization | MCP, A2A | Tool and agent communication should be protocol-based, not framework-locked |
| Model abstraction | Semantic Kernel | Connector-based model abstraction enables hardware heterogeneity |
| Actor-model agents | AutoGen/AG2 | Message-passing agents map well to distributed systems |
| Role-based composition | CrewAI | High-level agent definitions accelerate development |
| Multi-language support | ADK, SK | Runtime should not be Python-only |

---

## 7. Key Takeaways & Recommendations

### 7.1 For the AI-Native Runtime Initiative

1. **Adopt graph-based execution as the core model.** LangGraph's `StateGraph` with super-steps proves that directed cyclic graphs are the right primitive for agent workflows. The runtime should provide this natively with distributed execution.

2. **Implement MCP and A2A as first-class protocols.** These are becoming industry standards. The runtime should be both MCP server and client, and speak A2A for agent collaboration.

3. **Hardware-aware scheduling is the key differentiator.** No framework today can route inference to the right accelerator, balance VRAM across agents, or schedule based on compute topology. This is the greenfield opportunity.

4. **State persistence must be built into the runtime, not bolted on.** Checkpoint serialization, distributed locking, CRDT-based sync, and automatic schema migration should be runtime primitives.

5. **Local-first with cloud-burst.** The runtime should work fully offline with local models and transparently burst to cloud inference when needed—the inverse of current frameworks' cloud-first design.

6. **Multi-language is essential.** Python-only limits adoption. Follow Semantic Kernel (C#/Python/Java) and ADK (Python/TS/Go/Java) patterns.

### 7.2 Integration Strategy

Rather than replacing these frameworks, the runtime should:
- **Host them:** Enable running LangGraph/ADK/SK agents as runtime-managed workloads.
- **Extend them:** Provide hardware scheduling, distributed execution, and persistence that they lack.
- **Interoperate with them:** Speak MCP and A2A natively.

---

## 8. References

### Official Documentation
- LangGraph: https://langchain-ai.github.io/langgraph/
- LangChain: https://docs.langchain.com/
- Google ADK: https://adk.dev / https://github.com/google/adk-python
- Microsoft Semantic Kernel: https://learn.microsoft.com/en-us/semantic-kernel/
- AutoGen: https://microsoft.github.io/autogen/
- AG2: https://ag2.ai / https://github.com/ag2ai/ag2
- CrewAI: https://docs.crewai.com / https://github.com/crewAIInc/crewAI
- MCP Specification: https://modelcontextprotocol.io / https://github.com/modelcontextprotocol
- A2A Protocol: https://a2a-protocol.org / https://github.com/google/A2A

### GitHub Repositories
- LangGraph: https://github.com/langchain-ai/langgraph (~30k stars)
- Google ADK: https://github.com/google/adk-python (~15k stars)
- Semantic Kernel: https://github.com/microsoft/semantic-kernel (~22k stars)
- AutoGen: https://github.com/microsoft/autogen (~35k stars)
- AG2: https://github.com/ag2ai/ag2 (~15k stars)
- CrewAI: https://github.com/crewAIInc/crewAI (~25k stars)
- MCP: https://github.com/modelcontextprotocol/specification
- A2A: https://github.com/google/A2A

### Key Design Documents
- LangGraph Pregel Execution Model: https://langchain-ai.github.io/langgraph/concepts/low_level/
- MCP Specification (Architecture): https://modelcontextprotocol.io/specification
- A2A Specification: https://a2a-protocol.org/specification
- Semantic Kernel Agent Framework: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/
- AutoGen v0.4 Architecture: https://microsoft.github.io/autogen/stable/core-user-guide/
