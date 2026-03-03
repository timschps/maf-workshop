# Lab 16: Agent as MCP Server

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 15 minutes
**Objective:** Expose a MAF agent **as an MCP server** so that any MCP-compatible client (VS Code GitHub Copilot, other agents, Claude Desktop, etc.) can discover and use your agent as a tool.

---

## What You'll Learn

- How to convert an agent into a callable tool
- How to wrap the agent as an MCP-compatible server tool
- How to host the MCP server over stdio transport
- How other MCP clients can discover and call your agent

## When to Use This Pattern

Exposing your agent as an MCP server is the right choice when:

- **You want your agent usable from any MCP client** — VS Code Copilot, Claude Desktop, MCP Inspector, or any other tool that speaks MCP can discover and call your agent without custom integration code.
- **Tool ecosystem interoperability** — Your agent's capabilities become standardized tools that other agents or IDEs can compose alongside tools from other MCP servers (e.g., GitHub, file system, databases).
- **You're building a shared capability** — Instead of each team embedding your agent's logic, you expose it as a service that any MCP-aware consumer can use (e.g., a "company knowledge base agent" or "code review agent").

**When NOT to use it:**

| Instead of MCP Server… | Use… | Why |
|-------------------------|------|-----|
| Consumers are other MAF agents (in-process) | **Agent-as-Tool** (Lab 10) | Simpler, no network overhead |
| Consumers are remote agents | **A2A** (Lab 14/17) | Purpose-built for agent-to-agent collaboration, supports task delegation and streaming |
| It's a simple internal workflow | **Direct function call or Workflow** (Lab 11/12) | Less complexity, no protocol overhead |

> **Rule of thumb:** MCP Server = "make my agent's tools available to the broader tool ecosystem (IDEs, other frameworks)." A2A = "make my agent available to other agents."

## Prerequisites

- Completed Lab 1 (Hello Agent)
- Azure OpenAI endpoint configured

---

## Conceptual Overview

```
  Flip the script: your agent IS the MCP server

  ┌──────────────────────┐          ┌──────────────────────────┐
  │  MCP Client          │          │  Your MCP Server         │
  │                      │          │                          │
  │  VS Code Copilot     │──stdio──▶│  tools/list              │
  │  MCP Inspector       │          │  → ask_joke_agent()      │
  │  Any MCP client      │          │                          │
  │                      │          │  tools/call              │
  │  "Tell me a joke     │──────────▶  → runs your Agent      │
  │   about cats"        │          │    → LLM generates joke  │
  │                      │◀─────────│  → returns result        │
  └──────────────────────┘          └──────────────────────────┘

  Your MAF agent becomes a tool for external MCP clients!
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## How It Works

```
┌─────────────┐     stdio      ┌──────────────────┐     LLM API     ┌─────────────┐
│  MCP Client  │ ──────────────▶│  MCP Server       │ ──────────────▶│ Azure OpenAI │
│  (VS Code,   │ ◀──────────────│  (Your Agent)     │ ◀──────────────│              │
│   Inspector) │                └──────────────────┘                 └─────────────┘
└─────────────┘
```

1. An MCP client connects to your server via stdio
2. The client discovers the agent tool via the MCP protocol
3. When invoked, the tool runs the agent under the hood
4. The agent calls Azure OpenAI and returns the response
5. The response flows back to the MCP client

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Agent as Tool** | Converting an agent into a callable function/tool |
| **MCP Server Tool** | Wrapping the agent function as an MCP-compatible tool |
| **Stdio Transport** | Serves MCP over stdin/stdout (local process) |
| **stderr for logging** | stdout is reserved for MCP protocol; use stderr for logs |

## Testing with MCP Inspector

You can test your MCP server using the MCP Inspector tool:

```bash
npx -y @modelcontextprotocol/inspector <your-server-command>
```

This opens a web UI where you can see the exposed tool and invoke it.

## Configuring in VS Code (GitHub Copilot Agent Mode)

Add your server to VS Code `settings.json` to make it available as a tool in Copilot Chat:

```json
{
  "github.copilot.chat.mcpServers": {
    "joke-agent": {
      "command": "<your-server-command>",
      "args": ["<your-server-args>"]
    }
  }
}
```

---

## 🏋️ Exercises

### Exercise A: Test with MCP Inspector

Use the MCP Inspector to connect to your server, discover the tool, and invoke it with different inputs.

### Exercise B: Configure in VS Code

Add your MCP server to VS Code settings and use it from Copilot Chat in Agent Mode.

---

## 🎯 Challenge

Create a multi-tool MCP server that exposes **two** agents — a JokeAgent and a FactAgent — as separate tools in the same MCP server!

---

## ✅ Success Criteria

- [ ] Agent is exposed as an MCP server tool
- [ ] MCP Inspector can discover and invoke the tool
- [ ] You understand the stdio transport pattern
- [ ] Agent responses flow back through the MCP protocol

---

## What's Next?

In **Lab 17**, you'll use the A2A (Agent-to-Agent) protocol to call remote agents over HTTP — the network equivalent of what MCP does locally.
