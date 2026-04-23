# Lab 3: Agent with Tools — Function Calling — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab3_AgentTools
cd Lab3_AgentTools
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
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

## Step 3: Define Your First Function Tool

A function tool is just a C# method with `[Description]` attributes. The descriptions tell the LLM **what** the tool does and **when** to call it.

Create `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;

// ── Configuration ────────────────────────────────────────────────────────────
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// ── Define tools ─────────────────────────────────────────────────────────────

[Description("Get the current weather for a given location.")]
static string GetWeather(
    [Description("The city name, e.g. 'Amsterdam' or 'Seattle'.")] string location)
{
    // In a real app, this would call a weather API
    var conditions = new[] { "sunny", "cloudy", "rainy", "partly cloudy" };
    var temp = Random.Shared.Next(5, 30);
    var condition = conditions[Random.Shared.Next(conditions.Length)];
    return $"The weather in {location} is {condition} with a temperature of {temp}°C.";
}

// ── Create agent with the tool ───────────────────────────────────────────────
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful weather assistant. Use the GetWeather tool to answer weather questions. If the user asks about something else, politely redirect to weather topics.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// ── Ask a question that requires the tool ────────────────────────────────────
Console.WriteLine("Question: What is the weather like in Amsterdam?");
Console.WriteLine();
Console.WriteLine(await agent.RunAsync("What is the weather like in Amsterdam?"));
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:** The agent automatically decided to call `GetWeather("Amsterdam")`, received the result, and wove it into a natural-language response. You didn't tell it to call the function — the LLM figured that out from the tool description!

---

## Step 5: Add a Second Tool

Add this method above your agent creation:

```csharp
[Description("Get the current time in a given timezone.")]
static string GetTime(
    [Description("The timezone, e.g. 'CET', 'EST', 'PST', 'UTC'.")] string timezone)
{
    var offsets = new Dictionary<string, int>
    {
        ["UTC"] = 0, ["CET"] = 1, ["EST"] = -5, ["PST"] = -8,
        ["JST"] = 9, ["AEST"] = 11
    };
    if (offsets.TryGetValue(timezone.ToUpper(), out var offset))
    {
        var time = DateTimeOffset.UtcNow.ToOffset(TimeSpan.FromHours(offset));
        return $"The current time in {timezone} is {time:HH:mm}.";
    }
    return $"Unknown timezone: {timezone}. Try UTC, CET, EST, PST, JST, or AEST.";
}
```

Update the agent to include both tools:

```csharp
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant that can check weather and time around the world.",
        tools: [
            AIFunctionFactory.Create(GetWeather),
            AIFunctionFactory.Create(GetTime)
        ]);
```

Now test with a question that requires **both** tools:

```csharp
Console.WriteLine(await agent.RunAsync(
    "I'm planning a call with someone in Tokyo. What's the time there and what's the weather like?"));
```

**Observe:** The agent calls **both** tools and combines the results!

---

## 🏋️ Exercises

### Exercise B: Bad Descriptions Experiment

Try giving a tool a vague or misleading description:

```csharp
[Description("Does something.")]
static string GetWeather(string location) => ...
```

Ask the agent about the weather. Does it still call the tool? Now restore the good description. **Lesson:** Tool descriptions are critical for correct function calling.

### Exercise C (Stretch): Agent-as-a-Tool

Create a specialist agent and use it as a tool for a main "router" agent:

```csharp
// ── Specialist weather agent ─────────────────────────────────────────────────
AIAgent weatherAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You answer questions about the weather.",
        name: "WeatherAgent",
        description: "An agent that answers questions about the weather.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// ── Specialist time agent ────────────────────────────────────────────────────
AIAgent timeAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You answer questions about time in different timezones.",
        name: "TimeAgent",
        description: "An agent that answers questions about time around the world.",
        tools: [AIFunctionFactory.Create(GetTime)]);

// ── Main "router" agent that delegates to specialists ────────────────────────
AIAgent mainAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant. Delegate weather questions to the WeatherAgent and time questions to the TimeAgent.",
        tools: [
            weatherAgent.AsAIFunction(),
            timeAgent.AsAIFunction()
        ]);

Console.WriteLine(await mainAgent.RunAsync(
    "What's the weather in Paris and what time is it in Tokyo?"));
```

**Observe:** The main agent delegates to specialist agents, which in turn call their own tools!
