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
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
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

## Step 3: Run It

```bash
dotnet run
```

**Observe:** Three LLM agents work in a sequential workflow — each one transforms the data and passes it along via `BuildSequential`. The researcher gathers facts, the writer polishes them, and the translator produces the final French version.

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
- [ ] You understand how `AgentWorkflowBuilder.BuildSequential` and `InProcessExecution.Default.RunAsync` work together

---

## 📚 Reference

- [Workflows overview](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Official Step 5: Workflows](https://learn.microsoft.com/en-us/agent-framework/get-started/workflows)
