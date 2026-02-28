# Lab 4: Multi-Tool Agents

**Duration:** 20 minutes  
**Objective:** Build an agent with multiple tools and observe how the LLM decides which tool(s) to call based on the user's question.

---

## What You'll Learn

- How to register multiple tools with a single agent
- How the LLM selects the right tool(s) based on descriptions
- What happens when a question requires multiple tools
- The importance of clear, distinct tool descriptions

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Add a 5th Tool

Add your own tool (e.g., `GetFlightPrice`, `CheckHotelAvailability`, `GetRestaurantRating`) and ask questions that combine it with existing tools.

### Exercise B: Tool Description Quality

1. Change `GetWeather`'s description to just `"A tool."` — does the agent still call it for weather questions?
2. Make two tools with nearly identical descriptions — does the agent get confused?
3. Restore good descriptions and confirm behavior returns to normal.

**Lesson:** Tool descriptions are the LLM's decision guide. Ambiguity in descriptions → ambiguity in behavior.

---

## ✅ Success Criteria

- [ ] Agent correctly selects the right tool for single-tool questions
- [ ] Agent calls multiple tools and combines results for complex questions
- [ ] Agent answers without tools when no tool is needed
- [ ] You understand how tool descriptions drive selection

---

## 📚 Reference

- [Tools overview](https://learn.microsoft.com/en-us/agent-framework/agents/tools/)
- [Function tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/function-tools)
