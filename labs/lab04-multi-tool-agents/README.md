# Lab 4: Multi-Tool Agents

**Duration:** 20 minutes
**Objective:** Build an agent with multiple tools and observe how the LLM decides which tool(s) to call based on the user's question.

---

## What You'll Learn

- How to register multiple tools with a single agent
- How the LLM selects the right tool(s) based on descriptions
- What happens when a question requires multiple tools
- The importance of clear, distinct tool descriptions

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab4_MultiTool
cd Lab4_MultiTool
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Define Multiple Tools

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

// ── Tool 1: Weather ──────────────────────────────────────────────────────────
[Description("Get the current weather for a given city.")]
static string GetWeather(
    [Description("The city name, e.g. 'Amsterdam' or 'Tokyo'.")] string city)
{
    var conditions = new[] { "sunny", "cloudy", "rainy", "partly cloudy" };
    return $"Weather in {city}: {conditions[Random.Shared.Next(conditions.Length)]}, {Random.Shared.Next(5, 30)}°C.";
}

// ── Tool 2: Time ─────────────────────────────────────────────────────────────
[Description("Get the current local time in a given timezone.")]
static string GetTime(
    [Description("Timezone abbreviation, e.g. 'CET', 'EST', 'JST', 'UTC'.")] string timezone)
{
    var offsets = new Dictionary<string, int>
    { ["UTC"] = 0, ["CET"] = 1, ["EST"] = -5, ["PST"] = -8, ["JST"] = 9, ["AEST"] = 11 };
    if (offsets.TryGetValue(timezone.ToUpper(), out var offset))
        return $"Current time in {timezone}: {DateTimeOffset.UtcNow.ToOffset(TimeSpan.FromHours(offset)):HH:mm}.";
    return $"Unknown timezone '{timezone}'. Supported: {string.Join(", ", offsets.Keys)}.";
}

// ── Tool 3: Currency ─────────────────────────────────────────────────────────
[Description("Convert an amount from one currency to another.")]
static string ConvertCurrency(
    [Description("The amount to convert.")] double amount,
    [Description("The source currency code, e.g. 'USD', 'EUR'.")] string from,
    [Description("The target currency code, e.g. 'JPY', 'GBP'.")] string to)
{
    var rates = new Dictionary<string, double>
    { ["USD"] = 1.0, ["EUR"] = 0.92, ["GBP"] = 0.79, ["JPY"] = 149.5, ["CHF"] = 0.88 };
    if (rates.TryGetValue(from.ToUpper(), out var fromRate) && rates.TryGetValue(to.ToUpper(), out var toRate))
    {
        var result = amount / fromRate * toRate;
        return $"{amount} {from} = {result:F2} {to}.";
    }
    return $"Unsupported currency. Supported: {string.Join(", ", rates.Keys)}.";
}

// ── Tool 4: Translation ─────────────────────────────────────────────────────
[Description("Translate a short phrase into another language. Returns the translated text.")]
static string Translate(
    [Description("The text to translate.")] string text,
    [Description("The target language, e.g. 'French', 'Spanish', 'Japanese'.")] string targetLanguage)
{
    // In a real app, call a translation API — here we fake it
    return $"[Translated to {targetLanguage}]: «{text}» (simulated translation)";
}

// ── Create the multi-tool agent ──────────────────────────────────────────────
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: """
            You are a helpful travel assistant. You have tools for weather, time,
            currency conversion, and translation. Use the appropriate tools to
            answer the user's questions. If a question requires multiple tools,
            call all of them.
            """,
        tools: [
            AIFunctionFactory.Create(GetWeather),
            AIFunctionFactory.Create(GetTime),
            AIFunctionFactory.Create(ConvertCurrency),
            AIFunctionFactory.Create(Translate),
        ]);

// ── Test with various questions ──────────────────────────────────────────────
var questions = new[]
{
    // Single-tool questions
    "What's the weather in Tokyo?",
    "What time is it in New York (EST)?",
    "Convert 100 USD to EUR.",

    // Multi-tool questions — the agent should call multiple tools!
    "I'm planning a trip to Tokyo. What's the weather, what time is it (JST), and how much is 500 USD in JPY?",

    // Ambiguous question — which tool does it pick?
    "How do I say 'good morning' in Japanese?",

    // No tool needed
    "What's the capital of France?",
};

foreach (var q in questions)
{
    Console.WriteLine($"❓ {q}");
    Console.Write("💬 ");
    await foreach (var update in agent.RunStreamingAsync(q))
        Console.Write(update);
    Console.WriteLine("\n");
}
```

## Step 3: Run and Observe

```bash
dotnet run
```

**Key observations:**
- Single-tool questions → agent calls exactly one tool
- Multi-tool questions → agent calls multiple tools and combines results
- Questions that don't need tools → agent answers from its own knowledge
- The LLM uses tool **descriptions** to make the selection

---

## 🏋️ Exercises

### Exercise A: Add a 5th Tool

Add your own tool (e.g., `GetFlightPrice`, `CheckHotelAvailability`, `GetRestaurantRating`) and ask questions that combine it with existing tools.

### Exercise B: Tool Description Quality

1. Change `GetWeather`'s description to just `"A tool."` — does the agent still call it for weather questions?
2. Make two tools with nearly identical descriptions — does the agent get confused?
3. Restore good descriptions and confirm behavior returns to normal.

**Lesson:** Tool descriptions are the LLM's decision guide. Ambiguity in descriptions → ambiguity in behavior.

### Exercise C (Stretch): Tool Selection Logging

Add function calling middleware (from Lab 8) that logs which tools are called per question. Create a table showing which questions trigger which tools.

---

## ✅ Success Criteria

- [ ] Agent correctly selects the right tool for single-tool questions
- [ ] Agent calls multiple tools and combines results for complex questions
- [ ] Agent answers without tools when no tool is needed
- [ ] You understand how tool descriptions drive selection

---

## 📚 Reference

- [Tools overview](https://learn.microsoft.com/en-us/agent-framework/agents/tools/)
- [Function tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/function-tools)
