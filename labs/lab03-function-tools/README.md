# Lab 2: Agent with Tools — Function Calling

**Duration:** 25 minutes
**Objective:** Give your agent the ability to call custom functions, and observe how the LLM autonomously decides when to use them.

---

## What You'll Learn

- How to define function tools with descriptions
- How the LLM uses tool descriptions to decide when and how to call tools
- How to register multiple tools with an agent
- How to compose agents (Agent-as-a-Tool pattern)

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab2_AgentTools
cd Lab2_AgentTools
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Define Your First Function Tool

A function tool is just a C# method with `[Description]` attributes. The descriptions tell the LLM **what** the tool does and **when** to call it.

Create `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
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

## Step 3: Run It

```bash
dotnet run
```

**Observe:** The agent automatically decided to call `GetWeather("Amsterdam")`, received the result, and wove it into a natural-language response. You didn't tell it to call the function — the LLM figured that out from the tool description!

---

## Step 4: Add a Second Tool

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

### Exercise A: Build Your Own Tool

Create a tool relevant to a business scenario. Ideas:

| Tool | Description |
|------|-------------|
| `LookupCustomer` | Takes a customer ID, returns name + email |
| `GetProductPrice` | Takes a product name, returns the price |
| `SearchKnowledgeBase` | Takes a query, returns relevant FAQ entries |
| `CalculateShipping` | Takes weight + destination, returns cost |

Wire it to the agent and verify the LLM calls it at the right time.

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

---

## ✅ Success Criteria

- [ ] Agent calls `GetWeather` automatically when asked about weather
- [ ] Agent handles questions that DON'T require tools gracefully
- [ ] You've added a second tool and seen the agent use both
- [ ] You understand why tool descriptions matter

---

## 🐍 Python Alternative

<details>
<summary>Click to expand Python version</summary>

```python
import asyncio
import os
from typing import Annotated
from random import randint
from azure.identity import AzureCliCredential
from agent_framework import tool
from agent_framework.azure import AzureOpenAIResponsesClient
from pydantic import Field

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The city name.")],
) -> str:
    """Get the current weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "partly cloudy"]
    return f"Weather in {location}: {conditions[randint(0,3)]}, {randint(5,30)}°C."

@tool(approval_mode="never_require")
def get_time(
    timezone: Annotated[str, Field(description="The timezone, e.g. 'CET', 'EST'.")],
) -> str:
    """Get the current time in a given timezone."""
    from datetime import datetime, timezone as tz, timedelta
    offsets = {"UTC": 0, "CET": 1, "EST": -5, "PST": -8, "JST": 9}
    if timezone.upper() in offsets:
        now = datetime.now(tz(timedelta(hours=offsets[timezone.upper()])))
        return f"Time in {timezone}: {now.strftime('%H:%M')}"
    return f"Unknown timezone: {timezone}"

async def main():
    credential = AzureCliCredential()
    client = AzureOpenAIResponsesClient(
        project_endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        deployment_name=os.environ["AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME"],
        credential=credential,
    )

    agent = client.as_agent(
        name="ToolsAgent",
        instructions="You are a helpful assistant that can check weather and time.",
        tools=[get_weather, get_time],
    )

    result = await agent.run("What's the weather in Amsterdam and time in Tokyo?")
    print(f"Agent: {result}")

asyncio.run(main())
```

</details>

---

## 📚 Reference

- [Official Step 2 docs](https://learn.microsoft.com/en-us/agent-framework/get-started/add-tools)
- [Tools overview](https://learn.microsoft.com/en-us/agent-framework/agents/tools/)
- [Agent-as-a-Tool](https://learn.microsoft.com/en-us/agent-framework/agents/tools/#using-an-agent-as-a-function-tool)
