# Lab 18: Handoff Workflows — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab18_HandoffWorkflows
cd Lab18_HandoffWorkflows
```

## Step 2: Add NuGet Packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.AI.OpenAI --prerelease
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

## Step 5: Run It

```bash
dotnet run
```

You should see the triage agent route the billing question to BillingSpecialist and the tech question to TechSupport.
