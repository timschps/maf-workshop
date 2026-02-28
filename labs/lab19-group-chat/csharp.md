# Lab 19: Group Chat — C# Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab19_GroupChat
cd Lab19_GroupChat
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
using System.Threading;
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

var client = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient();

// --- Scenario 1: Round-Robin Group Chat ---
Console.WriteLine("========================================");
Console.WriteLine("  Scenario 1: Round-Robin Group Chat");
Console.WriteLine("========================================\n");

var writer1 = new ChatClientAgent(client,
    "You are a creative copywriter. Generate catchy slogans and marketing copy. Be concise and impactful. Keep responses under 60 words.",
    "CopyWriter",
    "A creative copywriter agent");

var reviewer1 = new ChatClientAgent(client,
    "You are a marketing reviewer. Evaluate slogans for clarity, impact, and brand alignment. " +
    "Provide constructive feedback or say APPROVED if the slogan is ready. Keep responses under 60 words.",
    "Reviewer",
    "A marketing review agent");

// Build group chat with round-robin speaker selection
var workflow1 = AgentWorkflowBuilder
    .CreateGroupChatBuilderWith(agents =>
        new RoundRobinGroupChatManager(agents)
        {
            MaximumIterationCount = 4  // Max 4 turns total
        })
    .AddParticipants(writer1, reviewer1)
    .Build();

var run1 = await InProcessExecution.Default.RunAsync<Microsoft.Extensions.AI.ChatMessage>(
    workflow1,
    new Microsoft.Extensions.AI.ChatMessage(ChatRole.User, "Create a slogan for an eco-friendly electric vehicle."));

foreach (var evt in run1.OutgoingEvents)
{
    if (evt is WorkflowOutputEvent outputEvt)
    {
        if (outputEvt.Data is List<Microsoft.Extensions.AI.ChatMessage> msgs)
        {
            foreach (var m in msgs)
                Console.WriteLine($"  [{m.AuthorName ?? m.Role.ToString()}]: {m.Text}");
        }
    }
}

// --- Scenario 2: Custom Approval-Based Manager ---
Console.WriteLine("\n========================================");
Console.WriteLine("  Scenario 2: Approval-Based Manager");
Console.WriteLine("========================================\n");

var writer2 = new ChatClientAgent(client,
    "You are a creative copywriter. Write short product descriptions (under 50 words). Revise based on feedback.",
    "Writer",
    "A product description writer");

var editor2 = new ChatClientAgent(client,
    "You are a senior editor. Review product descriptions for quality. " +
    "If the description is good, respond with exactly 'APPROVED: ' followed by the final text. " +
    "Otherwise, provide specific feedback for improvement. Keep responses under 60 words.",
    "Editor",
    "A senior editor agent");

// Build group chat with custom approval-based manager
var workflow2 = AgentWorkflowBuilder
    .CreateGroupChatBuilderWith(agents =>
        new ApprovalBasedManager(agents, "Editor")
        {
            MaximumIterationCount = 6  // Safety limit
        })
    .AddParticipants(writer2, editor2)
    .Build();

var run2 = await InProcessExecution.Default.RunAsync<Microsoft.Extensions.AI.ChatMessage>(
    workflow2,
    new Microsoft.Extensions.AI.ChatMessage(ChatRole.User, "Write a product description for a smart water bottle that tracks hydration."));

foreach (var evt in run2.OutgoingEvents)
{
    if (evt is WorkflowOutputEvent outputEvt)
    {
        if (outputEvt.Data is List<Microsoft.Extensions.AI.ChatMessage> msgs)
        {
            foreach (var m in msgs)
                Console.WriteLine($"  [{m.AuthorName ?? m.Role.ToString()}]: {m.Text}");
        }
    }
}

Console.WriteLine("\nGroup chat complete!");

// --- Custom Manager: terminates when the approver says "APPROVED" ---
public class ApprovalBasedManager : RoundRobinGroupChatManager
{
    private readonly string _approverName;

    public ApprovalBasedManager(IReadOnlyList<AIAgent> agents, string approverName)
        : base(agents)
    {
        _approverName = approverName;
    }

    protected override ValueTask<bool> ShouldTerminateAsync(
        IReadOnlyList<Microsoft.Extensions.AI.ChatMessage> history,
        CancellationToken cancellationToken = default)
    {
        var last = history.LastOrDefault();
        bool shouldTerminate = last?.AuthorName == _approverName &&
            (last.Text?.Contains("APPROVED", StringComparison.OrdinalIgnoreCase) == true);

        return ValueTask.FromResult(shouldTerminate);
    }
}
```

## Step 5: Run It

```bash
dotnet run
```

**Scenario 1** shows a round-robin group chat where the CopyWriter and Reviewer take turns — the writer creates a slogan, the reviewer provides feedback, and they iterate until the max turn count is reached.

**Scenario 2** demonstrates a custom `ApprovalBasedManager` that terminates the conversation early when the Editor responds with "APPROVED". This is more realistic — the chat ends based on content, not just a turn count.
