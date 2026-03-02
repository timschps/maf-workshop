# Lab 16: Agent as MCP Server — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab16_agent_as_mcp_server
cd lab16_agent_as_mcp_server
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework --pre azure-identity mcp
```

## Step 3: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_CHAT_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 4: Write the Code

Create a file named `main.py`:

```python
import asyncio
import sys
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types

# ── Create the agent ──────────────────────────────────────────────────────────
client = AzureOpenAIChatClient(credential=AzureCliCredential())
joke_agent = client.as_agent(
    name="JokeAgent",
    instructions="You are a comedian. Tell short, clever jokes. Keep responses under 100 words.",
)

# ── Create the MCP server ────────────────────────────────────────────────────
server = Server("joke-agent-mcp-server")


@server.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="ask_joke_agent",
            description="Ask the JokeAgent to tell a joke or respond humorously",
            inputSchema={
                "type": "object",
                "properties": {
                    "question": {
                        "type": "string",
                        "description": "The question or topic for the joke agent",
                    }
                },
                "required": ["question"],
            },
        )
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "ask_joke_agent":
        result = await joke_agent.run(arguments["question"])
        return [types.TextContent(type="text", text=result.text)]
    raise ValueError(f"Unknown tool: {name}")


async def main():
    print("🎭 JokeAgent MCP Server starting...", file=sys.stderr)
    print("   Tool name: ask_joke_agent", file=sys.stderr)
    print("   Waiting for MCP client connections via stdio...", file=sys.stderr)

    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())


asyncio.run(main())
```

> **Note:** The Python SDK doesn't have a direct "Agent as MCP Server" pattern like C#.
> Instead, we use the `mcp` Python package to build an MCP server that wraps agent calls.
> stdout is reserved for the MCP protocol — use `sys.stderr` for logging.

## Step 5: Build and Test

**Option A — Test with the MCP Inspector:**

```bash
npx -y @modelcontextprotocol/inspector python main.py
```

This opens a web UI where you can see the exposed tool and invoke it.

**Option B — Configure in VS Code (GitHub Copilot Agent Mode):**

Add to your VS Code `settings.json`:

```json
{
  "github.copilot.chat.mcpServers": {
    "joke-agent": {
      "command": "python",
      "args": ["path/to/lab16_agent_as_mcp_server/main.py"]
    }
  }
}
```

Then in Copilot Chat (Agent Mode), the JokeAgent will appear as an available tool!
