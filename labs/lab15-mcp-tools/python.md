# Lab 15: MCP Tools Integration — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab15_mcp_tools
cd lab15_mcp_tools
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework --pre azure-identity
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
from agent_framework import MCPStdioTool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

async def main():
    # ── Create the chat client ────────────────────────────────────────────────
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ── Step A: Connect to an MCP server via stdio ────────────────────────────
    # The "everything" test server exposes sample tools (echo, add, etc.)
    print("🔌 Connecting to MCP server...")

    mcp_tool = MCPStdioTool(
        name="everything",
        command="npx",
        args=["-y", "@modelcontextprotocol/server-everything"],
        description="Test MCP server with echo, add, and sample tools",
    )

    async with mcp_tool:
        # ── Step B: Create an agent with MCP tools ────────────────────────────
        agent = client.as_agent(
            name="McpToolAgent",
            instructions=(
                "You are a helpful assistant. Use the available tools to answer questions. "
                "When asked to echo something, use the echo tool. "
                "When asked to add numbers, use the add tool."
            ),
            tools=[mcp_tool],
        )

        # ── Step C: Use the agent with MCP tools ─────────────────────────────
        print("\n💬 Asking agent to use MCP tools...\n")

        result1 = await agent.run("Please echo back the message: 'Hello from MCP!'")
        print(f"Echo result: {result1.text}\n")

        result2 = await agent.run("What is 42 + 58?")
        print(f"Add result: {result2.text}\n")

        print("✅ MCP tools integration complete!")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

You should see the agent discover tools from the MCP server and use them to answer your questions.

## Connecting to Remote HTTP MCP Servers (Python)

For remote MCP servers, use `MCPStreamableHTTPTool`:

```python
from agent_framework import MCPStreamableHTTPTool

mcp_tool = MCPStreamableHTTPTool(
    name="demo",
    url="https://staybright-demo-app.azurewebsites.net/mcp",
    description="Demo MCP server",
    load_prompts=False,
    approval_mode='never_require',
)

async with mcp_tool:
    agent = client.as_agent(
        name="RemoteMcpAgent",
        instructions="You are a helpful assistant. Use available tools.",
        tools=[mcp_tool],
    )
    result = await agent.run("What tools are available?")
    print(result.text)
```

> **Note:** Remote MCP servers may require authentication. Configure headers as needed.
