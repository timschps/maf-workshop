# Lab 1: Hello Agent — Your First AI Agent

**Duration:** 15 minutes
**Objective:** Create a simple agent, invoke it, and stream its response.

---

## What You'll Learn

- How to set up a MAF project from scratch
- The `AIAgent` abstraction and how it wraps an LLM
- The difference between `RunAsync()` (complete response) and `RunStreamingAsync()` (token-by-token)
- How system instructions shape agent behavior

---

## Step 1: Create the Project

Open a terminal and run:

```bash
dotnet new console -n Lab1_HelloAgent
cd Lab1_HelloAgent
```

## Step 2: Add NuGet Packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 3: Set Environment Variables

Make sure these are set in your terminal (or your IDE launch config):

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 4: Write the Code

Replace the contents of `Program.cs` with:

```csharp
using System;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;

// ── Read configuration from environment ──────────────────────────────────────
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// ── Create the agent ─────────────────────────────────────────────────────────
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a friendly assistant. Keep your answers brief.",
        name: "HelloAgent");

// ── Run 1: Non-streaming (complete response at once) ─────────────────────────
Console.WriteLine("=== Non-Streaming ===");
Console.WriteLine(await agent.RunAsync("What is the largest city in France?"));
Console.WriteLine();

// ── Run 2: Streaming (token-by-token) ────────────────────────────────────────
Console.WriteLine("=== Streaming ===");
await foreach (var update in agent.RunStreamingAsync("Tell me a one-sentence fun fact about the Eiffel Tower."))
{
    Console.Write(update);
}
Console.WriteLine();
```

## Step 5: Run It

```bash
dotnet run
```

You should see the agent's response appear — first as a complete block, then token-by-token.

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

### Exercise C (Stretch): Interactive Console Agent

Turn your agent into an interactive chat loop:

```csharp
// Replace the two Run calls with this loop:
Console.WriteLine("Chat with the agent (type 'exit' to quit):");
while (true)
{
    Console.Write("\nYou: ");
    var input = Console.ReadLine();
    if (string.IsNullOrEmpty(input) || input.Equals("exit", StringComparison.OrdinalIgnoreCase))
        break;

    Console.Write("Agent: ");
    await foreach (var update in agent.RunStreamingAsync(input))
    {
        Console.Write(update);
    }
    Console.WriteLine();
}
```

> ⚠️ Note: This loop does NOT maintain conversation history between turns yet —
> each call is independent. We'll fix that in Lab 3!

---

## ✅ Success Criteria

- [ ] Agent responds to a question using non-streaming
- [ ] Agent streams tokens one-by-one
- [ ] You've tried at least 2 different persona instructions
- [ ] You understand the difference between `RunAsync` and `RunStreamingAsync`

---

## 🐍 Python Alternative

<details>
<summary>Click to expand Python version</summary>

```bash
pip install agent-framework --pre azure-identity
```

```python
import asyncio
import os
from azure.identity import AzureCliCredential
from agent_framework.azure import AzureOpenAIResponsesClient

async def main():
    credential = AzureCliCredential()
    client = AzureOpenAIResponsesClient(
        project_endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        deployment_name=os.environ["AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME"],
        credential=credential,
    )

    agent = client.as_agent(
        name="HelloAgent",
        instructions="You are a friendly assistant. Keep your answers brief.",
    )

    # Non-streaming
    print("=== Non-Streaming ===")
    result = await agent.run("What is the largest city in France?")
    print(f"Agent: {result}\n")

    # Streaming
    print("=== Streaming ===")
    print("Agent: ", end="", flush=True)
    async for chunk in agent.run("Tell me a one-sentence fun fact.", stream=True):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print()

asyncio.run(main())
```

</details>

---

## 📚 Reference

- [Official Step 1 docs](https://learn.microsoft.com/en-us/agent-framework/get-started/your-first-agent)
- [Full sample on GitHub](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/01-get-started/01_hello_agent)
