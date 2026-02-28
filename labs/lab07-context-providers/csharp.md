# Lab 7: Context Providers & Memory — C# Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab7_ContextProviders
cd Lab7_ContextProviders
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 3: Agent with Custom Context Injection

Replace `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// ── Simulate a user profile store ────────────────────────────────────────────
var userProfiles = new Dictionary<string, Dictionary<string, string>>
{
    ["user-alice"] = new()
    {
        ["name"] = "Alice",
        ["role"] = "Senior Developer",
        ["interests"] = "cloud architecture, AI agents, hiking",
        ["preferred_language"] = "C#",
    },
    ["user-bob"] = new()
    {
        ["name"] = "Bob",
        ["role"] = "Product Manager",
        ["interests"] = "roadmap planning, user research",
        ["preferred_language"] = "Python",
    },
};

// ── Create a personalized agent for each user ────────────────────────────────
async Task ChatAsUser(string userId)
{
    var profile = userProfiles[userId];

    // Build personalized instructions that include user context
    var personalizedInstructions = $"""
        You are a helpful assistant. You know the following about the current user:
        - Name: {profile["name"]}
        - Role: {profile["role"]}
        - Interests: {profile["interests"]}
        - Preferred programming language: {profile["preferred_language"]}

        Always address the user by name. Tailor your responses to their role and
        interests. When showing code examples, use their preferred language.
        """;

    AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
        .GetChatClient(deploymentName)
        .AsAIAgent(instructions: personalizedInstructions, name: $"Assistant-{profile["name"]}");

    AgentSession session = await agent.CreateSessionAsync();

    Console.WriteLine($"═══ Chatting as {profile["name"]} ({profile["role"]}) ═══\n");

    // The agent should already know who the user is
    Console.Write("Q: Who am I and what do I do?\nA: ");
    await foreach (var update in agent.RunStreamingAsync("Who am I and what do I do?", session))
        Console.Write(update);
    Console.WriteLine("\n");

    // The agent should use their preferred language
    Console.Write("Q: Show me a quick example of an HTTP request.\nA: ");
    await foreach (var update in agent.RunStreamingAsync("Show me a quick example of an HTTP request.", session))
        Console.Write(update);
    Console.WriteLine("\n");
}

// ── Run for both users — observe personalization ─────────────────────────────
await ChatAsUser("user-alice");
Console.WriteLine(new string('─', 70) + "\n");
await ChatAsUser("user-bob");
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:**
- Alice gets C# code examples; Bob gets Python
- Both agents address users by name without being told
- The "context" is injected at agent creation time via instructions

---

## 🏋️ Exercises

### Exercise A: Dynamic Context Injection

Instead of hardcoding context into instructions, build a helper that combines a base prompt with runtime context:

```csharp
static string BuildInstructions(string baseInstructions, Dictionary<string, string> context)
{
    var contextBlock = string.Join("\n", context.Select(kv => $"- {kv.Key}: {kv.Value}"));
    return $"{baseInstructions}\n\nCurrent user context:\n{contextBlock}";
}

var instructions = BuildInstructions(
    "You are a helpful assistant.",
    userProfiles["user-alice"]
);
```

### Exercise B: Custom Chat History Provider

Create a custom `ChatHistoryProvider` that stores conversations to a file (simulating database persistence):

```csharp
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(new ChatClientAgentOptions()
    {
        ChatOptions = new() { Instructions = "You are a helpful assistant." },
        ChatHistoryProvider = new CustomChatHistoryProvider()
    });
```

### Exercise C (Stretch): Knowledge Base Context

Create a "knowledge base" context provider that injects FAQ entries:

```csharp
var faqs = new[]
{
    "Q: What are your hours? A: We are open Mon-Fri 9am-5pm CET.",
    "Q: How do I reset my password? A: Go to Settings > Security > Reset.",
    "Q: What's the refund policy? A: Full refund within 30 days.",
};

var instructions = $"""
    You are a customer support agent. Answer based on these FAQs:

    {string.Join("\n", faqs)}

    If the answer is not in the FAQs, say "I'll escalate this to a human agent."
    """;
```
