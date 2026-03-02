# Lab 14: Hosting & A2A Protocol — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new web -n Lab14_HostedAgent
cd Lab14_HostedAgent
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
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

## Step 3: Build a Hosted Agent API

Replace `Program.cs`:

```csharp
using System.ComponentModel;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Agents.AI.Hosting;
using Microsoft.Agents.AI.Hosting.A2A;
using Microsoft.Extensions.AI;

var builder = WebApplication.CreateBuilder(args);

// ── Configure Azure OpenAI ───────────────────────────────────────────────────
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

var chatClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName);

// ── Define tools ─────────────────────────────────────────────────────────────
[Description("Get the current weather for a city")]
static string GetWeather([Description("City name")] string city)
    => $"Weather in {city}: partly cloudy, {Random.Shared.Next(15, 30)}°C";

[Description("Get the current time in a timezone")]
static string GetTime([Description("Timezone, e.g. UTC, EST, CET")] string timezone)
    => $"Current time in {timezone}: {DateTime.UtcNow:HH:mm:ss} UTC";

// ── Register the agent via Dependency Injection ──────────────────────────────
var agentBuilder = builder.AddAIAgent(
    name: "TravelAssistant",
    instructions: """
        You are a helpful travel assistant. You can provide weather information
        and current time for destinations around the world. Be concise and
        friendly. When asked about a destination, proactively share both weather
        and time information.
        """,
    chatClient: chatClient.AsIChatClient());

var app = builder.Build();

// ── Health check endpoint ────────────────────────────────────────────────────
app.MapGet("/health", () => Results.Ok(new { status = "healthy", agent = "TravelAssistant" }));

// ── Map the A2A protocol endpoint ────────────────────────────────────────────
app.MapA2A(agentBuilder, "/a2a/travel-assistant");

Console.WriteLine("╔══════════════════════════════════════════════════════════════╗");
Console.WriteLine("║  Travel Assistant API is running!                           ║");
Console.WriteLine("║                                                             ║");
Console.WriteLine("║  Endpoints:                                                 ║");
Console.WriteLine("║    GET  /health              - Health check                 ║");
Console.WriteLine("║    POST /a2a/travel-assistant - A2A protocol endpoint       ║");
Console.WriteLine("╚══════════════════════════════════════════════════════════════╝");

app.Run();
```

## Step 4: Run the Server

```bash
dotnet run --urls "http://localhost:5100"
```

## Step 5: Test the Agent with HTTP Requests

Open a **new terminal** and test:

### Health Check

```bash
curl http://localhost:5100/health
```

Expected output:
```json
{"status":"healthy","agent":"TravelAssistant"}
```

### Send a Message via A2A

```bash
curl -X POST http://localhost:5100/a2a/travel-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "message/send",
    "id": "1",
    "params": {
        "message": {
            "role": "user",
            "parts": [
                {
                    "kind": "text",
                    "text": "What is the weather like in Paris right now?"
                }
            ],
            "messageId": "msg-001"
        }
    }
}'
```

**Observe:** The response follows the A2A JSON-RPC protocol with structured parts.

### Exercise B: Multiple Agents on One Server

```csharp
var restaurantBuilder = builder.AddAIAgent(
    name: "RestaurantAdvisor",
    instructions: "You recommend restaurants based on cuisine and location.",
    chatClient: chatClient.AsIChatClient());

// Later:
app.MapA2A(restaurantBuilder, "/a2a/restaurant-advisor");
```

### Exercise C: Agent-to-Agent Communication

Create a separate console app that acts as an A2A **client**:

```csharp
using System.Net.Http;
using System.Net.Http.Json;

var client = new HttpClient();
var request = new
{
    jsonrpc = "2.0",
    method = "message/send",
    id = "1",
    @params = new
    {
        message = new
        {
            role = "user",
            parts = new[] { new { kind = "text", text = "Weather in London?" } },
            messageId = Guid.NewGuid().ToString()
        }
    }
};

var response = await client.PostAsJsonAsync(
    "http://localhost:5100/a2a/travel-assistant", request);
var body = await response.Content.ReadAsStringAsync();
Console.WriteLine(body);
```
