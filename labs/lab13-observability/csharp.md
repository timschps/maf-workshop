# Lab 13: Observability & Telemetry — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab13_Observability
cd Lab13_Observability
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Exporter.Console
dotnet add package Microsoft.Extensions.AI.OpenTelemetry
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

## Step 3: Add Telemetry to an Agent

Replace `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;
using OpenTelemetry;
using OpenTelemetry.Trace;
using OpenTelemetry.Resources;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

const string SourceName = "MAFWorkshop";

// ── Configure OpenTelemetry to export to console ─────────────────────────────
using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("maf-lab13"))
    .AddSource(SourceName)
    .AddSource("*Microsoft.Extensions.AI")
    .AddConsoleExporter()
    .Build();

// ── Tool ─────────────────────────────────────────────────────────────────────
[Description("Get the current weather for a city.")]
static string GetWeather([Description("City name.")] string city)
    => $"Weather in {city}: sunny, {Random.Shared.Next(15, 30)}°C.";

// ── Create an instrumented agent via the builder pipeline ────────────────────
var agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient()
    .AsBuilder()
    .UseOpenTelemetry(sourceName: SourceName, configure: cfg => cfg.EnableSensitiveData = true)
    .Build()
    .AsAIAgent(
        name: "TelemetryAgent",
        instructions: "You are a helpful weather assistant.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// ── Run the agent and observe telemetry ──────────────────────────────────────
Console.WriteLine("═══ Running agent with OpenTelemetry ═══\n");

AgentSession session = await agent.CreateSessionAsync();

Console.WriteLine("Q: What's the weather in Amsterdam?");
var response = await agent.RunAsync("What's the weather in Amsterdam?", session);
Console.WriteLine($"A: {response}\n");

Console.WriteLine("Q: How about Tokyo?");
response = await agent.RunAsync("How about Tokyo?", session);
Console.WriteLine($"A: {response}\n");

Console.WriteLine("═══ Check console output above for trace spans! ═══\n");
Console.WriteLine("Look for:");
Console.WriteLine("  - invoke_agent spans with agent name and instructions");
Console.WriteLine("  - gen_ai.usage.input_tokens and output_tokens");
Console.WriteLine("  - execute_tool spans with function name and duration");
```

## Step 4: Run It

```bash
dotnet run
```

**Observe in the console output:**
- `invoke_agent TelemetryAgent` spans with timing
- `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` attributes
- `execute_tool` spans showing which functions were called
- Trace IDs for correlating related operations

### Exercise B: Add Metrics Export

```csharp
using OpenTelemetry.Metrics;

using var meterProvider = Sdk.CreateMeterProviderBuilder()
    .AddMeter("*Microsoft.Agents.AI")
    .AddConsoleExporter()
    .Build();
```

### Exercise C: Export to Aspire Dashboard

```bash
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
.AddOtlpExporter(opt => opt.Endpoint = new Uri("http://localhost:4317"))
```
