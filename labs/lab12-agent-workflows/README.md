# Lab 12: Agent Workflows — LLM-Powered Pipelines

**Duration:** 20 minutes
**Objective:** Build a workflow where real LLM agents collaborate as pipeline steps — Research → Write → Translate.

---

## What You'll Learn

- How to wrap `AIAgent` instances as workflow executors
- How agent output flows as typed messages between workflow steps
- How to build a practical content generation pipeline

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab12_AgentWorkflows
cd Lab12_AgentWorkflows
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Build a Research → Writer → Translator Pipeline

Replace `Program.cs`:

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Workflows;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ══════════════════════════════════════════════════════════════════════════════
// EXECUTOR 1: Research — gathers key facts about a topic
// ══════════════════════════════════════════════════════════════════════════════
class ResearchExecutor : Executor
{
    private readonly AIAgent _agent;
    public ResearchExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Research(string topic, WorkflowContext<string> ctx)
    {
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"\n  📚 [Research] Investigating: '{topic}'...");
        Console.ResetColor();

        var result = await _agent.RunAsync(
            $"Research the following topic and provide exactly 5 key facts as bullet points. Be concise and factual: {topic}");

        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"  📚 [Research] Done — forwarding to Writer.\n");
        Console.ResetColor();

        await ctx.SendMessageAsync(result.ToString()!);
    }
}

// ══════════════════════════════════════════════════════════════════════════════
// EXECUTOR 2: Writer — transforms facts into a polished paragraph
// ══════════════════════════════════════════════════════════════════════════════
class WriterExecutor : Executor
{
    private readonly AIAgent _agent;
    public WriterExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Write(string facts, WorkflowContext<string> ctx)
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("  ✍️  [Writer] Crafting summary...");
        Console.ResetColor();

        var result = await _agent.RunAsync(
            $"Based on these research facts, write a concise and engaging 3-sentence summary paragraph for a blog post:\n\n{facts}");

        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("  ✍️  [Writer] Done — forwarding to Translator.\n");
        Console.ResetColor();

        await ctx.SendMessageAsync(result.ToString()!);
    }
}

// ══════════════════════════════════════════════════════════════════════════════
// EXECUTOR 3: Translator — translates to a target language
// ══════════════════════════════════════════════════════════════════════════════
class TranslatorExecutor : Executor
{
    private readonly AIAgent _agent;
    private readonly string _targetLanguage;

    public TranslatorExecutor(AIAgent agent, string targetLanguage)
    {
        _agent = agent;
        _targetLanguage = targetLanguage;
    }

    [Handler]
    public async Task Translate(string text, WorkflowContext<Never, string> ctx)
    {
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine($"  🌍 [Translator] Translating to {_targetLanguage}...");
        Console.ResetColor();

        var result = await _agent.RunAsync(
            $"Translate the following text to {_targetLanguage}. Only output the translation, no explanations:\n\n{text}");

        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine("  🌍 [Translator] Done.\n");
        Console.ResetColor();

        await ctx.YieldOutputAsync(result.ToString()!);
    }
}

// ── Create the agents ────────────────────────────────────────────────────────
AIAgent researchAgent = chatClient.AsAIAgent(
    instructions: "You are a research assistant. Provide clear, factual bullet points. Be concise.",
    name: "Researcher");

AIAgent writerAgent = chatClient.AsAIAgent(
    instructions: "You are a professional blog writer. Transform notes into engaging, polished prose. 3 sentences max.",
    name: "Writer");

AIAgent translatorAgent = chatClient.AsAIAgent(
    instructions: "You are a professional translator. Provide accurate, natural translations. Output only the translated text.",
    name: "Translator");

// ── Build the workflow graph ─────────────────────────────────────────────────
var research = new ResearchExecutor(researchAgent);
var writer = new WriterExecutor(writerAgent);
var translator = new TranslatorExecutor(translatorAgent, "French");

var workflow = new AgentWorkflowBuilder(startExecutor: research)
    .AddEdge(research, writer)
    .AddEdge(writer, translator)
    .Build();

// ── Run the pipeline ─────────────────────────────────────────────────────────
var topic = "The impact of artificial intelligence on healthcare";

Console.WriteLine("╔══════════════════════════════════════════════════════════╗");
Console.WriteLine("║  WORKFLOW: Research → Write → Translate                 ║");
Console.WriteLine("╚══════════════════════════════════════════════════════════╝");
Console.WriteLine($"\n📎 Topic: {topic}\n");

var output = await workflow.RunAsync(topic);

Console.WriteLine("╔══════════════════════════════════════════════════════════╗");
Console.WriteLine("║  FINAL OUTPUT (French):                                 ║");
Console.WriteLine("╚══════════════════════════════════════════════════════════╝\n");
Console.WriteLine(string.Join("\n", output.GetOutputs()));
```

## Step 3: Run It

```bash
dotnet run
```

**Observe:** Three LLM agents work in sequence — each one transforms the data and passes it along. The researcher gathers facts, the writer polishes them, and the translator produces the final French version.

---

## 🏋️ Exercises

### Exercise A: Change the Target Language

Modify the `TranslatorExecutor` to accept a different target language. Try Spanish, Japanese, or Dutch.

### Exercise B: Add a Quality Check Step

Insert a `ReviewerExecutor` between Writer and Translator that evaluates quality and either passes through or requests a rewrite:

```
Research → Writer → Reviewer → Translator → Output
```

### Exercise C (Stretch): Parallel Research

Create two research agents that investigate different aspects of the topic in parallel, then have the writer synthesize both:

```
              ┌→ TechResearch ──────┐
Topic ───────→│                      │→ Writer → Translator → Output
              └→ EthicsResearch ─────┘
```

---

## ✅ Success Criteria

- [ ] Three-step pipeline runs successfully: Research → Write → Translate
- [ ] Each agent transforms the output and passes it to the next step
- [ ] Console logs clearly show data flowing through each executor
- [ ] You understand how `SendMessageAsync` (forward) vs `YieldOutputAsync` (final output) work

---

## 📚 Reference

- [Workflows overview](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Official Step 5: Workflows](https://learn.microsoft.com/en-us/agent-framework/get-started/workflows)
