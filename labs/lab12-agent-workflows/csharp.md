# Lab 12: Agent Workflows — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab12_AgentWorkflows
cd Lab12_AgentWorkflows
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
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

## Step 3: Build a Research → Writer → Translator Pipeline

Replace `Program.cs`:

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ── Create the agents ────────────────────────────────────────────────────────
AIAgent researchAgent = chatClient.AsAIAgent(
    instructions: "You are a research assistant. Provide clear, factual bullet points. Be concise.",
    name: "Researcher");

AIAgent writerAgent = chatClient.AsAIAgent(
    instructions: "You are a professional blog writer. Transform notes into engaging, polished prose. 3 sentences max.",
    name: "Writer");

AIAgent translatorAgent = chatClient.AsAIAgent(
    instructions: "You are a professional translator. Translate the given text to French. Provide accurate, natural translations. Output only the translated text.",
    name: "Translator");

// ── Build a sequential workflow: Research → Write → Translate ─────────────────
var workflow = AgentWorkflowBuilder.BuildSequential([researchAgent, writerAgent, translatorAgent]);

// ── Run the pipeline ─────────────────────────────────────────────────────────
var topic = "The impact of artificial intelligence on healthcare";

Console.WriteLine("╔══════════════════════════════════════════════════════════╗");
Console.WriteLine("║  WORKFLOW: Research → Write → Translate                 ║");
Console.WriteLine("╚══════════════════════════════════════════════════════════╝");
Console.WriteLine($"\n📎 Topic: {topic}\n");

var result = await InProcessExecution.Default.RunAsync(workflow, topic);

Console.WriteLine("╔══════════════════════════════════════════════════════════╗");
Console.WriteLine("║  FINAL OUTPUT (French):                                 ║");
Console.WriteLine("╚══════════════════════════════════════════════════════════╝\n");
Console.WriteLine(result);
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:** Three LLM agents work in a sequential workflow — each one transforms the data and passes it along via `BuildSequential`. The researcher gathers facts, the writer polishes them, and the translator produces the final French version.
