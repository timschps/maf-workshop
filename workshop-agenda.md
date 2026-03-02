# Microsoft Agent Framework — Full-Day Workshop

> **Duration:** ~8 hours (9:00 AM – 5:00 PM)
> **Audience:** Developers with cloud/API experience; prior AI/LLM experience helpful but not required
> **Languages:** C# / .NET (primary), Python alternatives noted
> **Prerequisites:** See [prerequisites.md](./prerequisites.md)

---

## Day at a Glance

| Time          | Block                                          | Type        |
|---------------|-------------------------------------------------|-------------|
| 09:00 – 09:30 | Welcome & Intro: Why Agentic AI?               | Theory      |
| 09:30 – 10:15 | Module 1 — Agent Fundamentals (Labs 1–2)       | Theory+Lab  |
| 10:15 – 10:30 | ☕ Break                                        |             |
| 10:30 – 11:15 | Module 2 — Tools & Function Calling (Labs 3–5) | Theory+Lab  |
| 11:15 – 12:00 | Module 3 — Sessions, Context & Middleware (Labs 6–9) | Theory+Lab |
| 12:00 – 12:45 | 🍽️ Lunch                                       |             |
| 12:45 – 13:30 | Module 4 — Workflows, Hosting & Ops (Labs 10–14) | Theory+Lab |
| 13:30 – 13:45 | Hackathon Briefing                             | Briefing    |
| 13:45 – 14:00 | ☕ Break                                        |             |
| 14:00 – 16:30 | Hackathon: Add Agentic Experience to Partner App| Hack        |
| 16:30 – 17:00 | Show & Tell + Wrap-up                          | Presentation|

## Lab Overview (21 Labs)

| #  | Lab                              | Duration | Module | Difficulty |
|----|----------------------------------|----------|--------|------------|
| 1  | Hello Agent                      | 15 min   | 1      | ⭐         |
| 2  | Personas & Prompt Engineering    | 10 min   | 1      | ⭐         |
| 3  | Function Tools                   | 15 min   | 2      | ⭐⭐       |
| 4  | Multi-Tool Agents                | 15 min   | 2      | ⭐⭐       |
| 5  | Structured Output                | 10 min   | 2      | ⭐⭐       |
| 6  | Multi-Turn Conversations         | 15 min   | 3      | ⭐⭐       |
| 7  | Context Providers                | 15 min   | 3      | ⭐⭐       |
| 8  | Middleware Pipeline              | 15 min   | 3      | ⭐⭐⭐     |
| 9  | Human-in-the-Loop                | 15 min   | 3      | ⭐⭐⭐     |
| 10 | Agent-as-a-Tool                  | 15 min   | 4      | ⭐⭐⭐     |
| 11 | Simple Workflows                 | 15 min   | 4      | ⭐⭐⭐     |
| 12 | Agent Workflows                  | 20 min   | 4      | ⭐⭐⭐⭐   |
| 13 | Observability & Telemetry        | 20 min   | 4      | ⭐⭐⭐     |
| 14 | Hosting & A2A Protocol           | 20 min   | 4      | ⭐⭐⭐⭐   |
| 15 | MCP Tools Integration            | 20 min   | 5      | ⭐⭐⭐     |
| 16 | Agent as MCP Server              | 15 min   | 5      | ⭐⭐⭐     |
| 17 | A2A Client — Calling Remote Agents | 20 min | 5      | ⭐⭐⭐⭐   |
| 18 | Handoff Workflows                | 25 min   | 5      | ⭐⭐⭐⭐   |
| 19 | Group Chat Workflows             | 25 min   | 5      | ⭐⭐⭐⭐   |
| 20 | Concurrent Workflows             | 25 min   | 5      | ⭐⭐⭐⭐   |
| 21 | Hosted Multi-Agent Workflow      | 25 min   | 5      | ⭐⭐⭐⭐⭐ |

> **Note:** Not all labs need to be completed before the hackathon. Labs 1–9 cover core concepts; Labs 10–14 are advanced topics; Labs 15–21 are expert-level MCP, A2A, and multi-agent patterns for fast-paced participants.

---

## PART 1 — FRAMEWORK DEEP-DIVE (09:00 – 12:45)

---

### 🟢 Welcome & Intro (09:00 – 09:30)

**Goal:** Set the stage — what is agentic AI and why does it matter now?

**Content:**

1. **What is Agentic AI?**
   - From chatbots → copilots → autonomous agents
   - Key properties: goal-driven, tool-using, multi-step reasoning, memory
   - Real-world use cases: customer service, workflow automation, code generation, enterprise process agents

2. **The Microsoft Agent Framework (MAF) story**
   - Evolution: Semantic Kernel + AutoGen → unified Agent Framework
   - Design principles: enterprise-grade, open standards, multi-provider
   - Where MAF fits in the Microsoft AI stack (Azure AI Foundry, Copilot Studio, M365 Agents SDK)

3. **Architecture overview**
   - Core components: Agent, Tools, Session, Context Providers, Middleware, Workflows
   - Provider model: Azure OpenAI, OpenAI, Anthropic, Ollama, etc.
   - Open protocols: MCP (Model Context Protocol), AG-UI, A2A

```
  MAF Architecture — The Big Picture:

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Your Application                                                    │
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  Agent                                                        │  │
  │  │  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │  │
  │  │  │Instructions│  │  Tools   │  │ Session  │  │ Middleware │  │  │
  │  │  │ (persona)  │  │ (actions)│  │ (memory) │  │ (logging,  │  │  │
  │  │  └────────────┘  └──────────┘  └──────────┘  │  security) │  │  │
  │  │                                               └────────────┘  │  │
  │  └──────────────────────────┬─────────────────────────────────────┘  │
  │                             │                                        │
  │  ┌──────────────────────────┼──────────────────────────────────────┐ │
  │  │  Provider Layer          │                                      │ │
  │  │  ┌──────────┐ ┌─────────┴──┐ ┌──────────┐ ┌──────────┐        │ │
  │  │  │Azure     │ │ OpenAI     │ │Anthropic │ │ Ollama   │  ...   │ │
  │  │  │OpenAI    │ │            │ │          │ │ (local)  │        │ │
  │  │  └──────────┘ └────────────┘ └──────────┘ └──────────┘        │ │
  │  └─────────────────────────────────────────────────────────────────┘ │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────────┐│
  │  │  Integration Protocols                                          ││
  │  │  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐   ││
  │  │  │  MCP    │  │   A2A    │  │  AG-UI   │  │  Workflows     │   ││
  │  │  │ (tools) │  │ (agents) │  │ (UI)     │  │ (orchestration)│   ││
  │  │  └─────────┘  └──────────┘  └──────────┘  └────────────────┘   ││
  │  └──────────────────────────────────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────────────────┘
```

**Delivery:** Slides + architecture diagrams. Interactive poll: "Where are you on the AI agent journey?"

---

### 🟢 Module 1 — Agents Fundamentals (09:30 – 10:15)

**Goal:** Build and run your first agent; understand the core abstractions.

#### Theory (15 min)

- **What is an Agent in MAF?**
  - `AIAgent` — the central abstraction
  - Instructions (system prompt), model provider, tools
  - `RunAsync()` vs `RunStreamingAsync()`
  - Provider flexibility: swap Azure OpenAI ↔ Ollama without changing agent code

- **Creating an Agent — two patterns:**
  1. Quick start: `chatClient.AsAIAgent(instructions, tools)`
  2. Builder pattern for advanced config

- **Agent Providers** — brief overview of supported backends:
  - Azure OpenAI (Chat Completions & Responses API)
  - Azure AI Foundry Agent Service
  - OpenAI, Anthropic, Ollama, GitHub Copilot

#### 🔬 Lab 1: Hello Agent (15 min) → [labs/lab01-hello-agent](./labs/lab01-hello-agent/)

#### 🔬 Lab 2: Personas & Prompt Engineering (10 min) → [labs/lab02-personas-prompts](./labs/lab02-personas-prompts/)

#### Discussion (5 min)
- What did you observe about streaming vs non-streaming?
- How do instructions shape agent behavior?

---

### ☕ Break (10:15 – 10:30)

---

### 🟢 Module 2 — Tools & Function Calling (10:30 – 11:15)

**Goal:** Give agents the ability to *do things* — call functions, use structured output, compose tools.

#### Theory (15 min)

- **Why tools?** Agents are only as useful as the actions they can take
- **Function Tools** — expose C#/Python methods as callable tools
  - `AIFunctionFactory.Create(myMethod)` / `@tool` decorator
  - The LLM decides *when* and *how* to call them based on descriptions
  - Parameter descriptions matter — they guide the LLM

- **Tool types in MAF:**
  | Type | Description |
  |------|-------------|
  | Function Tools | Your own code/methods |
  | Code Interpreter | Sandboxed code execution |
  | File Search | Search uploaded documents |
  | Web Search | Live web search |
  | Hosted MCP Tools | Microsoft-hosted MCP servers (Foundry) |
  | Local MCP Tools | Your own or 3rd-party MCP servers |

```
  How Tool Calling Works:

  ┌──────────┐     ┌────────────────────┐     ┌──────────────────┐
  │  User     │────▶│  Agent (LLM)       │────▶│  Azure OpenAI    │
  │  message  │     │                    │     │                  │
  └──────────┘     │  1. Sees tools      │     │  Decides: "I     │
                   │  2. Sends to LLM    │     │  need to call    │
                   │  3. LLM picks tools │     │  get_weather()"  │
                   │  4. MAF executes    │     └──────────────────┘
                   │  5. Result → LLM    │
                   │  6. Final answer    │───▶ "It's 22°C in Paris"
                   └─────────┬──────────┘
                             │ auto-invoked
                    ┌────────┴────────┐
                    ▼                 ▼
              ┌──────────┐    ┌──────────────┐
              │get_weather│    │get_time      │
              │(your code)│    │(your code)   │
              └──────────┘    └──────────────┘
```

- **Structured Output** — getting typed, parseable responses from agents

#### 🔬 Lab 3: Function Tools (15 min) → [labs/lab03-function-tools](./labs/lab03-function-tools/)

#### 🔬 Lab 4: Multi-Tool Agents (15 min) → [labs/lab04-multi-tool-agents](./labs/lab04-multi-tool-agents/)

#### 🔬 Lab 5: Structured Output (10 min) → [labs/lab05-structured-output](./labs/lab05-structured-output/)

---

### 🟢 Module 3 — Sessions, Context & Middleware (11:15 – 12:00)

**Goal:** Build stateful, multi-turn agents with memory and cross-cutting concerns.

#### Theory (15 min)

- **Sessions & Multi-Turn Conversations**
  - `AgentSession` — the conversation state container
  - `CreateSessionAsync()` → use same session across turns
  - The agent "remembers" prior messages within a session

- **Context Providers**
  - Injecting persistent context into every agent run
  - Use cases: user preferences, knowledge base, business rules

- **Middleware**
  - Intercept agent runs and function calls without modifying core logic
  - Three types: Agent Run, Function Calling, IChatClient
  - Common patterns: logging, security validation, content filtering

```
  Module 3 — The Agent Pipeline:

  ┌─────────────────────────────────────────────────────────────────┐
  │  Context Providers              Middleware Pipeline              │
  │  (injected before run)          (wraps execution)               │
  │                                                                 │
  │  ┌──────────────────┐   ┌─ Logging ──────────────────────────┐ │
  │  │ User Preferences │   │  ┌─ Security ───────────────────┐  │ │
  │  │ Business Rules   │──▶│  │                              │  │ │
  │  │ Current DateTime │   │  │  ┌──────────────────────┐    │  │ │
  │  └──────────────────┘   │  │  │  Agent + Session     │    │  │ │
  │                         │  │  │  (remembers turns)   │    │  │ │
  │  ┌──────────────────┐   │  │  └──────────────────────┘    │  │ │
  │  │ Session (history) │──▶│  └──────────────────────────────┘  │ │
  │  │ Turn 1, 2, 3...  │   └─────────────────────────────────────┘ │
  │  └──────────────────┘                                           │
  └─────────────────────────────────────────────────────────────────┘
```

- **Human-in-the-Loop**
  - Requiring human approval before executing sensitive tools
  - `ApprovalRequiredAIFunction` wrapper

#### 🔬 Lab 6: Multi-Turn Conversations (15 min) → [labs/lab06-multi-turn-conversations](./labs/lab06-multi-turn-conversations/)

#### 🔬 Lab 7: Context Providers (15 min) → [labs/lab07-context-providers](./labs/lab07-context-providers/)

#### 🔬 Lab 8: Middleware Pipeline (15 min) → [labs/lab08-middleware-pipeline](./labs/lab08-middleware-pipeline/)

#### 🔬 Lab 9: Human-in-the-Loop (15 min) → [labs/lab09-human-in-the-loop](./labs/lab09-human-in-the-loop/)

---

### 🍽️ Lunch (12:00 – 12:45)

---

### 🟢 Module 4 — Workflows, Hosting & Operations (12:45 – 13:30)

**Goal:** Understand multi-agent orchestration, hosting, and production concerns.

#### Theory (15 min)

- **Agents vs Workflows — when to use which**
  | Agent | Workflow |
  |-------|---------|
  | Open-ended, conversational | Well-defined steps |
  | Autonomous tool use | Explicit control over execution |
  | Single LLM call (+ tools) | Multiple agents/functions coordinating |

```
  Orchestration Patterns:

  Agent-as-Tool:         Sequential:            Handoff:
  ┌────────────┐        ┌──┐──▶┌──┐──▶┌──┐     ┌─────────┐
  │  Manager   │        │A │   │B │   │C │     │ Triage  │
  │  ├─ Agent1 │        └──┘   └──┘   └──┘     └────┬────┘
  │  ├─ Agent2 │                                ┌────┼────┐
  │  └─ Tool   │        Concurrent:             ▼    ▼    ▼
  └────────────┘        ┌──┐                   ┌──┐ ┌──┐ ┌──┐
                        │A │──┐                │S1│ │S2│ │S3│
                        ├──┤  ├─▶ Merge        └──┘ └──┘ └──┘
                        │B │──┘
                        └──┘    Group Chat:
                                ┌──┐ ◀─▶ ┌──┐
                                │A │     │B │  shared conversation
                                └──┘ ◀─▶ └──┘
```

- **Workflow core concepts:**
  - **Graph-based architecture** — executors + edges
  - **Executors:** Agent executors, Function executors, Nested workflow executors
  - **Type safety** — messages are typed, validated at build time

- **Agent-as-a-Tool** — composing agents as callable tools
  - `innerAgent.AsAIFunction()` — delegation pattern

- **Hosting & Deployment**
  - Hosting an agent via ASP.NET Core
  - A2A (Agent-to-Agent) protocol for cross-system communication
  - AG-UI protocol for rich client integration

- **Observability**
  - OpenTelemetry integration for traces, metrics, logs
  - Token usage tracking, cost estimation

#### 🔬 Lab 10: Agent-as-a-Tool (15 min) → [labs/lab10-agent-as-tool](./labs/lab10-agent-as-tool/)

#### 🔬 Lab 11: Simple Workflows (15 min) → [labs/lab11-simple-workflows](./labs/lab11-simple-workflows/)

#### 🔬 Lab 12: Agent Workflows (20 min) → [labs/lab12-agent-workflows](./labs/lab12-agent-workflows/)

#### 🔬 Lab 13: Observability & Telemetry (20 min) → [labs/lab13-observability](./labs/lab13-observability/)

#### 🔬 Lab 14: Hosting & A2A Protocol (20 min) → [labs/lab14-hosting-a2a](./labs/lab14-hosting-a2a/)

> **Note:** Participants should aim to reach Lab 9 before lunch. Labs 10–14 are for advanced/fast participants. Unfinished labs can be completed during or after the hackathon.

---

### 🟢 Module 5 — MCP, A2A & Multi-Agent Patterns (Advanced)

**Goal:** Master open protocols (MCP, A2A) and advanced multi-agent orchestration patterns.

#### Theory (10 min)

- **Model Context Protocol (MCP)**
  - Open standard for agent ↔ tool integration
  - MCP servers expose tools, resources, and prompts
  - Consuming MCP tools: `McpClient.CreateAsync()` + `ListToolsAsync()`
  - Exposing agents as MCP servers: `McpServerTool.Create(agent.AsAIFunction())`

- **A2A (Agent-to-Agent) Protocol — Beyond Hosting**
  - `A2AAgent` as a client proxy to call remote agents
  - Agent discovery via agent cards
  - Cross-framework interoperability

- **Advanced Workflow Patterns:**
  | Pattern | API | Use Case |
  |---------|-----|----------|
  | Handoff | `CreateHandoffBuilderWith()` | Triage → specialist routing |
  | Concurrent | `BuildConcurrent()` | Parallel agents with merge |
  | Group Chat | `CreateGroupChatBuilderWith()` | Multi-agent discussion |
  | Hosted Workflow | `AddWorkflow().AddAsAIAgent()` | Workflow-as-service |

```
  Open Protocols — How agents connect to the world:

  MCP (Tools):                           A2A (Agents):
  ┌────────┐     ┌────────────────┐     ┌────────┐     ┌────────────────┐
  │ Agent  │────▶│ MCP Server     │     │ Agent  │────▶│ Remote Agent   │
  │        │◀────│ (tool provider)│     │        │◀────│ (A2A server)   │
  └────────┘     └────────────────┘     └────────┘     └────────────────┘
  Discovers tools at runtime            Calls agents over HTTP (JSON-RPC)

  Your Agent AS MCP Server:              Your Agent AS A2A Server:
  ┌────────────────┐     ┌────────┐     ┌────────────────┐     ┌────────┐
  │ VS Code Copilot│────▶│ Your   │     │ Other agents  │────▶│ Your   │
  │ MCP Inspector  │◀────│ Agent  │     │ curl / apps   │◀────│ Agent  │
  └────────────────┘     └────────┘     └────────────────┘     └────────┘
```

#### 🔬 Lab 15: MCP Tools Integration (20 min) → [labs/lab15-mcp-tools](./labs/lab15-mcp-tools/)

#### 🔬 Lab 16: Agent as MCP Server (15 min) → [labs/lab16-agent-as-mcp-server](./labs/lab16-agent-as-mcp-server/)

#### 🔬 Lab 17: A2A Client — Calling Remote Agents (20 min) → [labs/lab17-a2a-client](./labs/lab17-a2a-client/)

#### 🔬 Lab 18: Handoff Workflows (25 min) → [labs/lab18-conditional-workflows](./labs/lab18-conditional-workflows/)

#### 🔬 Lab 19: Group Chat Workflows (25 min) → [labs/lab19-group-chat](./labs/lab19-group-chat/)

#### 🔬 Lab 20: Concurrent Workflows (25 min) → [labs/lab20-parallel-workflows](./labs/lab20-parallel-workflows/)

#### 🔬 Lab 21: Hosted Multi-Agent Workflow (25 min) → [labs/lab21-hosted-multi-agent](./labs/lab21-hosted-multi-agent/)

> **Note:** Labs 15–20 are expert-level and designed for participants who finish the core labs ahead of schedule. They cover the most advanced MAF patterns and are excellent preparation for the hackathon.

---

## PART 2 — HACKATHON (13:30 – 17:00)

---

### 🟠 Hackathon Briefing (13:30 – 13:45)

**Goal:** Introduce the partner application and the hackathon challenge.

```
  Hackathon — Transforming an Existing App:

  BEFORE (Partner App):                  AFTER (With Agentic AI):
  ┌──────────────────────┐              ┌──────────────────────────────┐
  │  Traditional App     │              │  Agentic App                 │
  │  ┌──────────┐        │              │  ┌──────────┐  ┌──────────┐ │
  │  │   UI     │        │              │  │   UI     │  │  Agent   │ │
  │  ├──────────┤        │     ──▶      │  ├──────────┤  │ ┌──────┐ │ │
  │  │  API /   │        │              │  │  API /   │  │ │Tools │ │ │
  │  │ Services │        │              │  │ Services │◀─┤ │(app  │ │ │
  │  ├──────────┤        │              │  ├──────────┤  │ │ APIs)│ │ │
  │  │ Database │        │              │  │ Database │  │ └──────┘ │ │
  │  └──────────┘        │              │  └──────────┘  └──────────┘ │
  └──────────────────────┘              └──────────────────────────────┘
```

**Content:**

1. **Demo the existing partner application**
   - Walk through the current functionality
   - Identify the user journeys / pain points to enhance

2. **The challenge:** Add an agentic experience to the application
   - Define 2–3 specific scenarios where agents add value (examples below)
   - Teams can choose which scenario(s) to tackle

3. **Suggested scenarios** (customize for the partner app):

   | Scenario | Description | MAF Concepts |
   |----------|-------------|--------------|
   | **Conversational Assistant** | Add a chat-based agent that helps users navigate the app and answer questions | Agent, Tools, Session |
   | **Process Automation** | Automate a multi-step workflow (e.g., order processing, approval flow) | Workflows, Human-in-the-Loop |
   | **Smart Search & Insights** | Agent that searches app data and provides intelligent summaries | Tools (Function + File Search), Context Providers |
   | **Multi-Agent Collaboration** | Multiple specialist agents working together on a complex task | Agent-as-Tool, Workflows |

4. **Hackathon rules & logistics:**
   - Teams of 3–5 people
   - Starter code / scaffolding provided (integration points in the partner app)
   - Mentors circulating for help
   - 2.5 hours of hacking time
   - 5-minute demo per team at the end

5. **Judging criteria:**
   - Creativity & usefulness of the agentic experience
   - Correct use of MAF concepts (tools, sessions, workflows)
   - Working demo (doesn't need to be polished!)
   - Team collaboration & presentation

---

### ☕ Break (13:45 – 14:00)

*Teams form, pick scenarios, set up dev environments*

---

### 🟠 Hackathon — Hacking Time (14:00 – 16:30)

**Structure:**

| Time | Activity |
|------|----------|
| 14:00 – 14:15 | Team setup: clone repo, verify environment, review integration points |
| 14:15 – 15:30 | Build phase 1: Core agent + tool integration |
| 15:30 – 15:45 | ☕ Quick break + mid-hack check-in (optional 2-min status per team) |
| 15:45 – 16:15 | Build phase 2: Polish, add workflows, handle edge cases |
| 16:15 – 16:30 | Prepare demo: pick key scenarios to show, rehearse |

**Mentor support topics:**
- Setting up Azure OpenAI connection / credentials
- Wiring tools into existing application services
- Session management and state persistence
- Debugging tool calls and agent behavior
- Workflow graph construction

---

### 🟠 Show & Tell + Wrap-up (16:30 – 17:00)

1. **Team demos** (5 min each)
   - Show the scenario you tackled
   - Demo the working agent
   - Highlight one MAF concept you found most powerful

2. **Voting & prizes** (optional)

3. **Wrap-up & next steps:**
   - Key takeaways from the day
   - Resources for continued learning:
     - [Official docs](https://learn.microsoft.com/en-us/agent-framework/)
     - [GitHub repo](https://github.com/microsoft/agent-framework)
     - [Step-by-step workshop](https://github.com/warnov/ms-agent-framework-step-by-step-workshop)
     - [Agent Framework](https://aka.ms/agentframework)
   - Migration paths from Semantic Kernel / AutoGen
   - Feedback form

---

## Appendix

### A. Key MAF Concepts Cheat Sheet

| Concept | Description |
|---------|-------------|
| `AIAgent` | Core agent abstraction — wraps an LLM + instructions + tools |
| `AgentSession` | Conversation state container for multi-turn interactions |
| Function Tools | C#/Python methods exposed as callable tools for the agent |
| MCP Tools | External tool servers connected via Model Context Protocol |
| MCP Server | Expose your agent as a tool for other MCP-compatible clients |
| Context Providers | Inject persistent context (memory) into agent runs |
| Middleware | Intercept & modify agent runs, function calls, or LLM calls |
| Workflows | Graph-based multi-step orchestration with typed routing |
| Agent-as-Tool | Nest one agent inside another as a callable function |
| Handoff Workflows | Triage agent routes to specialist agents based on query type |
| Concurrent Workflows | Multiple agents process in parallel with result merging |
| Group Chat | Agents collaborate in a shared conversation with iterative refinement |
| AG-UI | Protocol for rich agent ↔ UI communication |
| A2A Server | Expose agents via the Agent-to-Agent protocol (HTTP) |
| `A2AAgent` | Client proxy to call remote agents over A2A protocol |

### B. Useful NuGet Packages / pip Packages

**C# / .NET:**
```
Microsoft.Agents.AI.OpenAI
Azure.AI.OpenAI
Azure.Identity
Microsoft.Agents.AI.Workflows           (workflows, handoffs, concurrent)
Microsoft.Agents.AI.Hosting.A2A.AspNetCore  (A2A hosting)
Microsoft.Agents.AI.A2A                  (A2A client proxy)
ModelContextProtocol                     (MCP server + client)
Microsoft.Agents.AI.Hosting.AGUI.AspNetCore  (AG-UI hosting)
```

**Python:**
```
agent-framework
azure-identity
azure-ai-projects
```

### C. Environment Variables

```bash
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4o-mini
# Or for Foundry:
AZURE_AI_PROJECT_ENDPOINT=https://your-foundry-project.services.ai.azure.com/
```
