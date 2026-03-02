# Lab 7: Context Providers & Memory

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 20 minutes  
**Objective:** Build a custom context provider that injects persistent knowledge into every agent run.

---

## What You'll Learn

- The difference between session history (conversation turns) and context providers (injected knowledge)
- How to create a custom context provider that remembers user preferences
- How context providers modify the agent's behavior across sessions

---

## Conceptual Overview

```
  Session history stores conversation;
  Context providers inject external knowledge before every run:

  ┌──────────────────────────────────────────────────────────┐
  │  Agent Run                                               │
  │                                                          │
  │  ┌─────────────────┐   injected before    ┌───────────┐ │
  │  │ Context Provider │──────each run───────▶│           │ │
  │  │                  │                      │   Agent   │ │
  │  │ "Today is Monday │                      │           │ │
  │  │  User prefers    │                      │  System   │ │
  │  │  metric units"   │                      │  prompt + │ │
  │  └─────────────────┘                       │  context  │ │
  │                                            │  + user   │ │
  │  ┌─────────────────┐                       │  message  │ │
  │  │ Another Provider │─────────────────────▶│           │ │
  │  │                  │                      └───────────┘ │
  │  │ "Company policy: │                                    │
  │  │  no discounts    │                                    │
  │  │  over 20%"       │                                    │
  │  └─────────────────┘                                     │
  └──────────────────────────────────────────────────────────┘

  Context providers add dynamic knowledge the LLM doesn't have.
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## ✅ Success Criteria

- [ ] Agent personalizes responses based on injected user context
- [ ] Different users get different code examples (C# vs Python)
- [ ] You understand how context injection differs from conversation history

---

## 📚 Reference

- [Official Step 4: Memory](https://learn.microsoft.com/en-us/agent-framework/get-started/memory)
- [Session docs](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/session)
