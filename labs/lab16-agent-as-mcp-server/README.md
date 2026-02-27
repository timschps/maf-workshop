# Lab 16: Agent as MCP Server

## Objective

Expose a MAF agent **as an MCP server** so that any MCP-compatible client (VS Code GitHub Copilot, other agents, Claude Desktop, etc.) can discover and use your agent as a tool.

## What You'll Learn

- How to convert an agent into an `AIFunction` using `.AsAIFunction()`
- How to wrap the function as an `McpServerTool`
- How to host the MCP server over stdio transport
- How other MCP clients can discover and call your agent

## Prerequisites

- Completed Lab 1 (Hello Agent)
- Azure OpenAI endpoint configured

## Duration

15 minutes

## Steps

### Step 1: Create a new console application

```bash
dotnet new console -n Lab16_AgentAsMcpServer
cd Lab16_AgentAsMcpServer
```

### Step 2: Add NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.Hosting
dotnet add package ModelContextProtocol --prerelease
```

### Step 3: Replace Program.cs

Replace the contents of `Program.cs` with:

```csharp
using System;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;
using OpenAI.Chat;

#pragma warning disable MEAI001

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

// --- Step A: Create the agent ---
AIAgent jokeAgent = new AzureOpenAIClient(
        new Uri(endpoint),
        new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a comedian. Tell short, clever jokes. Keep responses under 100 words.",
        name: "JokeAgent");

// --- Step B: Convert agent to MCP tool ---
McpServerTool tool = McpServerTool.Create(jokeAgent.AsAIFunction());

Console.Error.WriteLine("🎭 JokeAgent MCP Server starting...");
Console.Error.WriteLine($"   Tool name: {tool.ProtocolTool.Name}");
Console.Error.WriteLine("   Waiting for MCP client connections via stdio...");

// --- Step C: Host as MCP server over stdio ---
HostApplicationBuilder builder = Host.CreateEmptyApplicationBuilder(settings: null);
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithTools([tool]);

await builder.Build().RunAsync();
```

### Step 4: Build the application

```bash
dotnet build
```

### Step 5: Test with an MCP client

The server communicates via stdin/stdout. You can test it by connecting from another application.

**Option A — Test with the MCP Inspector:**

```bash
npx -y @modelcontextprotocol/inspector dotnet run --project .
```

This opens a web UI where you can see the exposed tool and invoke it.

**Option B — Configure in VS Code (GitHub Copilot Agent Mode):**

Add to your VS Code `settings.json`:

```json
{
  "github.copilot.chat.mcpServers": {
    "joke-agent": {
      "command": "dotnet",
      "args": ["run", "--project", "path/to/Lab16_AgentAsMcpServer"]
    }
  }
}
```

Then in Copilot Chat (Agent Mode), the JokeAgent will appear as an available tool!

## How It Works

```
┌─────────────┐     stdio      ┌──────────────────┐     LLM API     ┌─────────────┐
│  MCP Client  │ ──────────────▶│  MCP Server       │ ──────────────▶│ Azure OpenAI │
│  (VS Code,   │ ◀──────────────│  (Your Agent)     │ ◀──────────────│              │
│   Inspector) │                └──────────────────┘                 └─────────────┘
└─────────────┘
```

1. An MCP client connects to your server via stdio
2. The client discovers the `JokeAgent` tool via the MCP protocol
3. When invoked, the tool runs `agent.RunAsync()` under the hood
4. The agent calls Azure OpenAI and returns the response
5. The response flows back to the MCP client

## Key Concepts

| Concept | Description |
|---------|-------------|
| `agent.AsAIFunction()` | Converts an agent into a callable `AIFunction` |
| `McpServerTool.Create()` | Wraps an `AIFunction` as an MCP-compatible tool |
| `WithStdioServerTransport()` | Serves MCP over stdin/stdout (local process) |
| `Console.Error.WriteLine` | Log to stderr (stdout is reserved for MCP protocol) |

## 🎯 Challenge

Create a multi-tool MCP server that exposes **two** agents — a JokeAgent and a FactAgent — as separate tools in the same MCP server!

## What's Next?

In **Lab 17**, you'll use the A2A (Agent-to-Agent) protocol to call remote agents over HTTP — the network equivalent of what MCP does locally.
