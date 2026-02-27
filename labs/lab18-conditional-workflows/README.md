# Lab 18: Handoff Workflows — Intelligent Agent Routing

## Objective

Build a workflow where a **triage agent** intelligently routes customer queries to the right specialist using **handoff workflows**. You'll create a customer service system where the triage agent analyzes messages and hands off to a billing specialist or tech support agent.

## What You'll Learn

- How to use `AgentWorkflowBuilder.CreateHandoffBuilderWith()` for agent routing
- Defining handoff instructions and routes between agents
- Streaming agent responses with `AgentResponseUpdateEvent`
- Why handoff workflows must be rebuilt for each run (single-use)

## Prerequisites

- Completed Lab 11 (Simple Workflows)
- Azure OpenAI endpoint configured

## Duration

25 minutes

## Architecture

```
                              ┌─ [Billing query]  → BillingSpecialist → response
                              │
Customer Input → TriageAgent ─┤
                              │
                              └─ [Tech issue]     → TechSupport → response
```

## Steps

### Step 1: Create a new console application

```bash
dotnet new console -n Lab18_HandoffWorkflows
cd Lab18_HandoffWorkflows
```

### Step 2: Add NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.AI.OpenAI --prerelease
```

### Step 3: Replace Program.cs

Replace the contents of `Program.cs` with:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;
using OpenAI.Chat;

#pragma warning disable MEAI001

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

var triageAgent = chatClient.AsAIAgent(
    instructions: @"You are a customer service triage agent. Analyze the customer message and route to the right specialist.
- Billing/payment/invoice -> hand off to BillingSpecialist
- Technical issues/bugs/how-to -> hand off to TechSupport
- General -> answer directly",
    name: "TriageAgent");

var billingAgent = chatClient.AsAIAgent(
    instructions: "You are a billing specialist. Help with payment, invoice, refund questions. Be concise (under 80 words).",
    name: "BillingSpecialist");

var techAgent = chatClient.AsAIAgent(
    instructions: "You are tech support. Help troubleshoot issues and provide solutions. Be concise (under 80 words).",
    name: "TechSupport");

Workflow BuildWorkflow() => AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffInstructions("Route the customer to the appropriate specialist based on their query type.")
    .WithHandoff(triageAgent, billingAgent, "Billing, payment, and invoice related questions")
    .WithHandoff(triageAgent, techAgent, "Technical issues, bugs, and how-to questions")
    .Build();

async Task TestScenario(string scenario, string message)
{
    Console.WriteLine($"=== {scenario} ===");
    Console.WriteLine($"Customer: {message}\n");

    var workflow = BuildWorkflow();
    var run = await InProcessExecution.Default.RunAsync<Microsoft.Extensions.AI.ChatMessage>(
        workflow,
        new Microsoft.Extensions.AI.ChatMessage(ChatRole.User, message));

    string? currentAgent = null;
    var responseText = new System.Text.StringBuilder();

    foreach (var evt in run.OutgoingEvents)
    {
        if (evt is AgentResponseUpdateEvent updateEvt)
        {
            var agentId = updateEvt.ExecutorId ?? "";
            var agentName = agentId.Contains("_") ? agentId.Split('_')[0] : agentId;
            if (agentName != currentAgent && !string.IsNullOrEmpty(updateEvt.Update?.Text))
            {
                if (currentAgent != null) Console.WriteLine($"  [{currentAgent}]: {responseText}\n");
                currentAgent = agentName;
                responseText.Clear();
            }
            responseText.Append(updateEvt.Update?.Text ?? "");
        }
    }
    if (currentAgent != null) Console.WriteLine($"  [{currentAgent}]: {responseText}\n");
}

await TestScenario("Billing Query", "I was charged twice for my subscription last month. Can I get a refund?");
await TestScenario("Tech Support", "My app keeps crashing when I try to upload files. I'm on version 3.2.");

Console.WriteLine("Handoff workflow complete!");
```

### Step 4: Run the application

```bash
dotnet run
```

You should see the triage agent route the billing question to BillingSpecialist and the tech question to TechSupport.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Handoff Workflow** | An agent routing pattern where a triage agent delegates to specialists |
| `CreateHandoffBuilderWith()` | Creates a workflow builder with a root agent that can hand off to others |
| `WithHandoffInstructions()` | Sets instructions for how the triage agent should route messages |
| `WithHandoff(from, to, desc)` | Defines a handoff route from one agent to another |
| **Single-Use Workflows** | Handoff workflows must be rebuilt for each run (`BuildWorkflow()` factory) |
| `AgentResponseUpdateEvent` | Event emitted as agents stream their responses |

## How Handoff Routing Works

1. The **TriageAgent** receives the customer message
2. Based on its instructions and the handoff definitions, it decides which specialist to route to
3. The workflow **hands off** to the selected specialist agent
4. The specialist processes the query and returns a response
5. Events stream back via `AgentResponseUpdateEvent` as the agents work

> **Important:** Handoff workflows are single-use — you must rebuild the workflow for each new conversation. That's why we use the `BuildWorkflow()` factory function.

## 🎯 Challenge

Add a third specialist: a "SalesAgent" for product inquiries and pricing questions. Update the triage agent's instructions and add the handoff route!

## What's Next?

In **Lab 19**, you'll build **concurrent workflows** where multiple agents process data simultaneously in parallel.
