# Lab 15: MCP Tools Integration

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 20 minutes
**Objective:** Connect your agent to an external **Model Context Protocol (MCP)** server and use its tools. MCP is an open standard that lets agents discover and call tools exposed by any MCP-compatible server — giving your agent access to external data sources, APIs, and services without writing custom tool code.

---

## What You'll Learn

- How to connect to an MCP server and retrieve its tools
- How agents use MCP tools via function calling (just like local tools)
- The difference between stdio and HTTP-based MCP transports
- How to pass discovered MCP tools to an agent

## Prerequisites

- Completed Lab 1 (Hello Agent)
- Node.js installed (for running the MCP test server)
- Azure OpenAI endpoint configured

---

## Conceptual Overview

```
  MCP (Model Context Protocol) — dynamic tool discovery:

  ┌──────────────┐          ┌─────────────────────────────┐
  │  Agent       │          │  MCP Server (remote)        │
  │              │          │                             │
  │  "What tools │──GET────▶│  tools/list                 │
  │   do you     │          │  → get_products()           │
  │   have?"     │◀─────────│  → search_inventory()       │
  │              │          │  → check_price()            │
  │              │          └─────────────────────────────┘
  │  Now uses    │                       │
  │  them like   │──POST tools/call─────▶│
  │  local tools │◀── result ────────────│
  └──────────────┘

  The agent doesn't know the tools at compile time —
  it discovers them from the MCP server at runtime!
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **MCP Server** | A service that exposes tools, resources, and prompts via the Model Context Protocol |
| **MCP Client** | Connects to an MCP server to discover and invoke its tools |
| **Stdio Transport** | Runs an MCP server as a local subprocess, communicating via stdin/stdout |
| **HTTP Transport** | Connects to a remote MCP server over HTTP |
| **Tool Discovery** | Retrieves all available tools from the server at runtime |

## Connecting to Remote HTTP MCP Servers

For remote MCP servers (like `https://staybright-demo-app.azurewebsites.net/mcp`), use the HTTP transport instead of stdio. See the language-specific implementation guides for details.

> **Note:** Remote MCP servers may require authentication headers.

## Popular MCP Servers to Try

| Server | Install | Description |
|--------|---------|-------------|
| `@modelcontextprotocol/server-everything` | `npx -y @modelcontextprotocol/server-everything` | Test server with echo, add, and sample tools |
| `@modelcontextprotocol/server-github` | `npx -y @modelcontextprotocol/server-github` | GitHub API tools (requires `GITHUB_PERSONAL_ACCESS_TOKEN`) |
| `@modelcontextprotocol/server-filesystem` | `npx -y @modelcontextprotocol/server-filesystem /path` | File system access tools |
| Azure MCP Server | [github.com/Azure/azure-mcp](https://github.com/Azure/azure-mcp) | Azure services (Storage, Cosmos DB, etc.) |

---

## 🏋️ Exercises

### Exercise A: Explore MCP Tool Discovery

Run the application and observe the tools discovered from the MCP server. Note the tool names, descriptions, and how they map to function calls.

### Exercise B: Try a Remote MCP Server

Switch from the local stdio transport to an HTTP-based remote MCP server. Compare the two approaches.

---

## 🎯 Challenge

Try connecting to the GitHub MCP server and asking the agent to summarize recent commits from a repository you like!

---

## ✅ Success Criteria

- [ ] Agent connects to an MCP server and discovers available tools
- [ ] Agent successfully uses MCP tools to answer questions
- [ ] You understand the difference between stdio and HTTP transports
- [ ] You've explored the list of available MCP tools

---

## 📚 References

- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [MCP Server Registry](https://github.com/modelcontextprotocol/servers)

---

## What's Next?

In **Lab 16**, you'll flip the script — exposing your own agent *as* an MCP server that other tools (like VS Code Copilot) can discover and use.
