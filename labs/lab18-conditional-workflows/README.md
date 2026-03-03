# Lab 18: Handoff Workflows — Intelligent Agent Routing

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 25 minutes
**Objective:** Build a workflow where a **triage agent** intelligently routes customer queries to the right specialist using **handoff workflows**. You'll create a customer service system where the triage agent analyzes messages and hands off to a billing specialist or tech support agent.

---

## What You'll Learn

- How to build handoff workflows for agent routing
- Defining handoff instructions and routes between agents
- How a triage agent delegates to specialist agents
- Processing agent responses from workflow execution

## When to Use This Pattern

Use **handoff workflows** when routing depends on the content of the user's request:

- **Triage → specialist** — a front-door agent routes to billing, support, or sales based on the question
- **Intent-based routing** — different agent expertise for different intents
- **Escalation paths** — simple questions handled directly, complex ones handed off to specialists

**When alternatives are better:**

| Scenario | Use |
|----------|-----|
| Fixed step order, not content-dependent | **Sequential Workflows** (Lab 11/12) |
| LLM should decide dynamically | **Agent-as-Tool** (Lab 10) — LLM picks the specialist |
| All agents should discuss together | **Group Chat** (Lab 19) |
| Steps are independent and can parallelize | **Concurrent Workflows** (Lab 20) |

## Prerequisites

- Completed Lab 11 (Simple Workflows)
- Azure OpenAI endpoint configured

---

## Architecture

```
                              ┌─ [Billing query]  → BillingSpecialist → response
                              │
Customer Input → TriageAgent ─┤
                              │
                              └─ [Tech issue]     → TechSupport → response
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
| **Handoff Workflow** | An agent routing pattern where a triage agent delegates to specialists |
| **Triage Agent** | The entry point agent that analyzes and routes messages |
| **Handoff Route** | A defined path from one agent to another based on query type |
| **Single-Use Workflows** | Handoff workflows must be rebuilt for each run |

## How Handoff Routing Works

1. The **TriageAgent** receives the customer message
2. Based on its instructions and the handoff definitions, it decides which specialist to route to
3. The workflow **hands off** to the selected specialist agent
4. The specialist processes the query and returns a response
5. Events stream back as the agents work

> **Important:** Handoff workflows are single-use — you must rebuild the workflow for each new conversation.

---

## 🏋️ Exercises

### Exercise A: Test Different Scenarios

Try messages that are clearly billing-related, clearly technical, and some that are ambiguous. Observe how the triage agent routes each one.

### Exercise B: Observe Routing Decisions

Look at which specialist agent handles each query. Does the triage agent always make the right choice?

---

## 🎯 Challenge

Add a third specialist: a "SalesAgent" for product inquiries and pricing questions. Update the triage agent's instructions and add the handoff route!

---

## ✅ Success Criteria

- [ ] Triage agent correctly routes billing queries to BillingSpecialist
- [ ] Triage agent correctly routes tech issues to TechSupport
- [ ] You understand the handoff workflow pattern
- [ ] Workflow is rebuilt for each new conversation

---

## What's Next?

In **Lab 19**, you'll build a **group chat** where multiple agents collaborate in a shared conversation with iterative refinement.
