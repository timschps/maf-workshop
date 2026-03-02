# Lab 11: Simple Workflows — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Part A: Simple Function Workflow (10 min)

### Step 1: Create the Project

```bash
dotnet new console -n Lab11_Workflows
cd Lab11_Workflows
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
```

### Step 2: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Step 3: Build a Two-Step Function Workflow

Replace `Program.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ── Step 1: Agent to convert text to uppercase ──────────────────────────────
AIAgent uppercaseAgent = chatClient.AsAIAgent(
    instructions: "Convert the user's input text to UPPERCASE. Output only the uppercase text, nothing else.",
    name: "UpperCaseAgent");

// ── Step 2: Agent to reverse the string ─────────────────────────────────────
AIAgent reverseAgent = chatClient.AsAIAgent(
    instructions: "Reverse the user's input text character by character. Output only the reversed text, nothing else.",
    name: "ReverseAgent");

// ── Build and run the workflow ───────────────────────────────────────────────
var workflow = AgentWorkflowBuilder.BuildSequential([uppercaseAgent, reverseAgent]);

Console.WriteLine("=== Running Workflow: UpperCase → Reverse ===");
Console.WriteLine();
var result = await InProcessExecution.Default.RunAsync(workflow, "hello world");
Console.WriteLine();
Console.WriteLine($"Final output: {result}");
// Expected: DLROW OLLEH
```

### Step 4: Run It

```bash
dotnet run
```

**Observe:** Data flows through the sequential workflow: `"hello world"` → `UpperCaseAgent` → `"HELLO WORLD"` → `ReverseAgent` → `"DLROW OLLEH"`.

---

## Part B: Agent Workflow — Research & Write (15 min)

### Step 5: Add an Agent-Powered Workflow

Create a new file or replace `Program.cs` with the expanded version:

```csharp
using System;
using System.ComponentModel;
using System.Linq;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;

// ── Configuration ────────────────────────────────────────────────────────────
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ══════════════════════════════════════════════════════════════════════════════
// WORKFLOW: Topic → Research → Write Summary → Output
// ══════════════════════════════════════════════════════════════════════════════

// ── Create the agents ────────────────────────────────────────────────────────
AIAgent researchAgent = chatClient.AsAIAgent(
    instructions: "You are a research assistant. When given a topic, provide clear, factual bullet points. Be concise and accurate.",
    name: "ResearchAgent");

AIAgent writerAgent = chatClient.AsAIAgent(
    instructions: "You are a professional writer. Transform research notes into polished, engaging prose. Keep it concise — 3 sentences maximum.",
    name: "WriterAgent");

// ── Build and run the workflow ───────────────────────────────────────────────
var workflow = AgentWorkflowBuilder.BuildSequential([researchAgent, writerAgent]);

// ── Run it! ──────────────────────────────────────────────────────────────────
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine("  WORKFLOW: Research → Write Summary");
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine();

var topic = "The impact of artificial intelligence on healthcare";
Console.WriteLine($"Topic: {topic}");
Console.WriteLine();

var result = await InProcessExecution.Default.RunAsync(workflow, topic);

Console.WriteLine();
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine("  FINAL OUTPUT:");
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine();
Console.WriteLine(result);
```

### Step 6: Run the Agent Workflow

```bash
dotnet run
```

**Observe:** The research agent generates facts, those facts flow sequentially to the writer agent, which produces a polished summary.
