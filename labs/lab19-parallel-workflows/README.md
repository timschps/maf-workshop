# Lab 19: Concurrent Workflows — Parallel Agent Execution

## Objective

Build a workflow where multiple agents work **in parallel** using the `BuildConcurrent` pattern. A single input is distributed to multiple specialist agents simultaneously, and their results are combined by a merge function.

## What You'll Learn

- How to use `AgentWorkflowBuilder.BuildConcurrent()` for parallel execution
- Defining a merge function to combine results from multiple agents
- How all agents receive the same input simultaneously
- Processing `WorkflowOutputEvent` for merged results

## Prerequisites

- Completed Lab 11 (Simple Workflows)
- Azure OpenAI endpoint configured

## Duration

25 minutes

## Architecture

```
                  ┌─────────────────────────┐
                  │   HistoryResearcher      │──┐
                  └─────────────────────────┘  │
                                                │
Topic Input ──▶   ┌─────────────────────────┐  ├──▶ Merge Function ──▶ Combined Report
                  │   CultureResearcher      │──┤
                  └─────────────────────────┘  │
                                                │
                  ┌─────────────────────────┐  │
                  │   TravelTipsResearcher   │──┘
                  └─────────────────────────┘

              ◀──── Concurrent ────▶  ◀── Merge ──▶
```

## Steps

### Step 1: Create a new console application

```bash
dotnet new console -n Lab19_ConcurrentWorkflows
cd Lab19_ConcurrentWorkflows
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
using System.Text;
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

var historyAgent = chatClient.AsAIAgent(
    instructions: "You are a history researcher. When given a country, provide 3-4 key historical facts. Be concise (under 80 words). Start with 'HISTORY:'",
    name: "HistoryResearcher");

var cultureAgent = chatClient.AsAIAgent(
    instructions: "You are a culture researcher. When given a country, describe 3-4 cultural highlights (food, customs, art). Be concise (under 80 words). Start with 'CULTURE:'",
    name: "CultureResearcher");

var travelAgent = chatClient.AsAIAgent(
    instructions: "You are a travel tips expert. When given a country, provide 3-4 practical travel tips. Be concise (under 80 words). Start with 'TRAVEL TIPS:'",
    name: "TravelTipsResearcher");

var workflow = AgentWorkflowBuilder.BuildConcurrent(
    [historyAgent, cultureAgent, travelAgent],
    (IList<List<Microsoft.Extensions.AI.ChatMessage>> results) =>
    {
        var combined = new List<Microsoft.Extensions.AI.ChatMessage>();
        var sb = new StringBuilder();
        sb.AppendLine("=== COMBINED RESEARCH REPORT ===\n");
        
        for (int i = 0; i < results.Count; i++)
        {
            foreach (var msg in results[i])
            {
                if (msg.Role == ChatRole.Assistant && !string.IsNullOrWhiteSpace(msg.Text))
                {
                    sb.AppendLine(msg.Text);
                    sb.AppendLine();
                }
            }
        }
        
        combined.Add(new Microsoft.Extensions.AI.ChatMessage(ChatRole.Assistant, sb.ToString()));
        return combined;
    });

string topic = "Japan";
Console.WriteLine($"Researching '{topic}' with 3 parallel agents...\n");

var run = await InProcessExecution.Default.RunAsync<Microsoft.Extensions.AI.ChatMessage>(
    workflow,
    new Microsoft.Extensions.AI.ChatMessage(ChatRole.User, $"Tell me about {topic}"));

foreach (var evt in run.OutgoingEvents)
{
    if (evt is WorkflowOutputEvent outputEvt)
    {
        if (outputEvt.Data is List<Microsoft.Extensions.AI.ChatMessage> msgs)
        {
            foreach (var m in msgs)
                Console.WriteLine(m.Text);
        }
    }
}

Console.WriteLine("Concurrent workflow complete!");
```

### Step 4: Run the application

```bash
dotnet run
```

You should see all three researchers working simultaneously on the topic, with their results merged into a combined report.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Concurrent Workflow** | Multiple agents process the same input simultaneously |
| `BuildConcurrent()` | Creates a workflow that runs agents in parallel and merges results |
| **Merge Function** | `Func<IList<List<ChatMessage>>, List<ChatMessage>>` — combines all agent outputs |
| **Parallel Execution** | All agents receive the same input and run concurrently |
| `WorkflowOutputEvent` | Contains the merged result from all parallel agents |

## How It Works

1. **Input** arrives and is sent to all agents simultaneously
2. **Parallel Processing**: History, Culture, and Travel researchers work concurrently
3. **Merge**: The merge function receives a list of results (one per agent)
4. **Combination**: The merge function assembles a combined report from all results
5. **Output**: The merged report is yielded as a `WorkflowOutputEvent`

## Performance Benefits

Concurrent execution significantly reduces total time. Instead of running three researchers sequentially (~9 seconds), they run in parallel (~3 seconds):

```
Sequential:  [History: 3s] → [Culture: 3s] → [Travel: 3s] = ~9s total
Concurrent:  [History: 3s]
             [Culture: 3s]  = ~3s total
             [Travel: 3s]
```

## 🎯 Challenge

Add a fourth researcher (e.g., "BudgetExpert") and update the merge function to include its output. See how the concurrent pattern scales!

## What's Next?

In **Lab 20**, you'll combine everything — hosting multiple agents as a workflow, exposed via A2A protocol, creating a fully **hosted multi-agent system**.
