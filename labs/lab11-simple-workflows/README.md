# Lab 11: Simple Workflows — Function Pipelines

**Duration:** 15 minutes
**Objective:** Understand workflow basics by building function-based pipelines with executors and edges.

---

## What You'll Learn

- The difference between agents (open-ended) and workflows (explicit control)
- Workflow building blocks: Executors, Edges, WorkflowContext
- How to chain function executors and agent executors
- How data flows through a workflow graph

---

## Part A: Simple Function Workflow (10 min)

Start with a pure function workflow (no LLM) to understand the mechanics.

### Step 1: Create the Project

```bash
dotnet new console -n Lab4_Workflows
cd Lab4_Workflows
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

### Step 2: Build a Two-Step Function Workflow

Replace `Program.cs`:

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Agents.AI.Workflows;

// ── Step 1: Convert text to uppercase ────────────────────────────────────────
class UpperCase : Executor
{
    [Handler]
    public async Task ToUpperCase(string text, WorkflowContext<string> ctx)
    {
        Console.WriteLine($"  [UpperCase] Input:  '{text}'");
        var result = text.ToUpper();
        Console.WriteLine($"  [UpperCase] Output: '{result}'");
        await ctx.SendMessageAsync(result);
    }
}

// ── Step 2: Reverse the string and yield as workflow output ──────────────────
[Executor(Id = "reverse_text")]
static async Task ReverseText(string text, WorkflowContext<Never, string> ctx)
{
    Console.WriteLine($"  [Reverse]   Input:  '{text}'");
    var reversed = new string(text.Reverse().ToArray());
    Console.WriteLine($"  [Reverse]   Output: '{reversed}'");
    await ctx.YieldOutputAsync(reversed);
}

// ── Build and run the workflow ───────────────────────────────────────────────
var upper = new UpperCase();
var workflow = new AgentWorkflowBuilder(startExecutor: upper)
    .AddEdge(upper, ReverseText)
    .Build();

Console.WriteLine("=== Running Workflow: UpperCase → Reverse ===");
Console.WriteLine();
var result = await workflow.RunAsync("hello world");
Console.WriteLine();
Console.WriteLine($"Final output: {string.Join(", ", result.GetOutputs())}");
// Expected: DLROW OLLEH
```

### Step 3: Run It

```bash
dotnet run
```

**Observe:** Data flows through the graph: `"hello world"` → `UpperCase` → `"HELLO WORLD"` → `Reverse` → `"DLROW OLLEH"`.

---

## Part B: Agent Workflow — Research & Write (15 min)

Now let's build a workflow that uses real LLM agents.

### Step 4: Add an Agent-Powered Workflow

Create a new file or replace `Program.cs` with the expanded version:

```csharp
using System;
using System.ComponentModel;
using System.Linq;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
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

// ── Step 1: Research executor — uses an agent to gather facts ────────────────
class ResearchExecutor : Executor
{
    private readonly AIAgent _agent;

    public ResearchExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Research(string topic, WorkflowContext<string> ctx)
    {
        Console.WriteLine($"  [Research] Researching: '{topic}'...");

        var prompt = $"Research the following topic and provide 5 key facts as bullet points: {topic}";
        var result = await _agent.RunAsync(prompt);

        Console.WriteLine($"  [Research] Done — passing facts to next step.");
        await ctx.SendMessageAsync(result.ToString()!);
    }
}

// ── Step 2: Writer executor — uses an agent to write a summary ───────────────
class WriterExecutor : Executor
{
    private readonly AIAgent _agent;

    public WriterExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Write(string facts, WorkflowContext<Never, string> ctx)
    {
        Console.WriteLine($"  [Writer] Writing summary from facts...");

        var prompt = $"Based on these research facts, write a concise 3-sentence summary paragraph suitable for a blog post:\n\n{facts}";
        var result = await _agent.RunAsync(prompt);

        Console.WriteLine($"  [Writer] Done.");
        await ctx.YieldOutputAsync(result.ToString()!);
    }
}

// ── Create the agents ────────────────────────────────────────────────────────
AIAgent researchAgent = chatClient.AsAIAgent(
    instructions: "You are a research assistant. When given a topic, provide clear, factual bullet points. Be concise and accurate.",
    name: "ResearchAgent");

AIAgent writerAgent = chatClient.AsAIAgent(
    instructions: "You are a professional writer. Transform research notes into polished, engaging prose. Keep it concise — 3 sentences maximum.",
    name: "WriterAgent");

// ── Build the workflow graph ─────────────────────────────────────────────────
var research = new ResearchExecutor(researchAgent);
var writer = new WriterExecutor(writerAgent);

var workflow = new AgentWorkflowBuilder(startExecutor: research)
    .AddEdge(research, writer)
    .Build();

// ── Run it! ──────────────────────────────────────────────────────────────────
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine("  WORKFLOW: Research → Write Summary");
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine();

var topic = "The impact of artificial intelligence on healthcare";
Console.WriteLine($"Topic: {topic}");
Console.WriteLine();

var output = await workflow.RunAsync(topic);

Console.WriteLine();
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine("  FINAL OUTPUT:");
Console.WriteLine("═══════════════════════════════════════════════════════");
Console.WriteLine();
Console.WriteLine(string.Join("\n", output.GetOutputs()));
```

### Step 5: Run the Agent Workflow

```bash
dotnet run
```

**Observe:** The research agent generates facts, those facts flow through the edge to the writer agent, which produces a polished summary.

---

## 🏋️ Exercises

### Exercise A: Add a Third Step — Translator

Add a `TranslatorExecutor` that takes the English summary and translates it:

```
Topic → Research → Writer → Translator → Output
```

- Create a `TranslatorAgent` with instructions: `"Translate the following text to French."`
- Add a `TranslatorExecutor` that wraps this agent
- Add an edge from `writer` to `translator`
- Make the translator yield the final output

### Exercise B: Three-Step Pipeline with Validation

Add a validation step between Research and Writer:

```
Topic → Research → Validator → Writer → Output
```

The Validator checks if the research has at least 3 bullet points. If not, the workflow should request more research (you can simulate this with a conditional edge or by having the validator simply append "Please provide more detail" and loop back).

### Exercise C (Stretch): Parallel Fan-Out

Create a workflow where **two research agents** work in parallel on different aspects of a topic, then a writer combines both:

```
             ┌→ TechResearch ──┐
Topic ──────→│                  │→ Writer → Output
             └→ BusinessResearch┘
```

Hint: Look into `AddEdge` with multiple sources or `BuildConcurrent` patterns.

---

## ✅ Success Criteria

- [ ] Simple function workflow runs: UpperCase → Reverse
- [ ] Agent workflow runs: Research → Writer
- [ ] You can see data flowing between steps via the console logs
- [ ] You understand the difference between `SendMessageAsync` (forward to next) and `YieldOutputAsync` (final output)

---

## 🐍 Python Alternative

<details>
<summary>Click to expand Python version</summary>

```python
import asyncio
import os
from typing import Never
from azure.identity import AzureCliCredential
from agent_framework import Executor, WorkflowBuilder, WorkflowContext, handler
from agent_framework.azure import AzureOpenAIResponsesClient

# Simple function workflow
class UpperCase(Executor):
    def __init__(self):
        super().__init__(id="upper_case")

    @handler
    async def to_upper_case(self, text: str, ctx: WorkflowContext[str]) -> None:
        print(f"  [UpperCase] '{text}' → '{text.upper()}'")
        await ctx.send_message(text.upper())

@executor(id="reverse_text")
async def reverse_text(text: str, ctx: WorkflowContext[Never, str]) -> None:
    reversed_text = text[::-1]
    print(f"  [Reverse] '{text}' → '{reversed_text}'")
    await ctx.yield_output(reversed_text)

async def main():
    # Simple workflow
    upper = UpperCase()
    workflow = WorkflowBuilder(start_executor=upper).add_edge(upper, reverse_text).build()

    events = await workflow.run("hello world")
    print(f"Output: {events.get_outputs()}")

asyncio.run(main())
```

</details>

---

## 📚 Reference

- [Official Step 5: Workflows](https://learn.microsoft.com/en-us/agent-framework/get-started/workflows)
- [Workflows overview](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Workflow samples on GitHub](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/03-workflows)
