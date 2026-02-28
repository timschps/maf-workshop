# Lab 1: Hello Agent — C# Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

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

### Exercise C (Stretch): Interactive Console Agent

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
> each call is independent. We'll fix that in Lab 6!
