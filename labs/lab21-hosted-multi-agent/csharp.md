# Lab 21: Hosted Multi-Agent Workflow — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new web -n Lab21_HostedMultiAgent
cd Lab21_HostedMultiAgent
```

## Step 2: Add NuGet Packages

```bash
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
dotnet add package Microsoft.Agents.AI.Workflows
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore
dotnet add package Microsoft.Extensions.AI
dotnet add package Microsoft.Extensions.AI.OpenAI
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

## Step 4: Replace Program.cs

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

## Step 5: Set the Server Port

Add to `Properties/launchSettings.json` (or create it):

```json
{
  "profiles": {
    "Lab21_HostedMultiAgent": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5200"
    }
  }
}
```

## Step 6: Run It

```bash
dotnet run
```

## Step 7: Test with curl

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
