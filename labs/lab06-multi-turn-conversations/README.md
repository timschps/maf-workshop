# Lab 6: Multi-Turn Conversations & Sessions

**Duration:** 20 minutes
**Objective:** Build a stateful agent that maintains conversation context across multiple turns using sessions.

---

## What You'll Learn

- How `AgentSession` maintains conversation context across turns
- How the agent "remembers" what was said earlier in the conversation
- The difference between stateful (with session) and stateless (without session) calls
- How to serialize and restore sessions for persistence

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab6_MultiTurn
cd Lab6_MultiTurn
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Write a Multi-Turn Agent

Replace `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

// ── Configuration ────────────────────────────────────────────────────────────
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// ── Tool from Lab 2 ─────────────────────────────────────────────────────────
[Description("Get the current weather for a given location.")]
static string GetWeather(
    [Description("The city name.")] string location)
{
    var conditions = new[] { "sunny", "cloudy", "rainy", "partly cloudy" };
    return $"Weather in {location}: {conditions[Random.Shared.Next(conditions.Length)]}, {Random.Shared.Next(5, 30)}°C.";
}

// ── Create the agent ─────────────────────────────────────────────────────────
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a friendly travel assistant. You can check weather for cities. Remember the user's name and preferences throughout the conversation.",
        name: "TravelAssistant",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// ── Create a session for multi-turn conversation ─────────────────────────────
AgentSession session = await agent.CreateSessionAsync();

// ── Turn 1: Introduce yourself ───────────────────────────────────────────────
Console.WriteLine("--- Turn 1 ---");
Console.WriteLine(await agent.RunAsync("Hi! My name is Alice and I love warm weather destinations.", session));
Console.WriteLine();

// ── Turn 2: Ask for a recommendation (agent should remember the preference) ──
Console.WriteLine("--- Turn 2 ---");
Console.WriteLine(await agent.RunAsync("Can you check the weather in Barcelona for me?", session));
Console.WriteLine();

// ── Turn 3: Test memory — does the agent remember the name? ──────────────────
Console.WriteLine("--- Turn 3 ---");
Console.WriteLine(await agent.RunAsync("Based on my preferences, would you recommend Barcelona for me? Also, what's my name?", session));
Console.WriteLine();

// ── Turn 4: Without a session — agent should NOT remember ────────────────────
Console.WriteLine("--- Turn 4 (NO SESSION — new context!) ---");
Console.WriteLine(await agent.RunAsync("What's my name?"));
Console.WriteLine();
```

### Step 3: Run and Observe

```bash
dotnet run
```

**Key observations:**
- Turns 1–3 use the same `session` — the agent remembers Alice's name and preference
- Turn 4 has NO session — the agent doesn't know the user's name
- The weather tool is called automatically in Turn 2

---

## 🏋️ Exercises

### Exercise A: Interactive Multi-Turn Chat

Build an interactive console loop with session:

```csharp
AgentSession chatSession = await agent.CreateSessionAsync();
Console.WriteLine("Chat with the travel assistant (type 'exit' to quit):\n");
while (true)
{
    Console.Write("You: ");
    var input = Console.ReadLine();
    if (string.IsNullOrEmpty(input) || input.Equals("exit", StringComparison.OrdinalIgnoreCase))
        break;

    Console.Write("Agent: ");
    await foreach (var update in agent.RunStreamingAsync(input, chatSession))
        Console.Write(update);
    Console.WriteLine("\n");
}
```

Have a multi-turn conversation: introduce yourself, ask about weather, then test if it remembers your name.

### Exercise B: Session Serialization

Serialize the session, then restore it and continue the conversation:

```csharp
// After a few turns...
var serialized = agent.SerializeSession(session);
Console.WriteLine($"Serialized session: {serialized.Length} chars");

// Restore and continue
AgentSession restored = await agent.DeserializeSessionAsync(serialized);
Console.WriteLine(await agent.RunAsync("Do you still remember my name?", restored));
```

### Exercise C (Stretch): Parallel Sessions

Create two separate sessions and show they're independent:

```csharp
var session1 = await agent.CreateSessionAsync();
var session2 = await agent.CreateSessionAsync();

await agent.RunAsync("My name is Alice.", session1);
await agent.RunAsync("My name is Bob.", session2);

Console.WriteLine(await agent.RunAsync("What's my name?", session1)); // Alice
Console.WriteLine(await agent.RunAsync("What's my name?", session2)); // Bob
```

---

## ✅ Success Criteria

- [ ] Agent maintains conversation context across multiple turns using a session
- [ ] Agent loses context when no session is provided
- [ ] You've had a multi-turn conversation where the agent references earlier context
- [ ] You understand that sessions are isolated — different sessions = different conversations

---

## 📚 Reference

- [Official Step 3: Multi-Turn](https://learn.microsoft.com/en-us/agent-framework/get-started/multi-turn)
- [Official Step 4: Memory](https://learn.microsoft.com/en-us/agent-framework/get-started/memory)
- [Session docs](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/session)
