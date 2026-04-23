# Lab 17: A2A Client — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Part A: Create the A2A Server

### Step 1: Create the Server Project

```bash
dotnet new web -n Lab17_A2AServer
cd Lab17_A2AServer
```

### Step 2: Add Server Packages

```bash
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
dotnet add package Microsoft.Extensions.AI
dotnet add package Microsoft.Extensions.AI.OpenAI
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
```

### Step 3: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Step 4: Replace Program.cs (Server)

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

### Step 5: Set the Server Port

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

### Step 6: Start the Server

```bash
dotnet run
```

Leave this running and open a **new terminal** for the client.

---

## Part B: Create the A2A Client

### Step 7: Create the Client Project (in a new terminal)

```bash
dotnet new console -n Lab17_A2AClient
cd Lab17_A2AClient
```

### Step 8: Add Client Packages

```bash
dotnet add package Microsoft.Agents.AI.A2A --prerelease
```

### Step 9: Replace Program.cs (Client)

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

### Step 10: Run the Client

```bash
dotnet run
```

You should see the client communicate with the remote Travel Assistant agent and receive travel advice!
