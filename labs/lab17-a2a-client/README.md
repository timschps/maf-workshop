# Lab 17: A2A Client — Calling Remote Agents

## Objective

Use the **A2A (Agent-to-Agent) protocol** to call a remote agent hosted over HTTP. You'll first start an A2A server (from Lab 14) and then build a client that discovers and communicates with it using the `A2AAgent` proxy class.

## What You'll Learn

- How to use `A2AAgent` as a proxy to call remote agents
- Agent discovery via A2A agent cards
- How A2A enables cross-framework agent interoperability
- The client-server communication pattern in A2A

## Prerequisites

- Completed Lab 14 (Hosting & A2A Protocol)
- Azure OpenAI endpoint configured

## Duration

20 minutes

## Architecture

```
┌──────────────┐     HTTP/A2A      ┌──────────────────┐     LLM API     ┌─────────────┐
│  A2A Client   │ ────────────────▶│  A2A Server        │ ──────────────▶│ Azure OpenAI │
│  (A2AAgent    │ ◀────────────────│  (Hosted Agent)    │ ◀──────────────│              │
│   proxy)      │   JSON-RPC       └──────────────────┘                 └─────────────┘
└──────────────┘
```

## Steps

### Part A: Create the A2A Server

### Step 1: Create the server project

```bash
dotnet new web -n Lab17_A2AServer
cd Lab17_A2AServer
```

### Step 2: Add server packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
```

### Step 3: Replace Program.cs (Server)

```csharp
using A2A;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Hosting;
using Microsoft.Extensions.AI;
using OpenAI.Chat;

#pragma warning disable MEAI001

var builder = WebApplication.CreateBuilder(args);

string endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
string deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

IChatClient chatClient = new AzureOpenAIClient(
        new Uri(endpoint),
        new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient();
builder.Services.AddSingleton(chatClient);

// Register the agent
var travelAgent = builder.AddAIAgent("travel-assistant",
    instructions: "You are a travel assistant. Provide helpful travel advice, destination recommendations, and trip planning tips. Keep responses concise.");

var app = builder.Build();

app.MapGet("/health", () => "OK");

// Expose via A2A protocol
app.MapA2A(travelAgent, path: "/a2a/travel-assistant", agentCard: new AgentCard()
{
    Name = "Travel Assistant",
    Description = "An AI agent that provides travel advice and trip planning help.",
    Version = "1.0"
});

Console.WriteLine("🌍 Travel Assistant A2A Server running on http://localhost:5100");
app.Run();
```

### Step 4: Set the server port

Add to `Properties/launchSettings.json` (or create it):

```json
{
  "profiles": {
    "Lab17_A2AServer": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5100"
    }
  }
}
```

### Step 5: Start the server

```bash
dotnet run
```

Leave this running and open a **new terminal** for the client.

---

### Part B: Create the A2A Client

### Step 6: Create the client project (in a new terminal)

```bash
dotnet new console -n Lab17_A2AClient
cd Lab17_A2AClient
```

### Step 7: Add client packages

```bash
dotnet add package Microsoft.Agents.AI.A2A --prerelease
```

### Step 8: Replace Program.cs (Client)

```csharp
using System;
using System.Net.Http;
using A2A;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.A2A;

#pragma warning disable MEAI001

string serverUrl = "http://localhost:5100/a2a/travel-assistant";

Console.WriteLine("🌍 A2A Client — Calling Remote Travel Assistant\n");

// --- Step A: Create an A2AAgent proxy ---
using var httpClient = new HttpClient { Timeout = TimeSpan.FromSeconds(60) };
var a2aClient = new A2AClient(new Uri(serverUrl), httpClient);
var a2aAgent = new A2AAgent(a2aClient, "travel-proxy", "Proxy to remote travel agent");

// --- Step B: Call the remote agent ---
Console.WriteLine("📤 Sending: 'What are the top 3 must-visit places in Japan?'\n");
var response = await a2aAgent.RunAsync("What are the top 3 must-visit places in Japan?");
Console.WriteLine($"📥 Response:\n{response}\n");

// --- Step C: Another question ---
Console.WriteLine("📤 Sending: 'What should I pack for a winter trip to Iceland?'\n");
var response2 = await a2aAgent.RunAsync("What should I pack for a winter trip to Iceland?");
Console.WriteLine($"📥 Response:\n{response2}\n");

Console.WriteLine("✅ A2A client communication complete!");
```

### Step 9: Run the client

```bash
dotnet run
```

You should see the client communicate with the remote Travel Assistant agent and receive travel advice!

## Key Concepts

| Concept | Description |
|---------|-------------|
| **A2A Protocol** | Standardized agent-to-agent communication over HTTP using JSON-RPC |
| **A2AAgent** | A proxy class that wraps a remote A2A endpoint as a local `AIAgent` |
| **Agent Card** | Metadata document describing the agent's capabilities (GET `/v1/card`) |
| **Cross-Framework** | A2A enables agents built with different frameworks to communicate |

## Agent Discovery

The A2A protocol supports agent discovery via agent cards. You can fetch the card:

```bash
curl http://localhost:5100/a2a/travel-assistant/v1/card
```

This returns JSON metadata about the agent (name, description, version, capabilities).

## 🎯 Challenge

Extend the client to also act as an agent itself — create a "Trip Planner" agent that uses the remote Travel Assistant as a tool (via `A2AAgent.AsAIFunction()`)!

## What's Next?

In **Lab 18**, you'll build **handoff workflows** — where a triage agent intelligently routes customer queries to the right specialist agent.
