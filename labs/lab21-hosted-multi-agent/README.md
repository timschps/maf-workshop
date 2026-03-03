# Lab 21: Hosted Multi-Agent Workflow

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 25 minutes
**Objective:** Build a **hosted multi-agent system** that orchestrates multiple agents in a workflow and exposes the entire pipeline as a single A2A endpoint. This is the capstone lab — combining hosting, workflows, and A2A.

---

## What You'll Learn

- How to register multiple agents and orchestrate them in a workflow
- Converting a workflow into a single agent endpoint
- Exposing a multi-agent workflow via the A2A protocol
- The complete end-to-end pattern for hosted multi-agent systems

## When to Use This Pattern

Use **hosted multi-agent workflows** when you need a production-ready multi-agent service:

- **Production deployment** — expose a complete agent system as a single API endpoint
- **Encapsulation** — consumers see one agent, but internally it's a coordinated workflow
- **Scalability** — deploy as a microservice with independent scaling
- **Combines everything** — hosting (Lab 14) + workflows (Lab 11/12) + agent composition (Lab 10)

> **This is the "graduation" pattern** — it combines all the concepts from the workshop into a deployable service. Use it when you're ready to ship.

## Prerequisites

- Completed Labs 11-12 (Workflows) and Lab 14 (Hosting & A2A)
- Azure OpenAI endpoint configured

---

## Architecture

```
                          ┌─────────────────────────────────────┐
                          │  Host                                │
                          │                                      │
  HTTP Request  ────────▶ │  /a2a/content-pipeline               │
  (A2A Protocol)          │    │                                 │
                          │    ▼                                 │
                          │  [Researcher] → [Writer] → [Editor]  │
                          │                                      │
  HTTP Response ◀──────── │  Final edited article                │
                          └─────────────────────────────────────┘
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
| **Agent Registration** | Registering multiple agents in the application |
| **Sequential Workflow** | Chaining agents in order: Researcher → Writer → Editor |
| **Workflow as Agent** | Converting a workflow into a single agent for protocol integration |
| **A2A Exposure** | Making the workflow accessible over HTTP via the A2A protocol |

## How It Works

1. **Registration**: Three agents and one workflow are registered
2. **Workflow Construction**: Agents are chained sequentially
3. **Protocol Adapter**: The workflow is wrapped as a standalone agent
4. **A2A Exposure**: The agent is made accessible over HTTP via A2A
5. **Execution**: When a request arrives, the topic flows through Researcher → Writer → Editor
6. **Response**: The final edited article is returned as the A2A response

## The Full Journey

This lab combines concepts from throughout the workshop:

| Lab | Concept | Used Here |
|-----|---------|-----------|
| Lab 1 | Creating agents | Three specialist agents |
| Lab 2 | Personas | Each agent has a distinct role |
| Lab 11 | Sequential workflows | Pipeline chaining |
| Lab 14 | A2A hosting | A2A endpoint |

---

## 🏋️ Exercises

### Exercise A: Test the Pipeline

Send different topics to the content pipeline and observe how each agent contributes to the final article.

### Exercise B: Fetch the Agent Card

Use `curl` to fetch the agent card and examine the metadata exposed by the A2A endpoint.

---

## 🎯 Challenge

1. Add a **Translator** agent as a 4th step that translates the article to another language
2. Expose individual agents on their own A2A endpoints alongside the pipeline
3. Build a client (like Lab 17) that calls the content pipeline

---

## ✅ Success Criteria

- [ ] Multi-agent workflow processes a topic through all three agents
- [ ] The pipeline is exposed as an A2A endpoint
- [ ] Requests produce polished articles that went through research, writing, and editing
- [ ] You can fetch the agent card via HTTP

---

## Congratulations! 🎉

You've completed all 21 labs! You now have hands-on experience with:

- ✅ Agent fundamentals & prompt engineering
- ✅ Function tools & structured output
- ✅ Multi-turn conversations & context
- ✅ Middleware & human-in-the-loop
- ✅ Multi-agent composition (agent-as-tool)
- ✅ Handoff routing in workflows
- ✅ Group chat collaboration patterns
- ✅ Concurrent & parallel workflows
- ✅ MCP tools (consuming & exposing)
- ✅ A2A protocol (server & client)
- ✅ Hosted multi-agent systems

**Ready for the hackathon!** 🚀
