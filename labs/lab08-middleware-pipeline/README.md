# Lab 8: Middleware Pipeline

**Duration:** 20 minutes
**Objective:** Build a middleware pipeline that adds logging, security, and telemetry to an agent without modifying core logic.

---

## What You'll Learn

- Three types of middleware: Agent Run, Function Calling, IChatClient
- How middleware chains form a pipeline (order matters!)
- Practical patterns: logging, security, result modification
- How to keep agent code clean while adding enterprise concerns

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab8_Middleware
cd Lab8_Middleware
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Build a Middleware Pipeline

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

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// ── Tool ─────────────────────────────────────────────────────────────────────
[Description("Get the current weather for a given city.")]
static string GetWeather([Description("The city name.")] string city)
    => $"Weather in {city}: sunny, {Random.Shared.Next(15, 30)}°C.";

// ══════════════════════════════════════════════════════════════════════════════
// MIDDLEWARE 1: Security — blocks sensitive content
// ══════════════════════════════════════════════════════════════════════════════
static async Task<AgentResponse> SecurityMiddleware(
    IEnumerable<ChatMessage> messages,
    AgentSession? session,
    AgentRunOptions? options,
    AIAgent innerAgent,
    CancellationToken ct)
{
    var lastMessage = messages.LastOrDefault()?.Text ?? "";
    var blocked = new[] { "password", "secret", "credit card", "ssn", "api key" };

    if (blocked.Any(kw => lastMessage.Contains(kw, StringComparison.OrdinalIgnoreCase)))
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine("  🛡️ [Security] BLOCKED — sensitive content detected");
        Console.ResetColor();
        return new AgentResponse(
            [new ChatMessage(ChatRole.Assistant, "I can't process requests containing sensitive information.")]);
    }

    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine("  🛡️ [Security] Passed");
    Console.ResetColor();
    return await innerAgent.RunAsync(messages, session, options, ct);
}

// ══════════════════════════════════════════════════════════════════════════════
// MIDDLEWARE 2: Logging — tracks turn count and message volume
// ══════════════════════════════════════════════════════════════════════════════
static int turnCount = 0;

static async Task<AgentResponse> LoggingMiddleware(
    IEnumerable<ChatMessage> messages,
    AgentSession? session,
    AgentRunOptions? options,
    AIAgent innerAgent,
    CancellationToken ct)
{
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
}

// ══════════════════════════════════════════════════════════════════════════════
// MIDDLEWARE 3: Function call logging — tracks tool invocations
// ══════════════════════════════════════════════════════════════════════════════
static async ValueTask<object?> FunctionLoggingMiddleware(
    AIFunction function,
    IReadOnlyList<KeyValuePair<string, object?>> arguments,
    CancellationToken ct,
    Func<IReadOnlyList<KeyValuePair<string, object?>>, CancellationToken, ValueTask<object?>> next)
{
    var args = string.Join(", ", arguments.Select(a => $"{a.Key}={a.Value}"));
    Console.ForegroundColor = ConsoleColor.Cyan;
    Console.WriteLine($"  🔧 [FuncLog] → {function.Name}({args})");
    Console.ResetColor();

    var sw = Stopwatch.StartNew();
    var result = await next(arguments, ct);
    sw.Stop();

    Console.ForegroundColor = ConsoleColor.Cyan;
    Console.WriteLine($"  🔧 [FuncLog] ← {function.Name} returned in {sw.ElapsedMilliseconds}ms");
    Console.ResetColor();

    return result;
}

// ── Create agent with middleware pipeline ─────────────────────────────────────
AIAgent baseAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful weather assistant.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// Middleware order matters! Security first, then logging.
AIAgent agent = baseAgent
    .AsBuilder()
        .Use(runFunc: SecurityMiddleware)
        .Use(runFunc: LoggingMiddleware)
        .Use(FunctionLoggingMiddleware)
    .Build();

// ── Test the pipeline ────────────────────────────────────────────────────────
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

## Step 3: Run and Observe

```bash
dotnet run
```

**Observe the pipeline in action:**
- Test 1: Security ✅ → Logging → Function call logged → Response
- Test 3: Security ❌ → BLOCKED before reaching the agent

---

## 🏋️ Exercises

### Exercise A: Add Rate Limiting Middleware

Create middleware that limits the number of calls per session:

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

Create function middleware that appends a disclaimer to every tool result:

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

### Exercise C (Stretch): Middleware Ordering Experiment

1. Put `LoggingMiddleware` BEFORE `SecurityMiddleware` — what changes?
2. Remove security middleware — does the blocked query now succeed?
3. **Key insight:** Middleware order = execution order. Security should always be first.

---

## ✅ Success Criteria

- [ ] Security middleware blocks sensitive queries before they reach the agent
- [ ] Logging middleware shows turn counts and timing
- [ ] Function logging middleware shows which tools are called with what arguments
- [ ] You understand that middleware order determines execution order

---

## 📚 Reference

- [Middleware docs](https://learn.microsoft.com/en-us/agent-framework/agents/middleware/)
