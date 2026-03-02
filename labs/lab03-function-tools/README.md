# Lab 3: Agent with Tools — Function Calling

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 25 minutes  
**Objective:** Give your agent the ability to call custom functions, and observe how the LLM autonomously decides when to use them.

---

## What You'll Learn

- How to define function tools with descriptions
- How the LLM uses tool descriptions to decide when and how to call tools
- How to register multiple tools with an agent
- How to compose agents (Agent-as-a-Tool pattern)

---

## Conceptual Overview

```
  ┌──────────┐      ┌────────────────────────────────────────────┐
  │  User     │      │  Agent                                     │
  │           │      │                                            │
  │ "What's   │─────▶│  1. LLM sees tool descriptions             │
  │  the      │      │  2. Decides: "I need get_weather()"        │
  │  weather  │      │  3. MAF calls your function automatically  │
  │  in       │      │  4. LLM gets result, formulates answer     │
  │  Paris?"  │      │                                            │
  │           │◀─────│  "The weather in Paris is 22°C, sunny"     │
  └──────────┘      └────────────┬───────────────────────────────┘
                                  │
                                  │ Auto-invoked
                                  ▼
                         ┌──────────────────┐
                         │  get_weather()   │
                         │                  │
                         │  Your function!  │
                         │  Returns: "22°C" │
                         └──────────────────┘
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Build Your Own Tool

Create a tool relevant to a business scenario. Ideas:

| Tool | Description |
|------|-------------|
| `LookupCustomer` | Takes a customer ID, returns name + email |
| `GetProductPrice` | Takes a product name, returns the price |
| `SearchKnowledgeBase` | Takes a query, returns relevant FAQ entries |
| `CalculateShipping` | Takes weight + destination, returns cost |

Wire it to the agent and verify the LLM calls it at the right time.

---

## ✅ Success Criteria

- [ ] Agent calls `GetWeather` automatically when asked about weather
- [ ] Agent handles questions that DON'T require tools gracefully
- [ ] You've added a second tool and seen the agent use both
- [ ] You understand why tool descriptions matter

---

## 📚 Reference

- [Official Step 2 docs](https://learn.microsoft.com/en-us/agent-framework/get-started/add-tools)
- [Tools overview](https://learn.microsoft.com/en-us/agent-framework/agents/tools/)
- [Agent-as-a-Tool](https://learn.microsoft.com/en-us/agent-framework/agents/tools/#using-an-agent-as-a-function-tool)
