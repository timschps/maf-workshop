# Lab 8: Middleware Pipeline — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab8_Middleware
cd Lab8_Middleware
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

## Step 3: Build a Middleware Pipeline

Replace `Program.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Diagnostics;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI.Chat;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

[Description("Get the current weather for a given city.")]
static string GetWeather([Description("The city name.")] string city)
    => $"Weather in {city}: sunny, {Random.Shared.Next(15, 30)}°C.";

int turnCount = 0;

AIAgent baseAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful weather assistant.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

#pragma warning disable MEAI001
AIAgent agent = baseAgent
    .AsBuilder()
    // Agent run middleware: security check + logging
    .Use(
        // runFunc — intercepts RunAsync calls
        async (IEnumerable<ChatMessage> messages, AgentSession? session,
               AgentRunOptions? options, AIAgent innerAgent, CancellationToken ct) =>
        {
            // ── Security check ───────────────────────────────────────────────
            var lastMessage = messages.LastOrDefault()?.Text ?? "";
            var blocked = new[] { "password", "secret", "credit card", "ssn", "api key" };

            if (blocked.Any(kw => lastMessage.Contains(kw, StringComparison.OrdinalIgnoreCase)))
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("  🛡️ [Security] BLOCKED — sensitive content detected");
                Console.ResetColor();
                return new AgentResponse(
                    [new ChatMessage(ChatRole.Assistant,
                        "I can't process requests containing sensitive information.")]);
            }

            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("  🛡️ [Security] Passed");
            Console.ResetColor();

            // ── Logging ──────────────────────────────────────────────────────
            turnCount++;
            var sw = Stopwatch.StartNew();
            Console.ForegroundColor = ConsoleColor.DarkGray;
            Console.WriteLine($"  📊 [Logging] Turn #{turnCount} | Input messages: {messages.Count()}");
            Console.ResetColor();

            var response = await innerAgent.RunAsync(messages, session, options, ct);
            sw.Stop();

            Console.ForegroundColor = ConsoleColor.DarkGray;
            Console.WriteLine($"  📊 [Logging] Turn #{turnCount} | Response msgs: {response.Messages.Count} | Duration: {sw.ElapsedMilliseconds}ms");
            Console.ResetColor();
            return response;
        },
        // runStreamingFunc — null means streaming uses the default behavior
        null
    )
    // Function invocation middleware: log tool calls
    .Use(async (AIAgent a, FunctionInvocationContext ctx,
                Func<FunctionInvocationContext, CancellationToken, ValueTask<object?>> next,
                CancellationToken ct) =>
    {
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"  🔧 [FuncLog] → {ctx.Function.Name}(...)");
        Console.ResetColor();

        var sw = Stopwatch.StartNew();
        var result = await next(ctx, ct);
        sw.Stop();

        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"  🔧 [FuncLog] ← {ctx.Function.Name} returned in {sw.ElapsedMilliseconds}ms");
        Console.ResetColor();
        return result;
    })
    .Build();
#pragma warning restore MEAI001

AgentSession session = await agent.CreateSessionAsync();

Console.WriteLine("═══ Test 1: Normal question (triggers tool) ═══\n");
Console.WriteLine(await agent.RunAsync("What's the weather in Amsterdam?", session));
Console.WriteLine();

Console.WriteLine("═══ Test 2: Follow-up question (uses session) ═══\n");
Console.WriteLine(await agent.RunAsync("How about in Tokyo?", session));
Console.WriteLine();

Console.WriteLine("═══ Test 3: Blocked by security middleware ═══\n");
Console.WriteLine(await agent.RunAsync("What's the api key for the weather service?", session));
Console.WriteLine();

Console.WriteLine("═══ Test 4: No tool needed ═══\n");
Console.WriteLine(await agent.RunAsync("What's the capital of France?", session));
```

## Step 4: Run and Observe

```bash
dotnet run
```

**Observe the pipeline in action:**
- Test 1: Security ✅ → Logging → Function call logged → Response
- Test 3: Security ❌ → BLOCKED before reaching the agent

### Exercise A: Add Rate Limiting Middleware

```csharp
static int callsThisSession = 0;
const int MAX_CALLS = 5;

static async Task<AgentResponse> RateLimitMiddleware(
    IEnumerable<ChatMessage> messages, AgentSession? session,
    AgentRunOptions? options, AIAgent innerAgent, CancellationToken ct)
{
    callsThisSession++;
    if (callsThisSession > MAX_CALLS)
    {
        return new AgentResponse(
            [new ChatMessage(ChatRole.Assistant, $"Rate limit exceeded. Max {MAX_CALLS} calls per session.")]);
    }
    return await innerAgent.RunAsync(messages, session, options, ct);
}
```

### Exercise B: Result Modifier Middleware

```csharp
static async ValueTask<object?> DisclaimerMiddleware(
    AIFunction function, IReadOnlyList<KeyValuePair<string, object?>> args,
    CancellationToken ct,
    Func<IReadOnlyList<KeyValuePair<string, object?>>, CancellationToken, ValueTask<object?>> next)
{
    var result = await next(args, ct);
    return $"{result} (Data is simulated for demonstration.)";
}
```
