# Lab 16: Agent as MCP Server — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab16_AgentAsMcpServer
cd Lab16_AgentAsMcpServer
```

## Step 2: Add NuGet Packages

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Microsoft.Extensions.Hosting
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

## Step 5: Build and Test

```bash
dotnet build
```

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
