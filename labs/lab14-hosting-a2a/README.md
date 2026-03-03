# Lab 14: Hosting & A2A Protocol

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 20 minutes
**Objective:** Expose an agent as a web API endpoint using ASP.NET Core and the Agent-to-Agent (A2A) protocol.

---

## What You'll Learn

- How to host an agent inside an ASP.NET Core application
- How the A2A protocol enables interoperability between agents
- How to test your hosted agent with HTTP calls
- How AG-UI can expose agents to web frontends

## When to Use This Pattern

Use **A2A hosting** when your agent needs to be callable over the network:

- **Cross-team collaboration** — other teams call your agent without sharing code
- **Cross-language interop** — a Python agent calling a C# agent (or vice versa)
- **Microservice architecture** — agents as independently deployable services
- **Third-party access** — external partners can use your agent via HTTP

**Comparison with alternatives:**

| Pattern | Transport | Best for |
|---------|-----------|----------|
| **A2A Hosting** (this lab) | HTTP (JSON-RPC) | Agent-to-agent, cross-network |
| **MCP Server** (Lab 16) | stdio or HTTP | Exposing tools to IDEs and tool consumers |
| **Agent-as-Tool** (Lab 10) | In-process | Same codebase, no network overhead |

---

## Conceptual Overview

```
  Agent exposed as an HTTP endpoint via the A2A protocol:

  ┌──────────────┐                    ┌─────────────────────────────┐
  │  Any HTTP     │   A2A Protocol    │  Your Application           │
  │  Client       │   (JSON-RPC)      │                             │
  │               │                   │  ┌───────────────────────┐  │
  │  curl         │──── POST / ──────▶│  │  Agent                │  │
  │  another agent│                   │  │  (with tools,         │  │
  │  web frontend │◀── Response ──────│  │   instructions, etc.) │  │
  │               │                   │  └───────────────────────┘  │
  └──────────────┘                    │                             │
                                      │  GET /.well-known/          │
  ┌──────────────┐                    │      agent.json             │
  │  Discovery    │◀──────────────────│  → Agent Card (name,        │
  │               │                   │    skills, capabilities)    │
  └──────────────┘                    └─────────────────────────────┘
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Add More Tools

Add a `GetFlightInfo` tool that returns mock flight information. Test it via the A2A endpoint.

### Exercise B: Multiple Agents on One Server

Register a second agent (e.g., `RestaurantAdvisor`) and map it to a separate endpoint. Test both agents independently.

### Exercise C (Stretch): Agent-to-Agent Communication

Create a separate client application that acts as an A2A **client**, sending requests to your hosted agent. This demonstrates the core A2A pattern: agents communicating over HTTP using a standard protocol.

---

## ✅ Success Criteria

- [ ] Agent runs as an ASP.NET Core web API
- [ ] Health check endpoint responds correctly
- [ ] A2A endpoint accepts JSON-RPC requests and returns agent responses
- [ ] You understand how A2A enables agent-to-agent interoperability
- [ ] The agent uses tools (weather, time) when handling requests

---

## 📚 Reference

- [A2A hosting overview](https://learn.microsoft.com/en-us/agent-framework/concepts/host-agents)
- [A2A protocol spec](https://a2aprotocol.ai/)
- [AG-UI protocol](https://docs.ag-ui.com/concepts/overview)
