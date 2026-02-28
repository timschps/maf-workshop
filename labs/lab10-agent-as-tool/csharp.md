# Lab 10: Agent-as-a-Tool — C# Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab10_AgentAsTool
cd Lab10_AgentAsTool
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 3: Build Specialist Agents

Replace `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
var client = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ── Tool functions for specialists ───────────────────────────────────────────
[Description("Get the current weather for a city.")]
static string GetWeather([Description("City name.")] string city)
    => $"Weather in {city}: {new[] { "sunny", "cloudy", "rainy" }[Random.Shared.Next(3)]}, {Random.Shared.Next(10, 30)}°C.";

[Description("Get the current time in a timezone.")]
static string GetTime([Description("Timezone, e.g. CET, EST, JST.")] string tz)
{
    var offsets = new Dictionary<string, int> { ["UTC"] = 0, ["CET"] = 1, ["EST"] = -5, ["JST"] = 9 };
    return offsets.TryGetValue(tz.ToUpper(), out var h)
        ? $"Time in {tz}: {DateTimeOffset.UtcNow.ToOffset(TimeSpan.FromHours(h)):HH:mm}."
        : $"Unknown timezone: {tz}.";
}

[Description("Convert currency.")]
static string ConvertCurrency(
    [Description("Amount.")] double amount,
    [Description("Source currency code.")] string from,
    [Description("Target currency code.")] string to)
{
    var rates = new Dictionary<string, double> { ["USD"] = 1.0, ["EUR"] = 0.92, ["JPY"] = 149.5, ["GBP"] = 0.79 };
    if (rates.TryGetValue(from.ToUpper(), out var f) && rates.TryGetValue(to.ToUpper(), out var t))
        return $"{amount} {from} = {(amount / f * t):F2} {to}.";
    return $"Unsupported currency pair.";
}

// ══════════════════════════════════════════════════════════════════════════════
// SPECIALIST AGENTS
// ══════════════════════════════════════════════════════════════════════════════

AIAgent weatherAgent = client.AsAIAgent(
    instructions: "You are a weather specialist. Answer weather questions concisely.",
    name: "WeatherAgent",
    description: "Provides current weather information for any city in the world.",
    tools: [AIFunctionFactory.Create(GetWeather)]);

AIAgent timeAgent = client.AsAIAgent(
    instructions: "You are a timezone specialist. Answer time-related questions.",
    name: "TimeAgent",
    description: "Provides the current time in any timezone.",
    tools: [AIFunctionFactory.Create(GetTime)]);

AIAgent financeAgent = client.AsAIAgent(
    instructions: "You are a currency specialist. Help with currency conversions.",
    name: "FinanceAgent",
    description: "Converts between currencies using current exchange rates.",
    tools: [AIFunctionFactory.Create(ConvertCurrency)]);

// ══════════════════════════════════════════════════════════════════════════════
// MANAGER AGENT — delegates to specialists
// ══════════════════════════════════════════════════════════════════════════════

AIAgent managerAgent = client.AsAIAgent(
    instructions: """
        You are a travel planning assistant. You coordinate with specialist agents:
        - WeatherAgent: for weather information
        - TimeAgent: for time zone queries
        - FinanceAgent: for currency conversions

        When a user asks a complex question, delegate to the appropriate specialists
        and synthesize their responses into a helpful answer.
        Always be concise and well-organized.
        """,
    name: "TravelPlanner",
    tools: [
        weatherAgent.AsAIFunction(),
        timeAgent.AsAIFunction(),
        financeAgent.AsAIFunction(),
    ]);

// ── Test with various complexity levels ──────────────────────────────────────
var questions = new[]
{
    "What's the weather like in Tokyo right now?",
    "What time is it in CET?",
    "I'm traveling from New York to Tokyo next week. What's the weather there, what time is it, and how much is 1000 USD in JPY?",
};

foreach (var q in questions)
{
    Console.WriteLine($"❓ {q}\n");
    Console.Write("💬 ");
    await foreach (var update in managerAgent.RunStreamingAsync(q))
        Console.Write(update);
    Console.WriteLine("\n" + new string('─', 70) + "\n");
}
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:**
- Simple questions: manager delegates to one specialist
- Complex questions: manager delegates to **multiple** specialists and combines results
- Each specialist has its own tools — the manager doesn't know about `GetWeather` directly

### Exercise C: Named Tool Customization

```csharp
// Override how the specialist appears to the manager
var customTool = weatherAgent.AsAIFunction();
// The manager sees "WeatherAgent" with description "Provides current weather..."
```
