# Lab 9: Human-in-the-Loop Tool Approval

[📋 Back to Lab Guide](../../lab-guide.md)


**Duration:** 20 minutes
**Objective:** Build an agent where sensitive tool calls require human approval before execution.

---

## What You'll Learn

- How to use function invocation middleware to intercept sensitive tool calls
- How to prompt for human approval before executing sensitive operations
- How to build an approval flow (approve/reject with details)
- When and why human-in-the-loop matters for production agents

## When to Use This Pattern

Use **human-in-the-loop** when certain agent actions are too risky to execute automatically:

- **Irreversible actions** — deleting data, sending emails, processing payments
- **High-cost operations** — API calls that incur charges, resource provisioning
- **Compliance requirements** — regulated industries requiring human oversight
- **Trust building** — early deployment stages where you want to verify agent behavior

**When full automation is fine:**

| Scenario | Approach |
|----------|----------|
| Read-only operations (search, lookup) | **No approval needed** — safe by nature |
| Internal low-risk tools | **Auto-approve** — approval adds unnecessary friction |
| High-risk but time-critical | **Post-hoc audit** (Lab 13 Observability) instead of blocking |

---

## Conceptual Overview

```
  Agent wants to call a sensitive tool → middleware intercepts:

  ┌──────────┐     ┌───────────┐     ┌──────────────────┐
  │  User     │────▶│  Agent    │────▶│  LLM decides:    │
  │  "Send an │     │           │     │  call send_email  │
  │   email"  │     └───────────┘     └────────┬─────────┘
  └──────────┘                                 │
                                               ▼
                                ┌──────────────────────────┐
                                │  Approval Middleware      │
                                │                          │
                                │  ⚠️  "Agent wants to call │
                                │   send_email(to=bob,     │
                                │   subject=Hello)"        │
                                │                          │
                                │  [Approve] or [Reject]?  │
                                └─────────┬────────────────┘
                                          │
                           ┌──────────────┼──────────────┐
                           ▼                             ▼
                    ✅ Approved                    ❌ Rejected
                    → execute tool                 → skip tool
                    → agent continues              → agent told "denied"
```

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Selective Approval

Make some tools always require approval, and others only in certain conditions. For example: orders under $50 auto-approve, orders over $50 require approval.

### Exercise B: Approval with Details

Enhance the approval prompt to show more details and formatting — display specific argument values (e.g., recipient, subject) before asking for confirmation.

### Exercise C (Stretch): Audit Trail

Log all approval decisions to a list and print a summary at the end showing timestamp, tool name, approval status, and arguments.

---

## ✅ Success Criteria

- [ ] Safe tools (weather) execute without approval
- [ ] Sensitive tools (email, order) pause for human approval
- [ ] Approving a tool call results in execution
- [ ] Rejecting a tool call results in the agent adapting gracefully
- [ ] You understand why human-in-the-loop matters for production agents

---

## 📚 Reference

- [Tool Approval docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/tool-approval)
