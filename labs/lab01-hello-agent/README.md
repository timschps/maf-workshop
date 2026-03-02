# Lab 1: Hello Agent — Your First AI Agent

[📋 Back to Lab Guide](../lab-guide.md)


**Duration:** 15 minutes  
**Objective:** Create a simple agent, invoke it, and stream its response.

---

## What You'll Learn

- How to set up a MAF project from scratch
- The core Agent abstraction and how it wraps an LLM
- The difference between non-streaming (complete response) and streaming (token-by-token)
- How system instructions shape agent behavior

---

## Implementation

Choose your language:

- **[C# (.NET)](./csharp.md)**
- **[Python](./python.md)**

---

## 🏋️ Exercises

### Exercise A: Change the Persona

Modify the `instructions` parameter to create different agent personas. Try each one and ask the same question:

```
"What should I visit in Paris?"
```

| Persona | Instructions |
|---------|-------------|
| Pirate | `"You are a pirate captain. Speak like a pirate. Arrr!"` |
| Poet | `"You are a romantic poet from the 19th century. Answer in verse."` |
| Travel Expert | `"You are a professional travel guide specializing in European cities. Be detailed and practical."` |
| Minimalist | `"Answer in exactly 10 words. No more, no less."` |

**Observe:** How dramatically does behavior change just from instructions?

### Exercise B: Observe Streaming Behavior

1. Ask a long question that requires a lengthy answer (streaming)
2. Ask a short factual question (non-streaming)
3. **Question:** When would you use streaming vs non-streaming in a real application?

---

## ✅ Success Criteria

- [ ] Agent responds to a question using non-streaming
- [ ] Agent streams tokens one-by-one
- [ ] You've tried at least 2 different persona instructions
- [ ] You understand the difference between streaming and non-streaming

---

## 📚 Reference

- [Official Step 1 docs](https://learn.microsoft.com/en-us/agent-framework/get-started/your-first-agent)
- [Full sample on GitHub (C#)](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/01-get-started/01_hello_agent)
- [Full sample on GitHub (Python)](https://github.com/microsoft/agent-framework/tree/main/python/samples/01-get-started/01_hello_agent)
