# Hackathon Starter Guide

This guide provides scaffolding and integration patterns for adding agentic experiences to an existing application using the Microsoft Agent Framework.

---

## Quick Setup

### 1. Add MAF to an Existing .NET Project

```bash
cd YourPartnerApp
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
```

### 2. Create a Shared Agent Factory

Add a service class that creates and configures agents. This keeps agent setup centralized and consistent across the app.

```csharp
// Services/AgentFactory.cs
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

namespace YourApp.Services;

public class AgentFactory
{
    private readonly string _endpoint;
    private readonly string _deploymentName;

    public AgentFactory()
    {
        _endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
            ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
        _deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";
    }

    /// <summary>
    /// Create a basic agent with the given instructions and optional tools.
    /// </summary>
    public AIAgent CreateAgent(string name, string instructions, params AITool[] tools)
    {
        return new AzureOpenAIClient(new Uri(_endpoint), new AzureCliCredential())
            .GetChatClient(_deploymentName)
            .AsAIAgent(instructions: instructions, name: name, tools: tools);
    }
}
```

### 3. Register in DI (ASP.NET Core)

```csharp
// Program.cs — add to service registration
builder.Services.AddSingleton<AgentFactory>();
```

---

## Scenario Scaffolding

### Scenario A: Conversational Assistant

Add a chat agent that helps users navigate the application.

**Integration point:** Add a `/api/chat` endpoint that accepts user messages and returns agent responses.

```csharp
// Controllers/ChatController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.Agents.AI;
using YourApp.Services;

namespace YourApp.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ChatController : ControllerBase
{
    private readonly AgentFactory _factory;

    // Store sessions in memory (use a distributed cache in production)
    private static readonly Dictionary<string, AgentSession> _sessions = new();

    public ChatController(AgentFactory factory) => _factory = factory;

    [HttpPost]
    public async Task<IActionResult> Chat([FromBody] ChatRequest request)
    {
        // ── Create the agent with tools that wrap your existing services ─────
        var agent = _factory.CreateAgent(
            name: "AppAssistant",
            instructions: """
                You are a helpful assistant for [App Name].
                You can help users find information, navigate features, and answer questions.
                Always be concise and helpful.
                If you don't know something, say so — don't make things up.
                """,
            // TODO: Add your app-specific tools here (see "Wiring Tools" below)
            tools: [
                AIFunctionFactory.Create(SearchProducts),
                AIFunctionFactory.Create(GetOrderStatus),
            ]);

        // ── Get or create session for this user ─────────────────────────────
        if (!_sessions.TryGetValue(request.SessionId, out var session))
        {
            session = await agent.CreateSessionAsync();
            _sessions[request.SessionId] = session;
        }

        // ── Run the agent ───────────────────────────────────────────────────
        var response = await agent.RunAsync(request.Message, session);

        return Ok(new { reply = response.ToString() });
    }

    // ── Example tools — replace with your actual app services ────────────────

    [System.ComponentModel.Description("Search for products by name or category.")]
    static string SearchProducts(
        [System.ComponentModel.Description("The search query.")] string query)
    {
        // TODO: Replace with actual service call
        // e.g., return await _productService.SearchAsync(query);
        return $"Found 3 products matching '{query}': Widget A ($10), Widget B ($15), Widget C ($20).";
    }

    [System.ComponentModel.Description("Get the status of an order by order ID.")]
    static string GetOrderStatus(
        [System.ComponentModel.Description("The order ID.")] string orderId)
    {
        // TODO: Replace with actual service call
        return $"Order {orderId}: Shipped on Feb 25, expected delivery Feb 28.";
    }
}

public record ChatRequest(string SessionId, string Message);
```

---

### Scenario B: Process Automation (Workflow)

Automate a multi-step business process using a MAF workflow.

**Example:** Order processing workflow

```csharp
// Workflows/OrderProcessingWorkflow.cs
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Workflows;

namespace YourApp.Workflows;

// ── Step 1: Validate the order ───────────────────────────────────────────────
class ValidateOrderExecutor : Executor
{
    [Handler]
    public async Task Validate(string orderJson, WorkflowContext<string> ctx)
    {
        Console.WriteLine($"  [Validate] Checking order...");
        // TODO: Add real validation logic
        // e.g., check stock, validate address, verify payment
        var validationResult = $"Order validated: {orderJson}";
        await ctx.SendMessageAsync(validationResult);
    }
}

// ── Step 2: Agent enriches the order (e.g., suggests upsells) ────────────────
class EnrichOrderExecutor : Executor
{
    private readonly AIAgent _agent;

    public EnrichOrderExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Enrich(string validatedOrder, WorkflowContext<string> ctx)
    {
        Console.WriteLine($"  [Enrich] Agent analyzing order for recommendations...");
        var result = await _agent.RunAsync(
            $"Analyze this order and suggest one relevant upsell product: {validatedOrder}");
        await ctx.SendMessageAsync(result.ToString()!);
    }
}

// ── Step 3: Generate confirmation email ──────────────────────────────────────
class ConfirmationExecutor : Executor
{
    private readonly AIAgent _agent;

    public ConfirmationExecutor(AIAgent agent) => _agent = agent;

    [Handler]
    public async Task Confirm(string enrichedOrder, WorkflowContext<Never, string> ctx)
    {
        Console.WriteLine($"  [Confirm] Generating confirmation...");
        var result = await _agent.RunAsync(
            $"Write a brief, friendly order confirmation email based on: {enrichedOrder}");
        await ctx.YieldOutputAsync(result.ToString()!);
    }
}

// ── Workflow builder ─────────────────────────────────────────────────────────
public static class OrderWorkflowFactory
{
    public static AgentWorkflow Create(AgentFactory factory)
    {
        var enrichAgent = factory.CreateAgent(
            "EnrichAgent",
            "You analyze orders and suggest relevant upsell products. Be brief.");
        var confirmAgent = factory.CreateAgent(
            "ConfirmAgent",
            "You write friendly, concise order confirmation emails.");

        var validate = new ValidateOrderExecutor();
        var enrich = new EnrichOrderExecutor(enrichAgent);
        var confirm = new ConfirmationExecutor(confirmAgent);

        return new AgentWorkflowBuilder(startExecutor: validate)
            .AddEdge(validate, enrich)
            .AddEdge(enrich, confirm)
            .Build();
    }
}
```

---

### Scenario C: Smart Search & Insights

Add an agent that searches application data and provides intelligent summaries.

```csharp
// Services/InsightsAgent.cs
using System.ComponentModel;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

namespace YourApp.Services;

public class InsightsAgentService
{
    private readonly AIAgent _agent;

    public InsightsAgentService(AgentFactory factory)
    {
        _agent = factory.CreateAgent(
            name: "InsightsAgent",
            instructions: """
                You are a data insights assistant for [App Name].
                When users ask questions about their data, use the available tools to search
                and then provide a clear, concise summary with key takeaways.
                Always cite which data sources you used.
                Format responses with bullet points for readability.
                """,
            tools: [
                AIFunctionFactory.Create(SearchDatabase),
                AIFunctionFactory.Create(GetSalesMetrics),
                AIFunctionFactory.Create(GetCustomerFeedback),
            ]);
    }

    public async Task<string> AskAsync(string question)
    {
        var result = await _agent.RunAsync(question);
        return result.ToString()!;
    }

    // ── Tools that wrap your existing data access layer ──────────────────────

    [Description("Search the application database with a natural language query. Returns matching records.")]
    static string SearchDatabase(
        [Description("The search query.")] string query)
    {
        // TODO: Replace with actual database search
        // e.g., use Entity Framework, Azure AI Search, etc.
        return $"Found 42 records matching '{query}'. Top results: [Record 1], [Record 2], [Record 3].";
    }

    [Description("Get sales metrics for a given time period.")]
    static string GetSalesMetrics(
        [Description("The time period, e.g. 'last month', 'Q4 2025', 'this week'.")] string period)
    {
        // TODO: Replace with actual metrics query
        return $"Sales for {period}: Revenue $125K, Orders 340, Avg order $367, Growth +12%.";
    }

    [Description("Get recent customer feedback and reviews.")]
    static string GetCustomerFeedback(
        [Description("The product or category to get feedback for.")] string topic)
    {
        // TODO: Replace with actual feedback query
        return $"Recent feedback for '{topic}': 4.2/5 avg rating, 89% positive sentiment. Top complaint: shipping delays.";
    }
}
```

---

### Scenario D: Multi-Agent Collaboration

Build specialist agents that work together via Agent-as-a-Tool.

```csharp
// Services/MultiAgentService.cs
using System.ComponentModel;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

namespace YourApp.Services;

public class MultiAgentService
{
    public AIAgent CreateCollaborativeAgent(AgentFactory factory)
    {
        // ── Specialist: Data Analyst ─────────────────────────────────────────
        var analystAgent = factory.CreateAgent(
            name: "DataAnalyst",
            description: "Analyzes data and provides statistical insights.",
            instructions: "You are a data analyst. When given data or a question about data, provide statistical analysis, identify trends, and highlight anomalies. Be precise.",
            tools: [AIFunctionFactory.Create(QueryData)]);

        // ── Specialist: Recommendation Engine ────────────────────────────────
        var recommenderAgent = factory.CreateAgent(
            name: "Recommender",
            description: "Provides personalized recommendations based on user history and preferences.",
            instructions: "You are a recommendation specialist. Based on user data and preferences, suggest relevant products, actions, or next steps. Be specific and actionable.",
            tools: [AIFunctionFactory.Create(GetUserHistory)]);

        // ── Main orchestrator agent ──────────────────────────────────────────
        return factory.CreateAgent(
            name: "Orchestrator",
            instructions: """
                You are the main assistant for [App Name].
                You coordinate with specialist agents to answer complex questions.
                - Use the DataAnalyst for data-related questions
                - Use the Recommender for personalized suggestions
                Always synthesize the specialists' responses into a cohesive answer.
                """,
            tools: [
                analystAgent.AsAIFunction(),
                recommenderAgent.AsAIFunction(),
            ]);
    }

    [Description("Query application data with a SQL-like natural language request.")]
    static string QueryData(
        [Description("The data query.")] string query)
    {
        // TODO: Replace with actual data query
        return $"Query results for '{query}': [sample data rows]";
    }

    [Description("Get a user's purchase history and preferences.")]
    static string GetUserHistory(
        [Description("The user ID.")] string userId)
    {
        // TODO: Replace with actual user data lookup
        return $"User {userId}: 15 purchases, prefers electronics, avg spend $85, last active 2 days ago.";
    }
}
```

---

## Wiring Tools to Your Existing Services

The key to the hackathon is connecting MAF tools to the existing app's **existing services**. Here's the pattern:

```csharp
// ── Pattern: Wrap an existing service as a tool ──────────────────────────────

public class AgentToolBridge
{
    private readonly IProductService _productService;
    private readonly IOrderService _orderService;

    public AgentToolBridge(IProductService productService, IOrderService orderService)
    {
        _productService = productService;
        _orderService = orderService;
    }

    [Description("Search products in the catalog by name, category, or keyword.")]
    public async Task<string> SearchProducts(
        [Description("Search query — can be a product name, category, or keyword.")] string query)
    {
        var results = await _productService.SearchAsync(query);
        return System.Text.Json.JsonSerializer.Serialize(results.Take(5));
    }

    [Description("Get detailed information about a specific product by ID.")]
    public async Task<string> GetProductDetails(
        [Description("The product ID.")] string productId)
    {
        var product = await _productService.GetByIdAsync(productId);
        return product != null
            ? System.Text.Json.JsonSerializer.Serialize(product)
            : $"Product {productId} not found.";
    }

    [Description("Place a new order for a customer.")]
    public async Task<string> PlaceOrder(
        [Description("The customer ID.")] string customerId,
        [Description("The product ID to order.")] string productId,
        [Description("The quantity to order.")] int quantity)
    {
        var order = await _orderService.CreateAsync(customerId, productId, quantity);
        return $"Order {order.Id} created successfully. Total: {order.Total:C}.";
    }
}

// ── Usage: register tools from the bridge ────────────────────────────────────
var bridge = new AgentToolBridge(productService, orderService);
var agent = factory.CreateAgent(
    "AppAssistant",
    "You help users find and order products.",
    tools: [
        AIFunctionFactory.Create(bridge.SearchProducts),
        AIFunctionFactory.Create(bridge.GetProductDetails),
        AIFunctionFactory.Create(bridge.PlaceOrder),
    ]);
```

---

## Adding a Simple Chat UI

If the existing app has a web frontend, here's a minimal chat component:

```html
<!-- chat-widget.html — add to your existing page -->
<div id="agent-chat" style="position:fixed; bottom:20px; right:20px; width:380px; border:1px solid #ccc; border-radius:12px; background:#fff; box-shadow:0 4px 12px rgba(0,0,0,0.15); font-family:system-ui;">
  <div style="background:#0078d4; color:white; padding:12px 16px; border-radius:12px 12px 0 0; font-weight:600;">
    🤖 AI Assistant
  </div>
  <div id="chat-messages" style="height:300px; overflow-y:auto; padding:12px;"></div>
  <div style="display:flex; border-top:1px solid #eee; padding:8px;">
    <input id="chat-input" type="text" placeholder="Ask me anything..."
           style="flex:1; border:1px solid #ddd; border-radius:8px; padding:8px 12px; outline:none;"
           onkeypress="if(event.key==='Enter')sendMessage()">
    <button onclick="sendMessage()"
            style="margin-left:8px; background:#0078d4; color:white; border:none; border-radius:8px; padding:8px 16px; cursor:pointer;">
      Send
    </button>
  </div>
</div>

<script>
const sessionId = crypto.randomUUID();
const messagesDiv = document.getElementById('chat-messages');

function addMessage(text, isUser) {
  const div = document.createElement('div');
  div.style.cssText = `margin:8px 0; padding:8px 12px; border-radius:8px; max-width:80%; ${
    isUser ? 'margin-left:auto; background:#e3f2fd;' : 'background:#f5f5f5;'
  }`;
  div.textContent = text;
  messagesDiv.appendChild(div);
  messagesDiv.scrollTop = messagesDiv.scrollHeight;
}

async function sendMessage() {
  const input = document.getElementById('chat-input');
  const message = input.value.trim();
  if (!message) return;

  addMessage(message, true);
  input.value = '';

  try {
    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionId, message })
    });
    const data = await res.json();
    addMessage(data.reply, false);
  } catch (err) {
    addMessage('Sorry, something went wrong. Please try again.', false);
  }
}
</script>
```

---

## Hackathon Tips

1. **Start simple:** Get a basic agent responding to one question before adding tools
2. **Tools are your bridge:** Every existing service can become a tool with a `[Description]` attribute
3. **Don't over-engineer:** A working demo of 1 scenario beats a broken demo of 3
4. **Use sessions:** Multi-turn conversations make the demo much more impressive
5. **Add middleware last:** Logging/security middleware is polish, not core functionality
6. **Test with real questions:** Think about what a real user would actually ask
