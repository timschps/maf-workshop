# Lab 19: Group Chat — Collaborative Agent Conversations

## Objective

Build a **group chat workflow** where multiple agents collaborate in a shared conversation. A **round-robin manager** orchestrates who speaks next, and agents iteratively refine each other's work. You'll also create a **custom termination manager** that stops the conversation when an approver gives the green light.

## What You'll Learn

- How to use `AgentWorkflowBuilder.CreateGroupChatBuilderWith()` for group chat orchestration
- Using `ChatClientAgent` to create agents from an `IChatClient`
- Using `RoundRobinGroupChatManager` for speaker selection
- Building a custom manager with early termination logic
- Processing the full conversation history from `WorkflowOutputEvent`

## Prerequisites

- Completed Lab 11 (Simple Workflows) and Lab 18 (Handoff Workflows)
- Azure OpenAI endpoint configured

## Duration

25 minutes

## Architecture

```
                    ┌──────────────────────────┐
                    │   Group Chat Manager     │
                    │   (Round-Robin / Custom)  │
                    └────────┬─────────────────┘
                             │ selects next speaker
                ┌────────────┼────────────────┐
                │            │                │
                ▼            ▼                ▼
          ┌──────────┐ ┌──────────┐    ┌──────────┐
          │  Writer  │ │ Reviewer │    │   ...    │
          └──────────┘ └──────────┘    └──────────┘
                │            │
                └────────────┘
                  shared history
```

## Steps

### Step 1: Create a new console application

```bash
dotnet new console -n Lab19_GroupChat
cd Lab19_GroupChat
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

### Step 4: Run the application

```bash
dotnet run
```

**Scenario 1** shows a round-robin group chat where the CopyWriter and Reviewer take turns — the writer creates a slogan, the reviewer provides feedback, and they iterate until the max turn count is reached.

**Scenario 2** demonstrates a custom `ApprovalBasedManager` that terminates the conversation early when the Editor responds with "APPROVED". This is more realistic — the chat ends based on content, not just a turn count.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Group Chat** | Multiple agents share a conversation, coordinated by a manager |
| `ChatClientAgent` | Creates an agent directly from an `IChatClient` with instructions, name, and description |
| `CreateGroupChatBuilderWith()` | Creates a group chat workflow with a manager factory function |
| `RoundRobinGroupChatManager` | Built-in manager that cycles through agents in order |
| `MaximumIterationCount` | Safety limit — max number of turns before the chat ends |
| `AddParticipants()` | Registers agents as participants in the group chat |
| **Custom Manager** | Extend `RoundRobinGroupChatManager` and override `ShouldTerminateAsync` for custom termination |

## How Group Chat Works

1. **Input** arrives and is placed in the shared conversation history
2. The **manager** selects the next speaker (round-robin by default)
3. The selected agent sees the **full conversation history** and generates a response
4. The manager **broadcasts** the response to all participants
5. The manager checks **termination conditions** (max turns, custom logic)
6. Steps 2–5 repeat until the conversation ends
7. The full conversation is returned as a `WorkflowOutputEvent`

## Group Chat vs. Handoff

| Aspect | Group Chat (Lab 19) | Handoff (Lab 18) |
|--------|---------------------|-------------------|
| **Coordination** | Centralized manager | Triage agent decides |
| **Communication** | Shared history — all agents see everything | Direct handoff — one specialist handles the query |
| **Pattern** | Iterative refinement | One-shot routing |
| **Best for** | Collaborative tasks, review cycles | Categorization and delegation |

## 🎯 Challenge

1. Add a third participant — a "BrandStrategist" who ensures the slogan aligns with brand values
2. Create a custom manager that uses a scoring system: the reviewer rates 1–10, and the chat terminates only when the score is 8 or higher

## What's Next?

In **Lab 20**, you'll build **concurrent workflows** where multiple agents process data simultaneously in parallel.
