# Lab 9: Human-in-the-Loop Tool Approval — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab9_ToolApproval
cd Lab9_ToolApproval
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

## Step 3: Create Tools with Approval Middleware

Replace `Program.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
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

// ── Safe tool: no approval needed ────────────────────────────────────────────
[Description("Get the current weather for a given city. This is a read-only operation.")]
static string GetWeather([Description("The city name.")] string city)
    => $"Weather in {city}: sunny, {Random.Shared.Next(15, 30)}°C.";

// ── Sensitive tool: requires approval ────────────────────────────────────────
[Description("Send an email to a recipient with a subject and body. This sends a real email.")]
static string SendEmail(
    [Description("Recipient email address.")] string to,
    [Description("Email subject line.")] string subject,
    [Description("Email body text.")] string body)
    => $"✅ Email sent to {to} with subject '{subject}'.";

[Description("Place an order for a product. This charges the customer's account.")]
static string PlaceOrder(
    [Description("Product name.")] string product,
    [Description("Quantity to order.")] int quantity)
    => $"✅ Order placed: {quantity}x {product}.";

// Sensitive functions that require approval
var sensitiveFunctions = new HashSet<string> { "SendEmail", "PlaceOrder" };

AIAgent baseAgent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: """
            You are a helpful assistant that can check weather, send emails, and place orders.
            Always confirm the details before taking action.
            """,
        tools: [
            AIFunctionFactory.Create(GetWeather),
            AIFunctionFactory.Create(SendEmail),
            AIFunctionFactory.Create(PlaceOrder),
        ]);

// ── Add approval middleware via function invocation ───────────────────────────
#pragma warning disable MEAI001
AIAgent agent = baseAgent
    .AsBuilder()
    .Use(async (AIAgent a, FunctionInvocationContext ctx,
                Func<FunctionInvocationContext, CancellationToken, ValueTask<object?>> next,
                CancellationToken ct) =>
    {
        // Check if this function requires approval
        if (sensitiveFunctions.Any(s => ctx.Function.Name.Contains(s, StringComparison.OrdinalIgnoreCase)))
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"\n  ⚠️  Approval required for: {ctx.Function.Name}");
            if (ctx.Arguments != null)
                foreach (var arg in ctx.Arguments)
                    Console.WriteLine($"      {arg.Key} = {arg.Value}");
            Console.ResetColor();

            Console.Write("  Approve? (y/n): ");
            var input = Console.ReadLine()?.Trim().ToLower();
            bool approved = input == "y" || input == "yes";

            if (approved)
            {
                Console.WriteLine("  ✅ Approved!");
                return await next(ctx, ct);
            }
            else
            {
                Console.WriteLine("  ❌ Rejected.");
                return $"Action '{ctx.Function.Name}' was rejected by the user.";
            }
        }

        // Safe functions pass through without approval
        return await next(ctx, ct);
    })
    .Build();
#pragma warning restore MEAI001

// ── Test scenarios ───────────────────────────────────────────────────────────
Console.WriteLine("═══ Scenario 1: Safe tool (no approval needed) ═══");
Console.WriteLine($"💬 {await agent.RunAsync("What's the weather in Amsterdam?")}");

Console.WriteLine("\n═══ Scenario 2: Sensitive tool (needs approval) ═══");
Console.WriteLine($"💬 {await agent.RunAsync("Send an email to alice@example.com with subject 'Meeting tomorrow' and body 'Hi Alice, can we meet at 2pm?'")}");

Console.WriteLine("\n═══ Scenario 3: Another sensitive tool ═══");
Console.WriteLine($"💬 {await agent.RunAsync("Order 3 units of Widget Pro.")}");
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:**
- Weather queries execute immediately (no approval needed)
- Email and order tools are intercepted by middleware and prompt for approval
- Rejecting a tool call returns a rejection message to the agent

### Exercise B: Approval with Details

```csharp
Console.WriteLine($"  📧 To: {/* extract 'to' from arguments */}");
Console.WriteLine($"  📝 Subject: {/* extract 'subject' */}");
Console.Write("  Send this email? (y/n): ");
```

### Exercise C: Audit Trail

```csharp
var auditLog = new List<(string Tool, string Args, bool Approved, DateTime Time)>();
// ... add to auditLog in the approval loop ...
Console.WriteLine("\n═══ Audit Trail ═══");
foreach (var entry in auditLog)
    Console.WriteLine($"  {entry.Time:HH:mm:ss} | {entry.Tool} | {(entry.Approved ? "✅" : "❌")} | {entry.Args}");
```
