# Lab 15: MCP Tools Integration

## Objective

Connect your agent to an external **Model Context Protocol (MCP)** server and use its tools. MCP is an open standard that lets agents discover and call tools exposed by any MCP-compatible server — giving your agent access to external data sources, APIs, and services without writing custom tool code.

## What You'll Learn

- How to connect to an MCP server using the official C# SDK
- How to retrieve tools from an MCP server and pass them to an agent
- How agents use MCP tools via function calling (just like local tools)
- The difference between stdio and HTTP-based MCP transports

## Prerequisites

- Completed Lab 1 (Hello Agent)
- Node.js installed (for running the MCP test server)
- Azure OpenAI endpoint configured

## Duration

20 minutes

## Steps

### Step 1: Create a new console application

```bash
dotnet new console -n Lab15_McpTools
cd Lab15_McpTools
```

### Step 2: Add NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package ModelContextProtocol --prerelease
```

### Step 3: Replace Program.cs

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

### Step 4: Run the application

```bash
dotnet run
```

You should see the agent discover tools from the MCP server and use them to answer your questions.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **MCP Server** | A service that exposes tools, resources, and prompts via the Model Context Protocol |
| **MCP Client** | Connects to an MCP server to discover and invoke its tools |
| **StdioTransport** | Runs an MCP server as a local subprocess, communicating via stdin/stdout |
| **Tool Discovery** | `ListToolsAsync()` retrieves all available tools from the server |
| **Tool → AITool** | MCP tools are cast to `AITool` and passed to the agent like any other tool |

## Connecting to Remote HTTP MCP Servers

For remote MCP servers (like `https://staybright-demo-app.azurewebsites.net/mcp`), use `HttpClientTransport` from `ModelContextProtocol.Client`:

```csharp
await using var mcpClient = await McpClient.CreateAsync(
    new HttpClientTransport(new()
    {
        Name = "RemoteMcpServer",
        Endpoint = new Uri("https://staybright-demo-app.azurewebsites.net/mcp"),
    }));
```

> **Note:** Remote MCP servers may require authentication headers. Pass an `HttpClient` with the appropriate auth headers configured.

## Popular MCP Servers to Try

| Server | Install | Description |
|--------|---------|-------------|
| `@modelcontextprotocol/server-everything` | `npx -y @modelcontextprotocol/server-everything` | Test server with echo, add, and sample tools |
| `@modelcontextprotocol/server-github` | `npx -y @modelcontextprotocol/server-github` | GitHub API tools (requires `GITHUB_PERSONAL_ACCESS_TOKEN`) |
| `@modelcontextprotocol/server-filesystem` | `npx -y @modelcontextprotocol/server-filesystem /path` | File system access tools |
| Azure MCP Server | [github.com/Azure/azure-mcp](https://github.com/Azure/azure-mcp) | Azure services (Storage, Cosmos DB, etc.) |

## 🎯 Challenge

Try connecting to the GitHub MCP server and asking the agent to summarize recent commits from a repository you like!

## What's Next?

In **Lab 16**, you'll flip the script — exposing your own agent *as* an MCP server that other tools (like VS Code Copilot) can discover and use.
