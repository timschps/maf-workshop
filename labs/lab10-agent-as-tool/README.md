# Lab 10: Agent-as-a-Tool — Multi-Agent Composition

[📋 Back to Lab Guide](../lab-guide.md)


**Duration:** 20 minutes
**Objective:** Build specialist agents and compose them by using one agent as a tool for another.

---

## What You'll Learn

- How to convert an agent into a callable function tool with `AsAIFunction()`
- How a "manager" agent delegates to specialist agents
- The difference between Agent-as-Tool (simple) and Workflows (explicit orchestration)
- When to use each pattern

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
