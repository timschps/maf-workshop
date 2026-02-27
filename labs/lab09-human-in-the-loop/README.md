# Lab 9: Human-in-the-Loop Tool Approval

**Duration:** 20 minutes
**Objective:** Build an agent where sensitive tool calls require human approval before execution.

---

## What You'll Learn

- How to mark function tools as requiring human approval
- How to detect approval requests in agent responses
- How to build an approval loop (approve/reject/show details)
- When and why human-in-the-loop matters for production agents

---

## Step 1: Create the Project

```bash
dotnet new console -n Lab9_ToolApproval
cd Lab9_ToolApproval
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
```

## Step 2: Create Tools with Approval Requirements

Replace `Program.cs`:

```csharp
using System;
using System.ComponentModel;
using System.Linq;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

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
{
    // In production, this would actually send an email
    return $"✅ Email sent to {to} with subject '{subject}'.";
}

[Description("Place an order for a product. This charges the customer's account.")]
static string PlaceOrder(
    [Description("Product name.")] string product,
    [Description("Quantity to order.")] int quantity)
{
    return $"✅ Order placed: {quantity}x {product}.";
}

// ── Wrap sensitive tools with approval requirement ───────────────────────────
AIFunction weatherTool = AIFunctionFactory.Create(GetWeather);
AIFunction emailTool = new ApprovalRequiredAIFunction(AIFunctionFactory.Create(SendEmail));
AIFunction orderTool = new ApprovalRequiredAIFunction(AIFunctionFactory.Create(PlaceOrder));

// ── Create the agent ─────────────────────────────────────────────────────────
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: """
            You are a helpful assistant that can check weather, send emails, and place orders.
            Always confirm the details before taking action.
            """,
        tools: [weatherTool, emailTool, orderTool]);

// ── Approval loop ────────────────────────────────────────────────────────────
AgentSession session = await agent.CreateSessionAsync();

async Task<string> AskAgent(string question)
{
    Console.WriteLine($"\n❓ User: {question}");
    AgentResponse response = await agent.RunAsync(question, session);

    // Check if any tool calls need approval
    var approvalRequests = response.Messages
        .SelectMany(m => m.Contents)
        .OfType<FunctionApprovalRequestContent>()
        .ToList();

    if (approvalRequests.Count == 0)
    {
        // No approval needed — return the response directly
        return response.ToString()!;
    }

    // Process each approval request
    foreach (var request in approvalRequests)
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"\n  ⚠️  Approval required for: {request.FunctionCall.Name}");
        Console.WriteLine($"      Arguments: {request.FunctionCall.Arguments}");
        Console.ResetColor();

        Console.Write("  Approve? (y/n): ");
        var input = Console.ReadLine()?.Trim().ToLower();
        bool approved = input == "y" || input == "yes";

        Console.WriteLine(approved ? "  ✅ Approved!" : "  ❌ Rejected.");

        // Send the approval/rejection back to the agent
        var approvalMessage = new ChatMessage(
            ChatRole.User, [request.CreateResponse(approved)]);
        response = await agent.RunAsync(approvalMessage, session);
    }

    return response.ToString()!;
}

// ── Test scenarios ───────────────────────────────────────────────────────────
Console.WriteLine("═══ Scenario 1: Safe tool (no approval needed) ═══");
Console.WriteLine($"💬 {await AskAgent("What's the weather in Amsterdam?")}");

Console.WriteLine("\n═══ Scenario 2: Sensitive tool (needs approval) ═══");
Console.WriteLine($"💬 {await AskAgent("Send an email to alice@example.com with subject 'Meeting tomorrow' and body 'Hi Alice, can we meet at 2pm?'")}");

Console.WriteLine("\n═══ Scenario 3: Another sensitive tool ═══");
Console.WriteLine($"💬 {await AskAgent("Order 3 units of Widget Pro.")}");
```

## Step 3: Run It

```bash
dotnet run
```

**Observe:**
- Weather queries execute immediately (no approval needed)
- Email and order tools pause and ask for human confirmation
- Rejecting a tool call results in the agent adapting its response

---

## 🏋️ Exercises

### Exercise A: Selective Approval

Make some tools always require approval, and others only in certain conditions. For example: orders under $50 auto-approve, orders over $50 require approval.

### Exercise B: Approval with Details

Enhance the approval prompt to show more details and formatting:

```csharp
Console.WriteLine($"  📧 To: {/* extract 'to' from arguments */}");
Console.WriteLine($"  📝 Subject: {/* extract 'subject' */}");
Console.Write("  Send this email? (y/n): ");
```

### Exercise C (Stretch): Audit Trail

Log all approval decisions to a list and print a summary at the end:

```csharp
var auditLog = new List<(string Tool, string Args, bool Approved, DateTime Time)>();
// ... add to auditLog in the approval loop ...
Console.WriteLine("\n═══ Audit Trail ═══");
foreach (var entry in auditLog)
    Console.WriteLine($"  {entry.Time:HH:mm:ss} | {entry.Tool} | {(entry.Approved ? "✅" : "❌")} | {entry.Args}");
```

---

## ✅ Success Criteria

- [ ] Safe tools (weather) execute without approval
- [ ] Sensitive tools (email, order) pause for human approval
- [ ] Approving a tool call results in execution
- [ ] Rejecting a tool call results in the agent adapting gracefully
- [ ] You understand why human-in-the-loop matters for production agents

---

## 📚 Reference

- [Tool Approval docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/tool-approval)
