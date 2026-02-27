# Lab 20: Hosted Multi-Agent Workflow

## Objective

Build a **hosted multi-agent system** using the MAF hosting library. Multiple agents are registered via dependency injection, orchestrated in a workflow, and the entire workflow is exposed as a single A2A endpoint. This is the capstone lab — combining hosting, workflows, and A2A.

## What You'll Learn

- How to use `AddAIAgent` to register multiple agents in DI
- How to use `AddWorkflow` to create agent pipelines from DI
- Converting a workflow to an agent with `AddAsAIAgent()`
- Exposing a workflow as an A2A endpoint

## Prerequisites

- Completed Labs 11-12 (Workflows) and Lab 14 (Hosting & A2A)
- Azure OpenAI endpoint configured

## Duration

25 minutes

## Architecture

```
                          ┌─────────────────────────────────────┐
                          │  ASP.NET Core Host                   │
                          │                                      │
  HTTP Request  ────────▶ │  /a2a/content-pipeline               │
  (A2A Protocol)          │    │                                 │
                          │    ▼                                 │
                          │  [Researcher] → [Writer] → [Editor]  │
                          │                                      │
  HTTP Response ◀──────── │  Final edited article                │
                          └─────────────────────────────────────┘
```

## Steps

### Step 1: Create a new web application

```bash
dotnet new web -n Lab20_HostedMultiAgent
cd Lab20_HostedMultiAgent
```

### Step 2: Add NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.AI.OpenAI --prerelease
```

### Step 3: Replace Program.cs

Replace the contents of `Program.cs` with:

```csharp
using A2A;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Hosting;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.DependencyInjection;
using OpenAI.Chat;

#pragma warning disable MEAI001

var builder = WebApplication.CreateBuilder(args);

string endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
string deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// --- Register the shared chat client ---
IChatClient chatClient = new AzureOpenAIClient(
        new Uri(endpoint),
        new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient();
builder.Services.AddSingleton(chatClient);

// --- Register individual agents ---
builder.AddAIAgent("researcher",
    instructions: "You are a research specialist. When given a topic, provide 3-5 key facts and data points. Be factual and concise. Output only the research findings.");

builder.AddAIAgent("writer",
    instructions: "You are a professional writer. Take the research provided and write a short, engaging article (150-200 words). Use a clear structure with an introduction, body, and conclusion.");

builder.AddAIAgent("editor",
    instructions: "You are an editor. Review the article provided and improve it: fix grammar, improve clarity, add a catchy title. Output the final polished article with the title on the first line.");

// --- Register the workflow ---
var contentPipeline = builder.AddWorkflow("content-pipeline", (sp, key) =>
{
    var researcher = sp.GetRequiredKeyedService<AIAgent>("researcher");
    var writer = sp.GetRequiredKeyedService<AIAgent>("writer");
    var editor = sp.GetRequiredKeyedService<AIAgent>("editor");
    return AgentWorkflowBuilder.BuildSequential(key, [researcher, writer, editor]);
}).AddAsAIAgent();

// --- Expose via A2A ---
var app = builder.Build();

app.MapGet("/health", () => "OK");

app.MapA2A(contentPipeline, path: "/a2a/content-pipeline", agentCard: new AgentCard()
{
    Name = "Content Pipeline",
    Description = "A multi-agent content creation pipeline: Research → Write → Edit. Submit any topic and receive a polished article.",
    Version = "1.0"
});

Console.WriteLine("📝 Content Pipeline Multi-Agent System running on http://localhost:5200");
Console.WriteLine("   POST /a2a/content-pipeline — Submit a topic for article generation");
Console.WriteLine("   GET  /a2a/content-pipeline/v1/card — Agent card (discovery)");
Console.WriteLine("   GET  /health — Health check");
app.Run();
```

### Step 4: Set the server port

Add to `Properties/launchSettings.json` (or create it):

```json
{
  "profiles": {
    "Lab20_HostedMultiAgent": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5200"
    }
  }
}
```

### Step 5: Run the application

```bash
dotnet run
```

### Step 6: Test with curl

In a new terminal, send a request to the A2A endpoint:

```bash
curl -X POST http://localhost:5200/a2a/content-pipeline \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "message/send",
    "params": {
      "message": {
        "kind": "message",
        "messageId": "msg-1",
        "role": "user",
        "parts": [{"kind": "text", "text": "Write an article about quantum computing"}]
      },
      "contextId": "test-1"
    }
  }'
```

You should receive a polished article that went through all three agents!

## Key Concepts

| Concept | Description |
|---------|-------------|
| `builder.AddAIAgent()` | Registers an agent in the DI container as a keyed service |
| `builder.AddWorkflow()` | Registers a workflow that resolves agents from DI |
| `.AddAsAIAgent()` | Converts a workflow into an `AIAgent` for protocol integration |
| `app.MapA2A()` | Exposes the workflow-as-agent via the A2A protocol |
| `AgentWorkflowBuilder.BuildSequential()` | Creates a sequential pipeline from multiple agents |
| `GetRequiredKeyedService<AIAgent>()` | Resolves a specific named agent from the DI container |

## How It Works

1. **Registration**: Three agents and one workflow are registered in the DI container
2. **Workflow Construction**: The workflow resolves agents from DI and chains them sequentially
3. **Protocol Adapter**: `AddAsAIAgent()` wraps the workflow as a standalone agent
4. **A2A Exposure**: `MapA2A()` makes the workflow accessible over HTTP via A2A
5. **Execution**: When a request arrives, the topic flows through Researcher → Writer → Editor
6. **Response**: The final edited article is returned as the A2A response

## The Full Journey

This lab combines concepts from throughout the workshop:

| Lab | Concept | Used Here |
|-----|---------|-----------|
| Lab 1 | Creating agents | Three specialist agents |
| Lab 2 | Personas | Each agent has a distinct role |
| Lab 11 | Sequential workflows | `BuildSequential()` pipeline |
| Lab 14 | A2A hosting | `MapA2AServer()` endpoint |

## 🎯 Challenge

1. Add a **Translator** agent as a 4th step that translates the article to another language
2. Expose individual agents on their own A2A endpoints alongside the pipeline
3. Build a client (like Lab 17) that calls the content pipeline

## Congratulations! 🎉

You've completed all 20 labs! You now have hands-on experience with:

- ✅ Agent fundamentals & prompt engineering
- ✅ Function tools & structured output
- ✅ Multi-turn conversations & context
- ✅ Middleware & human-in-the-loop
- ✅ Multi-agent composition (agent-as-tool)
- ✅ Handoff routing in workflows
- ✅ Concurrent & parallel workflows
- ✅ MCP tools (consuming & exposing)
- ✅ A2A protocol (server & client)
- ✅ Conditional routing in workflows
- ✅ Hosted multi-agent systems

**Ready for the hackathon!** 🚀
