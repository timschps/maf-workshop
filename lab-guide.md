# 🧪 Microsoft Agent Framework — Lab Guide

Welcome to the hands-on labs for the **Microsoft Agent Framework (MAF) Workshop**! This guide provides an overview of all 21 labs, organized into five progressive modules. Each lab builds on the concepts from the previous ones, gradually taking you from a simple "Hello Agent" to production-ready multi-agent systems.

---

## How to Use This Guide

### Choose Your Language

Every lab supports **both C# and Python**. Each lab folder contains:

| File | Contents |
|------|----------|
| `README.md` | Shared overview — objectives, concepts, architecture |
| `csharp.md` | Step-by-step C# implementation |
| `python.md` | Step-by-step Python implementation |

Pick your preferred language and follow the corresponding implementation file. You can switch languages between labs if you like — the concepts are the same.

### Before You Begin

Make sure you've completed the [prerequisites](./prerequisites.md):

- **Azure OpenAI** resource with a deployed model (e.g. `gpt-4o-mini`)
- **Azure CLI** installed and authenticated (`az login`)
- **.NET 9+ SDK** (for C#) or **Python 3.11+** (for Python)
- Environment variables set:

  | Variable | C# | Python |
  |----------|-----|--------|
  | Endpoint | `AZURE_OPENAI_ENDPOINT` | `AZURE_OPENAI_ENDPOINT` |
  | Deployment | `AZURE_OPENAI_DEPLOYMENT_NAME` | `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` |

  > ⚠️ Note the different deployment variable names between C# and Python!

### Tips for Success

- **Read the README first** — each lab starts with context and learning objectives before diving into code.
- **Type the code, don't just copy-paste** — you'll learn more by writing it yourself.
- **Experiment!** — Each lab includes exercises and suggestions for extending the code. Try them!
- **Ask questions** — if something doesn't work or doesn't make sense, raise your hand.

---

## Module 1 — Agent Fundamentals

> **Goal:** Understand what an agent is in MAF and how to create, configure, and invoke one.

### [Lab 1: Hello Agent](./labs/lab01-hello-agent/README.md)

Your first agent! Create a simple MAF agent, send it a message, and observe both regular and streaming responses. This is the foundation everything else builds on.

**You'll learn:** Agent creation, `run()` invocation, streaming responses

### [Lab 2: Personas & Prompt Engineering](./labs/lab02-personas-prompts/README.md)

Explore how system instructions (personas) shape agent behavior. Create multiple agents with different personalities and observe how the same question gets very different answers.

**You'll learn:** System instructions, persona design, prompt engineering techniques

---

## Module 2 — Tools & Function Calling

> **Goal:** Give agents the ability to take actions by calling your custom functions and returning structured data.

### [Lab 3: Function Tools](./labs/lab03-function-tools/README.md)

Enable an agent to call a custom function. Watch how the LLM decides *when* to use a tool and how MAF handles the tool-calling loop automatically.

**You'll learn:** `@tool` decorator / `[McpServerTool]`, function descriptions, automatic tool invocation

### [Lab 4: Multi-Tool Agents](./labs/lab04-multi-tool-agents/README.md)

Build an agent equipped with multiple tools and see how it selects the right one(s) based on the user's question. Discover how tool descriptions guide the model's decisions.

**You'll learn:** Multi-tool registration, tool selection behavior, combining tool results

### [Lab 5: Structured Output](./labs/lab05-structured-output/README.md)

Instead of free-form text, get agents to return strongly-typed data structures. Perfect for when your agent output feeds into downstream code.

**You'll learn:** Pydantic models / C# records, `response_format`, typed agent responses

---

## Module 3 — Sessions, Context & Middleware

> **Goal:** Build stateful agents with memory, inject contextual knowledge, and add cross-cutting concerns like logging and security.

### [Lab 6: Multi-Turn Conversations](./labs/lab06-multi-turn-conversations/README.md)

Create agents that remember previous messages. Build a multi-turn conversation where the agent maintains context across exchanges — just like a real chat.

**You'll learn:** Sessions / threads, conversation history, stateful interactions

### [Lab 7: Context Providers & Memory](./labs/lab07-context-providers/README.md)

Inject persistent knowledge into every agent run using context providers. Build a custom provider that adds dynamic information (e.g., current date, user preferences) to the agent's context.

**You'll learn:** `BaseContextProvider` / `IContextProvider`, `extend_instructions()`, dynamic context injection

### [Lab 8: Middleware Pipeline](./labs/lab08-middleware-pipeline/README.md)

Add logging, security checks, and telemetry to your agents *without modifying the agent itself*. Middleware wraps agent execution like layers of an onion.

**You'll learn:** Agent middleware, function middleware, `call_next()` pattern, separation of concerns

### [Lab 9: Human-in-the-Loop](./labs/lab09-human-in-the-loop/README.md)

Build agents where sensitive operations (e.g., sending emails, making purchases) require explicit human approval before execution. Essential for production safety.

**You'll learn:** Approval modes, `approval_mode="always_require"`, interactive approval flow

---

## Module 4 — Workflows, Hosting & Operations

> **Goal:** Compose agents into workflows, add observability, and host agents as web APIs.

### [Lab 10: Agent-as-a-Tool](./labs/lab10-agent-as-tool/README.md)

Use one agent as a callable tool for another — the simplest form of agent composition. A "manager" agent delegates specialist tasks to "worker" agents.

**You'll learn:** `as_tool()`, agent composition, delegation patterns

### [Lab 11: Simple Workflows](./labs/lab11-simple-workflows/README.md)

Build explicit pipelines using the low-level `WorkflowBuilder` API. Define function-based executors connected by edges for full control over execution flow.

**You'll learn:** `WorkflowBuilder`, executors, edges, workflow state, `run_until_convergence()`

### [Lab 12: Agent Workflows (Sequential)](./labs/lab12-agent-workflows/README.md)

Create LLM-powered pipelines where agents collaborate as steps in a sequence. Each agent's output flows as input to the next — like an assembly line.

**You'll learn:** `SequentialBuilder`, agent-to-agent pipelines, `as_agent()` workflow wrapping

### [Lab 13: Observability & Telemetry](./labs/lab13-observability/README.md)

Instrument your agents with OpenTelemetry for production monitoring. See traces, spans, and metrics that show exactly what your agent is doing.

**You'll learn:** OpenTelemetry integration, console exporters, trace visualization

### [Lab 14: Hosting & A2A Protocol](./labs/lab14-hosting-a2a/README.md)

Expose your agent as a web API endpoint using the **Agent-to-Agent (A2A)** protocol. Other agents (or any HTTP client) can discover and call your agent.

**You'll learn:** A2A protocol, agent hosting, agent cards, JSON-RPC messaging

---

## Module 5 — MCP, A2A & Multi-Agent Patterns

> **Goal:** Master advanced integration patterns — MCP for tools, A2A for inter-agent communication, and multi-agent orchestration.

### [Lab 15: MCP Tools Integration](./labs/lab15-mcp-tools/README.md)

Connect your agent to an external **Model Context Protocol (MCP)** server. The agent dynamically discovers and uses tools provided by the MCP server — no hardcoded tool definitions needed.

**You'll learn:** `MCPStreamableHTTPTool`, dynamic tool discovery, MCP protocol

### [Lab 16: Agent as MCP Server](./labs/lab16-agent-as-mcp-server/README.md)

Flip the script — expose your MAF agent *as* an MCP server. Any MCP-compatible client (including VS Code Copilot) can use your agent as a tool.

**You'll learn:** MCP server creation, tool registration, stdio transport, VS Code integration

### [Lab 17: A2A Client](./labs/lab17-a2a-client/README.md)

Build an A2A client that calls a remote agent hosted over HTTP. This is how agents discover and communicate with each other in a distributed system.

**You'll learn:** `A2AAgent` client, remote agent invocation, agent discovery

### [Lab 18: Handoff Workflows](./labs/lab18-conditional-workflows/README.md)

Build intelligent routing where a **triage agent** analyzes the user's request and delegates to the appropriate **specialist agent**. The right expert handles each question.

**You'll learn:** `HandoffBuilder`, triage-based routing, conditional agent selection

### [Lab 19: Group Chat](./labs/lab19-group-chat/README.md)

Create multi-agent conversations where agents collaborate in a shared chat. Use round-robin turn-taking and build custom termination logic to control when the discussion ends.

**You'll learn:** `GroupChatBuilder`, round-robin selection, custom termination, iterative refinement

### [Lab 20: Concurrent Workflows](./labs/lab20-parallel-workflows/README.md)

Execute multiple agents simultaneously and merge their results. When tasks are independent, parallelism dramatically reduces response time.

**You'll learn:** `ConcurrentBuilder`, parallel execution, result aggregation

### [Lab 21: Hosted Multi-Agent Workflow ⭐](./labs/lab21-hosted-multi-agent/README.md)

The **capstone lab**! Combine everything: build a multi-agent sequential workflow (Researcher → Writer → Editor), wrap it as a single agent, and expose it via A2A. This is a production-ready pattern.

**You'll learn:** Workflow composition + hosting, end-to-end multi-agent system, A2A deployment

---

## Quick Reference

### Lab Progression Map

```
Module 1: Fundamentals          Module 2: Tools              Module 3: Context & Middleware
┌─────────────────────┐        ┌──────────────────┐         ┌─────────────────────────┐
│ Lab 1: Hello Agent   │──────▶│ Lab 3: Functions  │───────▶│ Lab 6: Multi-Turn       │
│ Lab 2: Personas      │       │ Lab 4: Multi-Tool │        │ Lab 7: Context Providers│
│                      │       │ Lab 5: Structured │        │ Lab 8: Middleware        │
└─────────────────────┘        └──────────────────┘         │ Lab 9: Human-in-Loop    │
                                                             └─────────────────────────┘
                                                                         │
                    ┌────────────────────────────────────────────────────┘
                    ▼
Module 4: Workflows & Hosting              Module 5: Advanced Patterns
┌──────────────────────────────┐          ┌──────────────────────────────┐
│ Lab 10: Agent-as-Tool        │────────▶│ Lab 15: MCP Tools            │
│ Lab 11: Simple Workflows     │         │ Lab 16: Agent as MCP Server  │
│ Lab 12: Sequential Workflows │         │ Lab 17: A2A Client           │
│ Lab 13: Observability        │         │ Lab 18: Handoff Workflows    │
│ Lab 14: Hosting & A2A        │         │ Lab 19: Group Chat           │
└──────────────────────────────┘         │ Lab 20: Concurrent Workflows │
                                          │ Lab 21: Hosted Multi-Agent ⭐│
                                          └──────────────────────────────┘
```

### Estimated Time per Lab

| Difficulty | Labs | Time |
|------------|------|------|
| 🟢 Beginner | Labs 1–2 | ~15 min each |
| 🟡 Intermediate | Labs 3–9 | ~20 min each |
| 🟠 Advanced | Labs 10–14 | ~25 min each |
| 🔴 Expert | Labs 15–21 | ~30 min each |

### Key Packages

| | C# (.NET) | Python |
|---|-----------|--------|
| **Core** | `Microsoft.Agents.AI.OpenAI` | `agent-framework` |
| **Workflows** | `Microsoft.Agents.AI.Workflows` | `agent-framework-orchestrations` |
| **A2A** | `Microsoft.Agents.A2A.Server` | `agent-framework-a2a` + `a2a-sdk` |
| **MCP** | `ModelContextProtocol` | `mcp` |
| **Auth** | `Azure.Identity` | `azure-identity` |

---

## Troubleshooting

### Common Issues

| Problem | Solution |
|---------|----------|
| `deployment name is required` | Set the correct env var (see table above — C# and Python differ!) |
| `401 Unauthorized` | Run `az login` to refresh your Azure CLI credentials |
| `model not found` | Verify your deployment name matches your Azure OpenAI resource |
| `Connection refused` on A2A labs | Make sure the server is running in a separate terminal |
| Python `ModuleNotFoundError` | Run `pip install agent-framework --pre` (note the `--pre` flag) |
| .NET build errors | Run `dotnet restore` first; ensure .NET 9+ is installed |

### Getting Help

- Check the [prerequisites](./prerequisites.md) — most issues are environment setup
- Review the error message carefully — MAF gives descriptive errors
- Try the other language — sometimes comparing C# and Python helps understanding

---

**Happy building! 🚀**
