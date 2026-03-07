# Slide Deck Outlines

Below are suggested slide-by-slide outlines for each section. Each slide has a **title**, **key content**, and **speaker notes**.

---

## 00 — Welcome & Intro (09:00 – 09:30)

### Slide 1: Title Slide
- **Title:** Microsoft Agent Framework — From Concepts to Code
- **Subtitle:** Full-Day Hands-On Workshop
- **Content:** Date, Presenter names, Partner logo, Microsoft logo
- **Notes:** Welcome everyone, housekeeping (Wi-Fi, restrooms, breaks schedule)

### Slide 2: Agenda Overview
- **Title:** Today's Journey
- **Content:** Visual timeline of the day — Morning (theory + labs), Afternoon (hackathon)
- **Notes:** Set expectations: by noon you'll have built 4 agents, by 5pm you'll have added agentic AI to a real app

### Slide 3: The AI Agent Evolution
- **Title:** From Chatbots to Autonomous Agents
- **Content:** Visual progression:
  - Chatbot (rule-based, scripted)
  - → Copilot (LLM-powered, human-directed)
  - → Agent (autonomous, goal-driven, tool-using)
- **Notes:** The key shift is autonomy — agents decide WHAT to do, not just HOW to respond

### Slide 4: What Makes an Agent an Agent?
- **Title:** Four Properties of an AI Agent
- **Content:**
  1. 🎯 **Goal-driven** — works toward an objective
  2. 🔧 **Tool-using** — calls functions, APIs, databases
  3. 🧠 **Reasoning** — plans multi-step actions
  4. 💾 **Memory** — maintains context across interactions
- **Notes:** Not all AI apps need agents. If a simple API call suffices, use that instead.

### Slide 5: Real-World Use Cases
- **Title:** Where Agents Shine
- **Content:** 3-4 examples with visuals:
  - Customer support agent (resolves tickets autonomously)
  - Process automation (expense approval workflows)
  - Code assistant (writes, reviews, tests code)
  - Enterprise search (finds and summarizes documents)
- **Notes:** Relate to the partner's industry if possible

### Slide 6: The Microsoft AI Stack
- **Title:** Where Does MAF Fit?
- **Content:** Layered diagram:
  - Top: Copilot Studio (no-code agents)
  - Middle: M365 Agents SDK (Teams/M365 agents)
  - Bottom: **Microsoft Agent Framework** (pro-code, full control)
  - Foundation: Azure AI Foundry, Azure OpenAI
- **Notes:** MAF is for developers who need maximum flexibility and control

### Slide 7: MAF Origins — Semantic Kernel + AutoGen
- **Title:** The Best of Both Worlds
- **Content:** Venn diagram:
  - Semantic Kernel: Enterprise-grade, DI, telemetry, type safety
  - AutoGen: Multi-agent patterns, simple abstractions, research-grade
  - MAF: All of the above + Workflows + MCP + AG-UI
- **Notes:** Same teams, unified vision. SK and AutoGen are predecessors, MAF is the successor.

### Slide 8: Architecture Overview
- **Title:** MAF Architecture at a Glance
- **Content:** Component diagram:
  ```
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │   Agent      │   │   Tools     │   │  Middleware  │
  │  (AIAgent)   │──→│ (Functions, │   │ (Logging,   │
  │              │   │  MCP, etc.) │   │  Security)  │
  └──────┬───────┘   └─────────────┘   └─────────────┘
         │
  ┌──────▼───────┐   ┌─────────────┐   ┌─────────────┐
  │   Session    │   │  Providers  │   │  Workflows  │
  │ (AgentSession│   │ (Azure AOAI,│   │ (Graph-based│
  │  + Context)  │   │  Ollama,..) │   │  orchestr.) │
  └──────────────┘   └─────────────┘   └─────────────┘
  ```
- **Notes:** This diagram will be our compass throughout the day. Keep it visible.

### Slide 9: Open Protocols
- **Title:** Built on Open Standards
- **Content:**
  - **MCP** (Model Context Protocol) — standardized tool integration
  - **AG-UI** — rich agent ↔ UI communication
  - **A2A** — agent-to-agent cross-system interop
- **Notes:** No vendor lock-in. These protocols work across providers.

### Slide 10: Interactive Poll
- **Title:** Where Are You on the AI Agent Journey?
- **Content:** Poll options:
  - 🟢 "I've built chatbots/copilots before"
  - 🟡 "I've experimented with AI APIs (OpenAI, etc.)"
  - 🔴 "This is my first time working with AI agents"
- **Notes:** Use Mentimeter, Slido, or show of hands. Adjust depth accordingly.

---

## 01 — Module 1: Agents Fundamentals (09:30 – 10:15)

### Slide 11: What is an Agent in MAF?
- **Title:** The `AIAgent` Abstraction
- **Content:**
  - An agent = LLM + Instructions + Tools
  - `AIAgent` is the core type
  - Two operations: `RunAsync()` (complete), `RunStreamingAsync()` (token-by-token)
- **Notes:** Emphasize simplicity — you can build an agent in ~10 lines of code

### Slide 12: Creating an Agent — The Quick Way
- **Title:** 10 Lines to Your First Agent
- **Content:** Code slide showing the "quick start" pattern:
  ```csharp
  AIAgent agent = new AzureOpenAIClient(...)
      .GetChatClient("gpt-4o-mini")
      .AsAIAgent(instructions: "...", name: "...");
  ```
- **Notes:** Live-code this. Show it running.

### Slide 13: Provider Flexibility
- **Title:** Swap Models, Keep Your Code
- **Content:** Table of supported providers:
  - Azure OpenAI ✅
  - OpenAI ✅
  - Anthropic ✅
  - Ollama ✅ (local)
  - Azure AI Foundry ✅
  - GitHub Copilot ✅
- **Notes:** The same agent code works with any provider. Demo swapping if time permits.

### Slide 14: Streaming vs Non-Streaming
- **Title:** Two Ways to Get Responses
- **Content:** Side-by-side comparison:
  - `RunAsync` → waits for full response → returns complete text
  - `RunStreamingAsync` → yields tokens as generated → feels faster to users
- **Notes:** Streaming is almost always preferred for user-facing apps

### Slide 15: Lab 1 Instructions
- **Title:** 🔬 Lab 1: Hello Agent
- **Content:** Lab overview + link to lab instructions
- **Notes:** 15 minutes. Circulate and help with Azure credential issues.

---

## 02 — Module 2: Tools & Function Calling (10:30 – 11:15)

### Slide 16: Why Tools?
- **Title:** Agents Without Tools Are Just Chatbots
- **Content:** Visual: Agent with tools can *do things* — query data, call APIs, take actions
- **Notes:** This is the "aha moment" module

### Slide 17: Function Tools in C#
- **Title:** Your Code → Agent's Superpower
- **Content:** Code showing `[Description]` attributes and `AIFunctionFactory.Create()`:
  ```csharp
  [Description("Get the weather for a location.")]
  static string GetWeather(
      [Description("City name")] string location) => ...
  ```
- **Notes:** Stress: descriptions are CRITICAL — they tell the LLM when/how to call the tool

### Slide 18: How Function Calling Works
- **Title:** The LLM Decides — You Don't Hard-Code
- **Content:** Sequence diagram:
  1. User: "What's the weather in Amsterdam?"
  2. LLM sees available tools → decides to call `GetWeather("Amsterdam")`
  3. Framework executes function → returns result to LLM
  4. LLM crafts natural-language response incorporating the result
- **Notes:** The magic: the LLM autonomously decided to call the function

### Slide 19: Tool Types Overview
- **Title:** The MAF Tool Ecosystem
- **Content:** Grid of tool types with icons:
  - Function Tools (your code)
  - Code Interpreter (sandboxed execution)
  - File Search (uploaded docs)
  - Web Search (live web)
  - MCP Tools (hosted + local)
- **Notes:** Function tools are 80% of what you'll use. MCP is the future.

### Slide 20: Agent-as-a-Tool
- **Title:** Agents Calling Agents
- **Content:** Diagram: Main Agent → calls WeatherAgent as a tool → WeatherAgent calls GetWeather function
- **Notes:** This is the simplest multi-agent pattern. Show code briefly.

### Slide 21: MCP — Model Context Protocol
- **Title:** Standardized Tool Integration
- **Content:**
  - MCP = universal adapter for tools
  - Hosted MCP (via Foundry) — Microsoft manages the server
  - Local MCP — you run your own tool server
  - Any MCP server works with any MCP client
- **Notes:** MCP is becoming the industry standard. Brief mention — not a lab topic today.

### Slide 22: Lab 2 Instructions
- **Title:** 🔬 Lab 2: Agent with Tools
- **Content:** Lab overview + link
- **Notes:** 25 minutes. Walk the room during the "bad description" exercise.

---

## 03 — Module 3: Conversations, Memory & Middleware (11:15 – 12:00)

### Slide 23: The Memory Problem
- **Title:** Agents Forget (Unless You Help)
- **Content:**
  - Each `RunAsync()` call is stateless by default
  - `AgentSession` = the solution for multi-turn memory
  - Session stores conversation history + arbitrary state
- **Notes:** Show the contrast: with vs without session

### Slide 24: AgentSession in Action
- **Title:** Sessions Keep the Conversation Alive
- **Content:** Code showing session creation and multi-turn usage
- **Notes:** Live-demo: "My name is Alice" → "What's my name?" with and without session

### Slide 25: Memory & Context Providers
- **Title:** Beyond Chat History — Injecting Knowledge
- **Content:**
  - Context Providers inject persistent info into every run
  - Use cases: user prefs, org knowledge, business rules
  - `ChatHistoryProvider` for custom persistence
  - `InMemoryHistoryProvider` (default) vs custom storage (CosmosDB, etc.)
- **Notes:** Context providers are how you bring domain knowledge to agents

### Slide 26: Middleware — Enterprise Plumbing
- **Title:** Middleware: Intercept Everything
- **Content:** Pipeline diagram:
  ```
  Request → [Security] → [Logging] → [Agent] → [Logging] → Response
                                        ↓
                              [FuncMiddleware] → Tool Call → Result
  ```
- **Notes:** Three types: Agent Run, Function Calling, IChatClient

### Slide 27: Middleware Use Cases
- **Title:** What Can Middleware Do?
- **Content:** Grid:
  - 🔒 Security: block sensitive queries
  - 📊 Telemetry: log latency, token counts
  - 🛡️ Content filtering: check responses for policy violations
  - 🔄 Retry logic: retry failed LLM calls
  - 📝 Audit: log all tool calls for compliance
- **Notes:** Security teams love middleware — it's how you make agents production-safe

### Slide 28: Lab 3 Instructions
- **Title:** 🔬 Lab 3: Multi-Turn Agent with Middleware
- **Content:** Lab overview + link
- **Notes:** 25 minutes. The security middleware exercise is great for discussion.

---

## 04 — Module 4: Workflows (12:45 – 13:30)

### Slide 29: Agents vs Workflows
- **Title:** When a Single Agent Isn't Enough
- **Content:** Decision table:
  | Use an Agent when… | Use a Workflow when… |
  | Open-ended, conversational | Well-defined steps |
  | Autonomous tool use | Explicit control |
  | Single task | Multi-agent coordination |
- **Notes:** "If you can draw a flowchart, use a workflow"

### Slide 30: Workflow Building Blocks
- **Title:** Executors + Edges = Graph
- **Content:** Diagram showing executor types:
  - Function Executor (runs code)
  - Agent Executor (runs an LLM agent)
  - Nested Workflow Executor
  - Connected by Edges (arrows in the graph)
- **Notes:** Type-safe messages flow along edges

### Slide 31: Workflow Patterns
- **Title:** Common Multi-Agent Patterns
- **Content:** Visual diagrams of 4 patterns:
  1. Sequential: A → B → C
  2. Parallel: A → [B, C] → D (fan-out/fan-in)
  3. Conditional: A → if X then B, else C
  4. Supervisor: Manager → delegates to [B, C, D]
- **Notes:** Most real workflows combine these patterns

### Slide 32: Checkpointing & Human-in-the-Loop
- **Title:** Long-Running Workflows Need State
- **Content:**
  - Checkpointing: save/restore workflow state at any point
  - Human-in-the-Loop: pause, get approval, resume
  - Durable execution: survive restarts
- **Notes:** Critical for production workflows that take hours/days

### Slide 33: Hosting Your Agent
- **Title:** From Console to Production
- **Content:** Hosting options table:
  - ASP.NET Core (web API)
  - Azure Functions (serverless)
  - A2A Protocol (agent-to-agent)
  - AG-UI Protocol (rich web UI)
  - OpenAI-compatible endpoints
- **Notes:** Brief overview — full hosting is beyond today's scope but show the 5-line hosting code

### Slide 34: Lab 4 Instructions
- **Title:** 🔬 Lab 4: Build a Workflow
- **Content:** Lab overview + link
- **Notes:** 25 minutes. If short on time, do the function workflow as guided walkthrough and skip the agent workflow to hands-on.

---

## 05 — Hackathon Briefing (13:30 – 13:45)

### Slide 35: Hackathon Time!
- **Title:** 🏗️ Hackathon: Transform the Existing App
- **Content:** Big visual + excitement
- **Notes:** Energy transition — shift from learning to building

### Slide 36: The Existing Application
- **Title:** Meet [App Name]
- **Content:** Screenshots/demo of the existing application
- **Notes:** CUSTOMIZE THIS — walk through the app live, show the user journeys

### Slide 37: The Challenge
- **Title:** Add an Agentic Experience
- **Content:** Scenario cards (customize for existing app):
  1. 💬 Conversational Assistant
  2. ⚙️ Process Automation
  3. 🔍 Smart Search & Insights
  4. 🤖 Multi-Agent Collaboration
- **Notes:** Teams choose 1-2 scenarios. All are valid.

### Slide 38: Rules & Logistics
- **Title:** Hackathon Ground Rules
- **Content:**
  - Teams of 3-5
  - 2.5 hours of hacking
  - Starter code provided
  - Mentors available
  - 5-min demo per team
  - Judging: creativity, MAF concepts, working demo
- **Notes:** Form teams now, pick scenarios, set up environments during break

---

## 06 — Wrap-Up (16:30 – 17:00)

### Slide 39: Demo Time
- **Title:** 🎤 Show & Tell
- **Content:** Timer + team name display
- **Notes:** 5 min per team. Keep it moving.

### Slide 40: Key Takeaways
- **Title:** What We Learned Today
- **Content:**
  1. Agents = LLM + Instructions + Tools + Memory
  2. Tools are the superpower — function calling is where magic happens
  3. Sessions make agents stateful, middleware makes them production-ready
  4. Workflows give explicit control over multi-agent orchestration
  5. MAF is the unified successor to Semantic Kernel + AutoGen
- **Notes:** Reinforce the "so what" — these aren't toys, they're production-grade tools

### Slide 41: Continue Your Journey
- **Title:** Resources & Next Steps
- **Content:**
  - 📚 [Official MAF docs](https://learn.microsoft.com/en-us/agent-framework/)
  - 💻 [GitHub repo](https://github.com/microsoft/agent-framework)
  - 🧪 [Step-by-step workshop](https://github.com/warnov/ms-agent-framework-step-by-step-workshop)
  - 🌐 [Agent Framework](https://aka.ms/agentframework)
  - 🔄 Migration guides: SK → MAF, AutoGen → MAF
- **Notes:** Send follow-up email with all links + today's code

### Slide 42: Thank You + Feedback
- **Title:** Thank You!
- **Content:** QR code to feedback form, presenter contact info
- **Notes:** Encourage honest feedback. Thank the partner.
