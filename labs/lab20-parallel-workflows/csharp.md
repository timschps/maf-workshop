# Lab 20: Concurrent Workflows — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab20_ConcurrentWorkflows
cd Lab20_ConcurrentWorkflows
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

## Step 5: Run It

```bash
dotnet run
```

You should see all three researchers working simultaneously on the topic, with their results merged into a combined report.
