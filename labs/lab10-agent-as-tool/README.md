# Lab 10: Agent-as-a-Tool — Multi-Agent Composition

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 20 minutes
**Objective:** Build specialist agents and compose them by using one agent as a tool for another.

---

## What You'll Learn

- How to convert an agent into a callable function tool with `AsAIFunction()`
- How a "manager" agent delegates to specialist agents
- The difference between Agent-as-Tool (simple) and Workflows (explicit orchestration)
- When to use each pattern

## When to Use This Pattern

Use **Agent-as-Tool** when the LLM should dynamically decide which specialist to call:

- **Manager → specialist pattern** — an orchestrator delegates to experts (data analyst, recommender, writer)
- **Optional delegation** — the manager can answer directly or escalate to a specialist
- **Simple composition** — you want multi-agent without the complexity of workflows

**Comparison with alternatives:**

| Pattern | Who decides routing? | Process boundary | Best for |
|---------|---------------------|-----------------|----------|
| **Agent-as-Tool** (this lab) | LLM decides | In-process | Dynamic delegation, optional specialists |
| **Workflows** (Lab 11/12) | You decide (code) | In-process | Deterministic pipelines, fixed steps |
| **A2A** (Lab 14/17) | Your code | Cross-network | Remote agents, cross-language |
| **Group Chat** (Lab 19) | Turn-taking | In-process | Collaborative discussion |

---

## Conceptual Overview

```
  Agent-as-Tool: one agent uses another as a callable function

  ┌──────────────────────────────────────────────────────────────┐
  │  Manager Agent                                               │
  │                                                              │
  │  "What's the weather in Tokyo and tell me a joke about it"   │
  │                                                              │
  │  LLM sees tools:                                             │
  │   ├── get_weather()      ← regular function tool             │
  │   ├── ask_joke_agent()   ← agent wrapped as tool             │
  │   └── get_time()         ← regular function tool             │
  │                                                              │
  │  Decides to call:                                            │
  │   1. get_weather("Tokyo") → "22°C, sunny"                   │
  │   2. ask_joke_agent("joke about sunny Tokyo")               │
  │         │                                                    │
  └─────────┼────────────────────────────────────────────────────┘
            │
            ▼
  ┌───────────────────┐
  │  Joke Agent       │  ← separate agent with own instructions
  │  "You are a       │     runs independently, returns result
  │   comedian..."    │
  └───────────────────┘
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Add a 4th Specialist

Create a `TranslationAgent` with instructions to translate phrases, and add it as a tool to the manager. Test with: "How do I say 'hello' in Japanese and what's the weather there?"

### Exercise B: Specialist Feedback Loop

Ask the manager a question where one specialist's answer informs another:
- "Is it a good time to visit Tokyo? Check the weather and the time difference from CET."

### Exercise C (Stretch): Named Tool Customization

Customize the tool name and description when converting an agent to a tool. Override how the specialist appears to the manager agent.

---

## ✅ Success Criteria

- [ ] Manager agent delegates to the correct specialist for simple questions
- [ ] Manager coordinates multiple specialists for complex questions
- [ ] Each specialist maintains its own tools and instructions
- [ ] You understand the delegation pattern: Manager → Specialist → Tools

---

## 📚 Reference

- [Agent-as-Tool docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/#using-an-agent-as-a-function-tool)
