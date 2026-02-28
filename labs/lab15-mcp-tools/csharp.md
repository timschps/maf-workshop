# Lab 15: MCP Tools Integration — C# Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab15_McpTools
cd Lab15_McpTools
```

## Step 2: Add NuGet Packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package ModelContextProtocol --prerelease
```

## Step 3: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 4: Replace Program.cs

Replace the contents of `Program.cs` with:

```csharp
using System;
using System.Linq;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using ModelContextProtocol.Client;
using OpenAI.Chat;

#pragma warning disable MEAI001

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// --- Step A: Connect to an MCP server ---
// The "everything" test server exposes sample tools (echo, add, etc.)
Console.WriteLine("🔌 Connecting to MCP server...");

await using var mcpClient = await McpClient.CreateAsync(
    new StdioClientTransport(new()
    {
        Name = "TestMcpServer",
        Command = "npx",
        Arguments = ["-y", "@modelcontextprotocol/server-everything"],
    }));

// --- Step B: Retrieve tools from the MCP server ---
var mcpTools = await mcpClient.ListToolsAsync();
Console.WriteLine($"📦 Found {mcpTools.Count} MCP tools:");
foreach (var tool in mcpTools)
{
    Console.WriteLine($"   🔧 {tool.Name}: {tool.Description}");
}

// --- Step C: Create an agent with MCP tools ---
AIAgent agent = new AzureOpenAIClient(
        new Uri(endpoint),
        new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant. Use the available tools to answer questions. When asked to echo something, use the echo tool. When asked to add numbers, use the add tool.",
        tools: [.. mcpTools.Cast<AITool>()]);

// --- Step D: Use the agent with MCP tools ---
Console.WriteLine("\n💬 Asking agent to use MCP tools...\n");

var response1 = await agent.RunAsync("Please echo back the message: 'Hello from MCP!'");
Console.WriteLine($"Echo result: {response1}\n");

var response2 = await agent.RunAsync("What is 42 + 58?");
Console.WriteLine($"Add result: {response2}\n");

Console.WriteLine("✅ MCP tools integration complete!");
```

## Step 5: Run It

```bash
dotnet run
```

You should see the agent discover tools from the MCP server and use them to answer your questions.

## Connecting to Remote HTTP MCP Servers (C#)

For remote MCP servers, use `HttpClientTransport`:

```csharp
await using var mcpClient = await McpClient.CreateAsync(
    new HttpClientTransport(new()
    {
        Name = "RemoteMcpServer",
        Endpoint = new Uri("https://staybright-demo-app.azurewebsites.net/mcp"),
    }));
```

> **Note:** Remote MCP servers may require authentication headers. Pass an `HttpClient` with the appropriate auth headers configured.
